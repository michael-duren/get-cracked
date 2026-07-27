---
course: build-a-container-runtime
title: Build an OCI Container Runtime in C
language: c
description: >
  Containers are not virtual machines — they're ordinary Linux processes
  wearing a disguise the kernel provides. Build minict, a real OCI runtime
  in C, one piece per lesson: parse config.json with your own scanner, turn
  namespace lists into clone flags, jail paths so a hostile config can't
  escape the rootfs, map container root to an unprivileged user, translate
  resource limits into cgroup v2 writes, hand a PTY master down a console
  socket, and drive the create/start/kill/delete lifecycle. Every lesson
  ends with a command you run on your own machine, and the last one drops
  you into a shell inside a container your own runtime built — rootless,
  no Docker, no root.
duration_hours: 12
tags: [systems, linux, containers, c]
extended_reading:
  - title: OCI Runtime Specification — config.json
    url: https://github.com/opencontainers/runtime-spec/blob/main/config.md
  - title: "namespaces(7) — the Linux man page"
    url: https://man7.org/linux/man-pages/man7/namespaces.7.html
  - title: "pivot_root(2) — including the '.' '.' trick"
    url: https://man7.org/linux/man-pages/man2/pivot_root.2.html
  - title: Control Group v2 — the kernel documentation
    url: https://docs.kernel.org/admin-guide/cgroup-v2.html
  - title: "runc docs — terminals, stdio, and the console socket"
    url: https://github.com/opencontainers/runc/blob/main/docs/terminals.md
  - title: "user_namespaces(7) — mapping files, and the rules around them"
    url: https://man7.org/linux/man-pages/man7/user_namespaces.7.html
  - title: "Rootless containers — the delegation rules cgroups follow"
    url: https://rootlesscontaine.rs/getting-started/common/cgroup2/
  - title: "Liz Rice — Containers From Scratch (talk)"
    url: https://www.youtube.com/watch?v=8fi7uSYlOdc
---

# Lesson: What Is a Container, Really? {#what-is-a-container}

Strip away the branding and a container is an ordinary Linux process that
the kernel has agreed to lie to. It has a pid, it shows up in `ps` on the
host, you can `kill` it. Three kernel features make it feel like a machine
of its own:

- **Namespaces** change what the process can *see*: its own pid numbering,
  its own hostname, its own network interfaces, its own mount table.
- **Cgroups** change what it can *use*: caps on memory, CPU time, and how
  many processes it may spawn.
- **A pivoted root filesystem** changes what it can *reach*: `/` points
  into an unpacked image, and the host's tree is unreachable.

There is no "container object" in the kernel — no syscall creates one.
A container is a *recipe*: a dozen syscalls issued in the right order, by
a program called a **container runtime**. That's what you're building.

## Where a runtime sits

When you type `docker run` (or kubelet schedules a pod), your command
travels down a well-defined stack. The highlighted layer is the one this
course builds — the part that touches the kernel:

```d2
direction: down
you: "docker run · kubectl — the commands you actually type" { shape: oval }
hl: "high-level runtime: containerd / CRI-O — pulls images, unpacks them into a bundle"
oci: "OCI runtime: runc, crun — and the one you're building. Reads config.json, makes syscalls" {
  style.stroke: "#d97706"
  style.stroke-width: 3
}
k: "Linux kernel — namespaces · cgroups · pivot_root"
you -> hl
hl -> oci
oci -> k
```

The boundary between the top layers and the bottom one is a written
contract: the **OCI Runtime Specification** (OCI = Open Container
Initiative). It says a runtime is handed a **bundle** — one directory —
and must be able to create, start, kill, and delete a container from it.
Because runc, crun, youki, and gVisor's runsc all honor the same
contract, Docker and Kubernetes can swap one for another without
noticing. Your runtime will honor it too.

## The bundle

A bundle is refreshingly boring: a config file and an unpacked root
filesystem.

```
bundle/
├── config.json     ← everything about HOW to run it
└── rootfs/         ← everything the container can see
    ├── bin/
    ├── etc/
    ├── usr/
    └── ...
```

`config.json` is the entire input to your runtime: which namespaces to
create, what hostname to set, what command to exec, what uid the process
runs as, what memory it may use. Images, registries, layers, tags — none
of that exists at this layer; the high-level runtime already flattened all
of it into `rootfs/`.

## The project: minict

You are building one program across this whole course. It is called
**minict**, it is about 700 lines of C, and when you are done this
works on your own laptop, with no Docker daemon and no `sudo`:

```
$ minict create demo ./bundle --console-socket /tmp/console.sock
minict: created demo — host pid 40219, pid 1 inside
$ minict start demo
/ # hostname
duck
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    4 root      0:00 ps
/ # exit
$ minict delete demo
```

That shell is pid 1 of its own pid namespace, its `/` is an unpacked
image, it believes it is root, and it cannot spawn more than the
processes its cgroup allows. It is a container, and nothing built it
but your code.

Each lesson adds one piece, and each source file is mostly the
functions those lessons' challenges grade:

```
minict/
  Makefile
  bundle/            the container you'll run: config.json + rootfs/
  src/
    minict.h         shared declarations
    oci.c            L2 the JSON scanner · L1 nsflags · L4 secure_join
    nspid.c          L3 the pid-ladder parser
    idmap.c          L5 id translation, both directions
    cgroup.c         L6 the three formatters, and the writes they feed
    state.c          L7 the lifecycle FSM, and state.json
    console.c        L8 the console plan, the PTY, the fd handoff
    container.c      the syscall choreography — given to you in full
    main.c           CLI dispatch — given to you in full
    attach.c         the caller's side of the console socket — given
```

Two files (`container.c`, `main.c`) are the privileged choreography;
they are printed in full in the epilogue and you paste them in. Every
other file is yours. That division is not arbitrary — it is roughly how
runc itself divides, and it is why the challenges here grade *decision
logic* rather than syscalls: parsing, validation, mapping, and
formatting are the overwhelming majority of a real runtime's code, and
the only part that can be tested without a kernel to talk to. The
grader runs in a sandbox with no privileges to hand you, so the
challenges below are graded; **the milestones you run are not
submitted** — they're yours, and they're the point.

## What you need

A Linux machine (or VM — WSL2 works; macOS does not, since the syscalls
*are* the subject). Then:

- **gcc or clang**, and `make`.
- **Unprivileged user namespaces enabled.** Check with
  `unshare -U -r id` — it should print `uid=0(root)`. If it errors,
  see the epilogue's troubleshooting note.
- **A rootfs to run.** Any unpacked image works. The one-liner used
  throughout this course, if you have Docker available:

  ```sh
  mkdir -p bundle/rootfs
  cid=$(docker create busybox)
  docker export "$cid" | tar -x -C bundle/rootfs
  docker rm "$cid"
  ```

  No Docker? Download a busybox static binary into `bundle/rootfs/bin/`
  and symlink `sh` to it — a rootfs is just a directory tree, and the
  runtime never asks where it came from.

Everything after this runs as your normal user. If a command in this
course needs `sudo`, it is a bug in the course.

## First piece: names to flags

The OCI config lists namespaces by name — `"pid"`,
`"mount"`, `"network"`. The kernel doesn't take names; it takes a bitmask
of `CLONE_*` flags passed to the `clone` or `unshare` syscall. Each flag
is one bit (`CLONE_NEWPID` is `0x20000000`, `CLONE_NEWNS` is
`0x00020000`, …), so a set of namespaces is just those bits OR-ed
together — and the very first thing a runtime does with a config is fold
the name list into that mask.

Two things deserve rejection with an error, not tolerance: a namespace
type you don't handle — whether a typo, a config from the future, or the
newer `time` namespace this course's seven-flag subset leaves out —
because refusing beats silently running less isolated than asked; and
the same type listed twice (the spec forbids it, and a duplicate usually
means a generator bug).

## Milestone 0: the bitmask is the whole interface

Before writing `nsflags`, prove to yourself that a bitmask is really all
the kernel wants. `unshare(1)` is a thin wrapper over the syscall your
runtime will call:

```sh
$ unshare -U -r --uts sh -c 'hostname duck; hostname; id -u'
duck
0
$ hostname
your-laptop
```

You just changed the hostname — as "root" — and your real machine did
not notice. Two flags (`CLONE_NEWUSER | CLONE_NEWUTS`) bought that.
`nsflags`, your first challenge, is the function that turns
`["uts","user"]` from a config file into those two bits. Solve it, and
save it as `src/oci.c` in your project directory; the next lesson adds
the scanner that reads the names out of `config.json` in the first
place.

## Challenge: Names to Clone Flags {#ns-clone-flags points=10}

Implement `nsflags`: fold an array of OCI namespace type names into one
`CLONE_*` bitmask. Watch the names — the OCI spec says `"mount"` and
`"network"`, even though `/proc/self/ns/` calls the same namespaces `mnt`
and `net`. A runtime speaks spec on the way in and kernel on the way out.

### Starter

```c
#include <stddef.h>
#include <string.h>

/* Values from <linux/sched.h> — defined here so no kernel headers are
   needed. One bit per namespace. */
#define CLONE_NEWNS     0x00020000u  /* OCI name: "mount" */
#define CLONE_NEWCGROUP 0x02000000u  /* OCI name: "cgroup" */
#define CLONE_NEWUTS    0x04000000u  /* OCI name: "uts" */
#define CLONE_NEWIPC    0x08000000u  /* OCI name: "ipc" */
#define CLONE_NEWUSER   0x10000000u  /* OCI name: "user" */
#define CLONE_NEWPID    0x20000000u  /* OCI name: "pid" */
#define CLONE_NEWNET    0x40000000u  /* OCI name: "network" */

/* OR together the flag for each OCI namespace type in names[0..n).
   0 on success (writing the mask to *out); -1 on an unknown name or a
   duplicate. An empty list is valid: zero flags. */
int nsflags(const char *const names[], size_t n, unsigned long *out) {
	/* TODO: map each name to its flag; reject unknowns and repeats */
	(void)names;
	(void)n;
	(void)out;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>

#define CLONE_NEWNS     0x00020000u
#define CLONE_NEWCGROUP 0x02000000u
#define CLONE_NEWUTS    0x04000000u
#define CLONE_NEWIPC    0x08000000u
#define CLONE_NEWUSER   0x10000000u
#define CLONE_NEWPID    0x20000000u
#define CLONE_NEWNET    0x40000000u

int nsflags(const char *const names[], size_t n, unsigned long *out);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

int main(void) {
	unsigned long flags;

	const char *one[] = {"pid"};
	check(nsflags(one, 1, &flags) == 0 && flags == CLONE_NEWPID,
	      "test_single_namespace");

	const char *two[] = {"pid", "mount"};
	check(nsflags(two, 2, &flags) == 0 &&
	      flags == (CLONE_NEWPID | CLONE_NEWNS),
	      "test_flags_or_together");

	const char *all[] = {"pid", "network", "mount", "ipc",
	                     "uts", "user", "cgroup"};
	check(nsflags(all, 7, &flags) == 0 &&
	      flags == (CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS |
	                CLONE_NEWIPC | CLONE_NEWUTS | CLONE_NEWUSER |
	                CLONE_NEWCGROUP),
	      "test_all_seven");

	flags = 0xdead;
	check(nsflags(NULL, 0, &flags) == 0 && flags == 0,
	      "test_empty_list_is_zero_flags");

	/* OCI says "network" and "mount" — not the /proc/self/ns names. */
	const char *procname[] = {"net"};
	check(nsflags(procname, 1, &flags) == -1, "test_net_is_not_an_oci_name");
	const char *mnt[] = {"mnt"};
	check(nsflags(mnt, 1, &flags) == -1, "test_mnt_is_not_an_oci_name");

	const char *unknown[] = {"pid", "banana"};
	check(nsflags(unknown, 2, &flags) == -1, "test_unknown_name_rejected");

	const char *dup[] = {"pid", "uts", "pid"};
	check(nsflags(dup, 3, &flags) == -1, "test_duplicate_rejected");

	return failed;
}
```

# Lesson: Reading the Spec — config.json {#config-json}

Everything your runtime does is dictated by `config.json`. A trimmed but
real one looks like this:

```json
{
  "ociVersion": "1.2.0",
  "hostname": "duck",
  "process": {
    "args": ["/bin/sh"],
    "cwd": "/"
  },
  "root": { "path": "rootfs" },
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "mount" },
      { "type": "uts" }
    ],
    "resources": {
      "memory": { "limit": 268435456 }
    }
  }
}
```

runc parses this with Go's `encoding/json`. In C the traditional move is
to vendor a JSON library — but this course is dependency-free, and it
turns out a runtime touches such a small, predictable slice of JSON that
a **hand-rolled scanner** covers it: find a key, check the value's shape,
copy it out. Writing one also teaches you exactly what a parser promises
and what it doesn't — which is the difference between trusting your
tools and being surprised by them.

## Scanning for a key

The plan for `json_str(doc, "hostname", out, cap)` — find the *key*, then
copy its string *value*. Red is the failure exit:

```d2
direction: down
k: "scan for \"hostname\" used as a key — quoted, and followed by ':'"
v: "skip the ':' and whitespace — the value must open with a quote"
cp: "copy bytes until the closing quote, unescaping \\\" and \\\\"
ok: "return 0 — out holds \"duck\"" { shape: oval }
err: "return -1" {
  shape: oval
  style.stroke: "#dc2626"
  style.stroke-width: 2
}
k -> v -> cp -> ok
k -> err: "no such key"
v -> err: "not a string"
cp -> err: "buffer full"
```

The subtle step is the first one. The byte sequence `"hostname"` can
appear in a document as a key — or as somebody's *value*:

```json
{ "note": "hostname", "hostname": "duck" }
```

The scanner must not bite on the first occurrence. The tell: after a
closing quote, a **key** is followed by `:` (possibly with whitespace);
a value is followed by `,` or `}`. So the loop is: find a `"`, check the
key text and closing quote follow, skip whitespace, and only accept if
the next byte is `:` — otherwise keep scanning.

Three C details will make or break it:

- **`strchr` is your scan loop.** `strchr(p, '"')` finds each candidate
  quote; `strncmp(p + 1, key, klen)` checks the text. No index
  arithmetic, no manual loop over every byte.
- **Escapes exist even in minimal JSON.** A path value like
  `"a \"quoted\" dir"` stores `\"` for each inner quote. Your copy loop
  handles exactly two escapes — `\"` and `\\` — by copying the escaped
  byte and skipping the backslash. (Full JSON has `\n`, `\uXXXX`, etc.;
  the configs a runtime meets in practice don't, and rejecting oddities
  loudly beats mishandling them quietly.)
- **The buffer contract.** `out` holds at most `cap` bytes *including*
  the NUL terminator. A value that doesn't fit is an error — truncating
  a path and using it would be far worse than refusing.

One honest limitation to carry forward: this scanner finds the *first*
key with a given name anywhere in the document — it has no notion of
nesting. For the config subset we handle, key names don't collide across
sections, and where they do (two sections each have a `"limit"`), the
final challenge scans *from the section's position*, which you'll meet in
the last lesson. A production runtime uses a real parser; know which tool
you're holding.

## Milestone 1: your bundle's config

Write the `config.json` your runtime will spend the rest of the course
reading. Save it as `bundle/config.json`, next to the `rootfs/` you
unpacked in lesson one:

```json
{
  "ociVersion": "1.2.0",
  "hostname": "duck",
  "root": { "path": "rootfs" },
  "process": {
    "terminal": false,
    "args": ["/bin/sh"],
    "cwd": "/"
  },
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "mount" },
      { "type": "uts" },
      { "type": "user" }
    ],
    "resources": {
      "memory": { "limit": 268435456 },
      "cpu": { "quota": 50000, "period": 100000 },
      "pids": { "limit": 64 }
    }
  }
}
```

Four namespaces, not seven: `network` and `ipc` are left out so the
container shares your network (nothing to configure, and DNS works),
and `cgroup` is left out because rootless cgroup setup gets its own
lesson. Add them later and watch what breaks — that's a good hour.

Then prove your scanner reads it. Once you've solved the challenge
below, save it as `src/oci.c` alongside `nsflags` and point a throwaway
`main` at the real file:

```c
/* scratch.c — cc -o t src/oci.c scratch.c && ./t bundle/config.json */
#include <stdio.h>
int json_str(const char *doc, const char *key, char *out, size_t cap);

int main(int argc, char **argv) {
	FILE *f = fopen(argv[1], "rb");
	char doc[4096];
	doc[fread(doc, 1, sizeof doc - 1, f)] = '\0';

	char host[64], path[64];
	json_str(doc, "hostname", host, sizeof host);
	json_str(doc, "path", path, sizeof path);
	printf("hostname=%s rootfs=%s\n", host, path);
	return 0;
}
```

```
hostname=duck rootfs=rootfs
```

Two strings out of a real document, using no library. That is the whole
input side of a container runtime — everything else this course does is
a consequence of those bytes.

## Challenge: A Key Scanner {#json-string-scan points=15}

Implement `json_str`. It must skip key-lookalike values, tolerate
whitespace around the `:`, unescape `\"` and `\\`, and enforce the buffer
cap. Empty string values are legal.

### Starter

```c
#include <stddef.h>
#include <string.h>

/* Find `"key"` used as a JSON key (its closing quote is followed by
   optional whitespace and a ':'), then copy its string value into out.

   0 on success. -1 if: the key never appears as a key, the value is not
   a string, the document ends mid-string, or the value (plus its NUL)
   does not fit in cap bytes. Handles the escapes \" and \\ in values. */
int json_str(const char *doc, const char *key, char *out, size_t cap) {
	/* TODO: strchr to each '"', strncmp the key, demand a ':',
	   then copy the value byte by byte */
	(void)doc;
	(void)key;
	(void)out;
	(void)cap;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>
#include <string.h>

int json_str(const char *doc, const char *key, char *out, size_t cap);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

int main(void) {
	char out[64];

	check(json_str("{\"hostname\":\"duck\"}", "hostname", out, sizeof out) == 0 &&
	      strcmp(out, "duck") == 0,
	      "test_simple_value");

	check(json_str("{ \"hostname\" :   \"duck\" }", "hostname", out, sizeof out) == 0 &&
	      strcmp(out, "duck") == 0,
	      "test_whitespace_around_colon");

	const char *nested =
		"{\n"
		"  \"ociVersion\": \"1.2.0\",\n"
		"  \"root\": { \"path\": \"rootfs\" }\n"
		"}";
	check(json_str(nested, "path", out, sizeof out) == 0 &&
	      strcmp(out, "rootfs") == 0,
	      "test_nested_key");
	check(json_str(nested, "ociVersion", out, sizeof out) == 0 &&
	      strcmp(out, "1.2.0") == 0,
	      "test_first_key");

	check(json_str("{\"a\": \"x\"}", "hostname", out, sizeof out) == -1,
	      "test_missing_key");

	/* "hostname" appears first as a VALUE — the scanner must not bite. */
	check(json_str("{\"note\": \"hostname\", \"hostname\": \"duck\"}",
	               "hostname", out, sizeof out) == 0 &&
	      strcmp(out, "duck") == 0,
	      "test_key_text_as_value_is_skipped");

	check(json_str("{\"path\": \"a \\\"quoted\\\" dir\"}", "path",
	               out, sizeof out) == 0 &&
	      strcmp(out, "a \"quoted\" dir") == 0,
	      "test_escaped_quote_in_value");

	check(json_str("{\"path\": \"a\\\\b\"}", "path", out, sizeof out) == 0 &&
	      strcmp(out, "a\\b") == 0,
	      "test_escaped_backslash");

	check(json_str("{\"pids\": 64}", "pids", out, sizeof out) == -1,
	      "test_non_string_value");

	check(json_str("{\"path\": \"", "path", out, sizeof out) == -1,
	      "test_truncated_document");

	check(json_str("{\"hostname\": \"\"}", "hostname", out, sizeof out) == 0 &&
	      out[0] == '\0',
	      "test_empty_value");

	check(json_str("{\"hostname\": \"duck\"}", "hostname", out, 4) == -1,
	      "test_value_too_long_for_buffer");
	check(json_str("{\"hostname\": \"duck\"}", "hostname", out, 5) == 0 &&
	      strcmp(out, "duck") == 0,
	      "test_value_exactly_fits");

	return failed;
}
```

# Lesson: Namespaces — Lying to a Process {#namespaces}

A namespace wraps one global kernel resource — the pid table, the
hostname, the network stack, the mount table — and gives a group of
processes a private copy of it. Linux has eight kinds; the seven your
runtime handles map one-to-one onto the `CLONE_*` flags from lesson one
(the eighth, `time`, is newer and rarer). Two syscalls create them:

- `unshare(flags)` — *this* process leaves the shared namespaces and gets
  fresh ones.
- `clone(flags, ...)` — like fork, but the *child* is born into fresh
  namespaces.

Real runtimes use `clone`: the parent stays on the host to supervise,
and the child becomes the container. The child that `clone` creates is
the **container init** — pid 1 of a brand-new pid namespace, even though
the host sees it under an ordinary number:

```d2
grid-columns: 2
horizontal-gap: 100
host: "host pid namespace" {
  s: "systemd — pid 1"
  b: "your shell — pid 3020"
  r: "runtime parent — pid 3021"
  s -> b -> r
}
ctr: "new pid namespace" {
  i: "container init\nhost sees 3022 · it sees itself as 1" {
    style.stroke: "#d97706"
    style.stroke-width: 3
  }
  w: "worker\nhost sees 3040 · it sees itself as 7"
  i -> w
}
host.r -> ctr.i: "clone(CLONE_NEWPID)"
```

Same processes, two truths: every container process has one pid *per
namespace level*, and which one you see depends on where you're standing.

## The pid namespace's sharp edges

The pid namespace is the one that punishes sloppy runtimes, because it
breaks three habits at once:

- **`unshare(CLONE_NEWPID)` does not move the caller.** A process's pid
  is fixed at birth, so `unshare` only marks the *next* child to be born
  into the new namespace. Forget this and your "container" is still in
  the host's pid namespace, mystified.
- **Pid 1 is special.** Inside the namespace, the init process must reap
  orphaned children (or zombies accumulate), and if it dies the kernel
  kills every process in the namespace. Real images ship a tiny init
  (`tini`, `dumb-init`) for exactly this reason.
- **`ps` lies until you remount /proc.** `ps` reads `/proc`, and a stale
  `/proc` mount still shows the *host's* processes. A runtime mounts a
  fresh proc inside the container's mount namespace so pid 1 sees itself.

## Reading both truths from /proc

The kernel will happily show you a process's full pid stack. Every
`/proc/<pid>/status` file has an `NSpid` field — one column per
namespace level, outermost first:

```
$ grep NSpid /proc/4020/status
                ┌────── pid on the host
                │     ┌────── pid one namespace in
                │     │    ┌────── pid in the innermost namespace
NSpid:       4020    17    7
```

One column means "not in a nested pid namespace"; three columns means
containers-in-containers (nesting is real: Docker-in-Docker, Kubernetes
pods). Runtimes and debuggers parse this line whenever they need to
translate "pid 7 inside the container" into "pid 4020, the one I can
actually signal from out here."

That parsing is your challenge: `NSpid:` then one or more decimal pids,
separated by tabs or spaces. As always at a trust boundary, garbage —
a wrong field name, a non-numeric token, more levels than your buffer
holds — is an error, not a shrug.

## Milestone 2: two truths about one process

See the double pid yourself, with no code at all. In one terminal, put a
shell in a fresh pid namespace and have it report what it believes:

```sh
$ unshare -U -r --pid --fork --mount-proc sh -c 'echo "inside I am pid $$"; exec sleep 300'
inside I am pid 1
```

(The `exec` matters: it replaces the shell with `sleep`, so the process
still sitting there afterwards is the same one that just told you it is
pid 1.) In another terminal, ask the host about that process:

```
$ pgrep -x sleep
41120
$ grep NSpid /proc/41120/status
NSpid:	41120	1
```

One process, two numbers, both true. `41120` is the pid you can signal
from out here; `1` is what it calls itself in there. `--mount-proc` is
the flag that makes the inside view honest — drop it and `ps` inside
still lists your whole host, which is the third sharp edge from the
list above, live.

Your challenge is the parser for that `NSpid:` line. Save it as
`src/nspid.c`; `minict create` will use it at the end of this course to
print `host pid 41120, pid 1 inside` — the single most useful line of
output a runtime can give you when something is wrong.

## Challenge: Parse the Pid Ladder {#nspid-parse points=10}

Implement `nspid_parse`: given one line from `/proc/<pid>/status`,
extract the pids into `out` (outermost first) and return how many there
were, or -1 on anything malformed.

### Starter

```c
#include <stddef.h>
#include <string.h>

/* Parse an "NSpid:" line from /proc/<pid>/status: the literal field name
   "NSpid:" then 1+ decimal pids separated by spaces/tabs, optionally
   ending in a newline.

   Fill out[] (outermost pid first) and return the count. -1 if the line
   is not an NSpid line, a token is not a number, there are no pids at
   all, or there are more than cap pids. */
int nspid_parse(const char *line, int *out, size_t cap) {
	/* TODO: check the prefix, then loop: skip separators, parse digits */
	(void)line;
	(void)out;
	(void)cap;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>

int nspid_parse(const char *line, int *out, size_t cap);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

int main(void) {
	int pids[8];

	check(nspid_parse("NSpid:\t3021\t17\t1\n", pids, 8) == 3 &&
	      pids[0] == 3021 && pids[1] == 17 && pids[2] == 1,
	      "test_three_levels");

	check(nspid_parse("NSpid:\t42\n", pids, 8) == 1 && pids[0] == 42,
	      "test_host_only");

	check(nspid_parse("NSpid: 3021 17 1", pids, 8) == 3 &&
	      pids[0] == 3021 && pids[2] == 1,
	      "test_spaces_and_no_newline");

	check(nspid_parse("Pid:\t3021\n", pids, 8) == -1,
	      "test_wrong_field_rejected");

	check(nspid_parse("NSpid:\t3021\tx\t1\n", pids, 8) == -1,
	      "test_garbage_token_rejected");

	check(nspid_parse("NSpid:\n", pids, 8) == -1,
	      "test_no_pids_rejected");

	check(nspid_parse("NSpid:\t3021\t17\t1\n", pids, 2) == -1,
	      "test_too_many_for_buffer");

	return failed;
}
```

# Lesson: The Root of the Matter — pivot_root {#pivot-root}

The container has its own pid table and hostname, but it can still see
your entire disk. Fixing that is the most delicate sequence in the whole
runtime — and the one with the worst failure mode, because getting it
*almost* right leaves a door back to the host.

## Why not chroot?

`chroot` moves a process's *idea* of `/` but not its reality: the old
root stays mounted, and a root process inside a chroot can climb back
out (the classic: open a directory fd outside the jail, `fchdir` to it,
then `chroot(".")` — the kernel happily walks `..` past the old
barrier). `pivot_root` is the grown-up version: it swaps what `/`
*is* in this mount namespace, and lets you unmount the old root
entirely. Nothing you've unmounted can be escaped into.

## The dance

Six calls, in an order where every line exists to satisfy a kernel rule.
Amber is the line being executed; the tree on the right shows what the
mount namespace looks like after it runs:

```d2
grid-columns: 2
horizontal-gap: 60
code: "" {
  grid-columns: 1
  l1: " mount(\"/\", MS_REC | MS_PRIVATE)" {
    height: 30
    style.font: mono
    style.stroke: "#9ca3af"
    label.near: center-left
  }
  l2: " mount(rootfs, rootfs, MS_BIND)" {
    height: 30
    style.font: mono
    style.stroke: "#9ca3af"
    label.near: center-left
  }
  l3: " chdir(rootfs)" {
    height: 30
    style.font: mono
    style.stroke: "#9ca3af"
    label.near: center-left
  }
  l4: " pivot_root(\".\", \".\")" {
    height: 30
    style.font: mono
    style.stroke: "#9ca3af"
    label.near: center-left
  }
  l5: " umount2(\".\", MNT_DETACH)" {
    height: 30
    style.font: mono
    style.stroke: "#9ca3af"
    label.near: center-left
  }
  l6: " chdir(\"/\")" {
    height: 30
    style.font: mono
    style.stroke: "#9ca3af"
    label.near: center-left
  }
}
tree: "the mount tree" {
  hostroot: "/ — the host root"
  rootfs: "/ctr/rootfs — a directory"
  hostroot -> rootfs
  rootfs -> hostroot: "old root stacks here" {
    style.stroke-dash: 4
    style.stroke: "#9ca3af"
  }
}
steps: {
  "make every mount private — mount events stop leaking to the host": {
    code.l1.style.stroke: "#d97706"
    code.l1.style.stroke-width: 2
    code.l1.style.bold: true
    tree.hostroot.style.stroke: "#d97706"
    tree.hostroot.style.stroke-width: 3
  }
  "bind rootfs over itself — the directory becomes a mount point": {
    code.l1.style.stroke: "#9ca3af"
    code.l1.style.stroke-width: 1
    code.l1.style.bold: false
    code.l2.style.stroke: "#d97706"
    code.l2.style.stroke-width: 2
    code.l2.style.bold: true
    tree.hostroot.style.stroke-width: 1
    tree.hostroot.style.stroke: "#9ca3af"
    tree.rootfs.label: "/ctr/rootfs — a mount point"
    tree.rootfs.style.stroke: "#d97706"
    tree.rootfs.style.stroke-width: 3
  }
  "step inside the new root before pivoting": {
    code.l2.style.stroke: "#9ca3af"
    code.l2.style.stroke-width: 1
    code.l2.style.bold: false
    code.l3.style.stroke: "#d97706"
    code.l3.style.stroke-width: 2
    code.l3.style.bold: true
  }
  "pivot_root — rootfs becomes /, the old root stacks on top of it": {
    code.l3.style.stroke: "#9ca3af"
    code.l3.style.stroke-width: 1
    code.l3.style.bold: false
    code.l4.style.stroke: "#d97706"
    code.l4.style.stroke-width: 2
    code.l4.style.bold: true
    tree.rootfs.label: "/ — the container root"
    tree.hostroot.label: "old root — still an escape!"
    tree.hostroot.style.stroke: "#dc2626"
    tree.hostroot.style.stroke-width: 3
    tree.(rootfs -> hostroot)[0].style.stroke: "#dc2626"
    tree.(rootfs -> hostroot)[0].style.stroke-dash: 0
    tree.(rootfs -> hostroot)[0].style.stroke-width: 2
  }
  "detach the old root — the host's tree vanishes from this namespace": {
    code.l4.style.stroke: "#9ca3af"
    code.l4.style.stroke-width: 1
    code.l4.style.bold: false
    code.l5.style.stroke: "#d97706"
    code.l5.style.stroke-width: 2
    code.l5.style.bold: true
    tree.hostroot.label: "old root — gone"
    tree.hostroot.style.stroke: "#9ca3af"
    tree.hostroot.style.stroke-width: 1
    tree.hostroot.style.stroke-dash: 4
    tree.hostroot.style.font-color: "#9ca3af"
    tree.(rootfs -> hostroot)[0].style.stroke: "#9ca3af"
    tree.(rootfs -> hostroot)[0].style.stroke-dash: 4
    tree.(rootfs -> hostroot)[0].style.stroke-width: 1
    tree.(hostroot -> rootfs)[0].style.stroke: "#9ca3af"
    tree.(hostroot -> rootfs)[0].style.stroke-dash: 4
  }
  "chdir(\"/\") — the container can never see outside rootfs again": {
    code.l5.style.stroke: "#9ca3af"
    code.l5.style.stroke-width: 1
    code.l5.style.bold: false
    code.l6.style.stroke: "#d97706"
    code.l6.style.stroke-width: 2
    code.l6.style.bold: true
  }
}
```

The kernel rules those steps satisfy:

- **Step 1** (`MS_REC | MS_PRIVATE`): on systemd hosts every mount is
  *shared* — mount and unmount events propagate between namespaces. If
  you skip this, your unmount in step 5 propagates back and detaches the
  host's real root from *its* namespace (and `pivot_root` refuses shared
  roots with `EINVAL` precisely to stop you). One recursive remount makes
  the whole tree private: events stop at the wall.
- **Step 2**: `pivot_root`'s new root must be a *mount point*, not a mere
  directory. Bind-mounting `rootfs` onto itself is the standard trick to
  make it one without moving any data.
- **Steps 3–5**: `pivot_root(".", ".")` is the documented idiom — new
  root and old root are the *same directory*, so the old root ends up
  mounted *on top of* the new one, and `umount2(".", MNT_DETACH)` peels
  it away. No temp directory inside the image, nothing to clean up. Until
  that unmount happens, the red edge in the diagram is a working escape
  hatch back to the host — which is why the unmount is not optional.

## The other door: paths in the config

`pivot_root` seals the walls, but your runtime still *builds* paths from
`config.json` before the seal: the rootfs location itself, plus every
mount destination the config asks for (`/proc`, `/dev`, volumes). Those
strings come from an untrusted file, and a malicious one will happily
say:

```json
{ "destination": "../../../../etc/cron.d" }
```

Resolve that naively against your rootfs and the runtime — running as
root, remember — writes to the *host's* `/etc/cron.d`. This exact class
of bug has hit real runtimes more than once (runc patched a variant of
it as CVE-2021-30465). The defense is a joiner that resolves `.` and
`..` *lexically, inside the jail*: `..` pops one component but can never
pop past the rootfs; absolute paths are re-rooted; the result is
guaranteed to start with the rootfs prefix. runc ships this as a
dedicated library (`filepath-securejoin`); yours is next. (Symlinks
inside the rootfs add another layer of attack that lexical resolution
can't see — real runtimes pair this with `openat2`'s `RESOLVE_BENEATH`.
Build the lexical core first; it's the part every defense shares.)

## Milestone 3: stand inside the image

You can perform the whole dance from a shell, and you should — it is
six commands, and watching `/` change under you is worth more than
reading about it. `unshare -Urm` gives you a user namespace (so you may
mount) and a mount namespace (so your damage is private):

```sh
$ cd bundle
$ unshare -U -r -m
# mount --make-rprivate /              # step 1
# mount --bind rootfs rootfs           # step 2
# cd rootfs                            # step 3
# mkdir -p oldroot
# pivot_root . oldroot                 # steps 4–5, the explicit form
# cd /
# ls
bin  dev  etc  home  lib  lib64  oldroot  proc  root  sys  tmp  usr  var
# ls /oldroot/home                     # the host is STILL reachable
your-username
# umount -l /oldroot && rmdir /oldroot
# ls /oldroot
ls: /oldroot: No such file or directory
# cat /etc/hostname
```

Read those last four commands again, because they are the entire point
of this lesson. Between `pivot_root` and `umount -l`, the host's
filesystem is sitting at `/oldroot`, fully readable — that is the red
edge in the diagram, and a container that got that far and stopped
would be no container at all. The unmount is what closes it.

(This shell version uses an explicit `oldroot` directory, which is the
easy form to *watch*. The `pivot_root(".", ".")` idiom your runtime
uses does the same thing without needing that directory to exist in the
image — worth having seen both.)

Then `exit`: the mount namespace evaporates, and your real `/` was
never touched. Nothing you just did needed root.

## Challenge: Jail a Path {#secure-join points=15}

Implement `secure_join(root, path, out, cap)`: join `path` onto `root`
so the result can never land outside `root`. `root` is absolute with no
trailing slash. Process `path` one component at a time: skip empty
components and `.`; `..` pops the last component but never above `root`;
anything else (including names like `...` or `..hidden`) appends. -1
only when the result won't fit in `cap`.

### Starter

```c
#include <stddef.h>
#include <string.h>

/* Join path onto root, resolving "." and ".." lexically, so that the
   result always stays at or below root. root is absolute, no trailing
   slash. Leading slashes on path make no difference: everything is
   relative to root. 0 on success; -1 if the result (plus NUL) does not
   fit in cap bytes, in which case out is unspecified. */
int secure_join(const char *root, const char *path, char *out, size_t cap) {
	/* TODO: copy root, then walk path component by component;
	   ".." pops back at most to root's length */
	(void)root;
	(void)path;
	(void)out;
	(void)cap;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>
#include <string.h>

int secure_join(const char *root, const char *path, char *out, size_t cap);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

static int join_is(const char *path, const char *want) {
	char out[128];
	if (secure_join("/ctr/rootfs", path, out, sizeof out) != 0)
		return 0;
	return strcmp(out, want) == 0;
}

int main(void) {
	check(join_is("etc/passwd", "/ctr/rootfs/etc/passwd"),
	      "test_plain_relative_path");
	check(join_is("/etc/passwd", "/ctr/rootfs/etc/passwd"),
	      "test_absolute_path_stays_inside");
	check(join_is("", "/ctr/rootfs"), "test_empty_path_is_root");
	check(join_is("/", "/ctr/rootfs"), "test_slash_is_root");
	check(join_is(".", "/ctr/rootfs"), "test_dot_is_root");

	check(join_is("a//b///c", "/ctr/rootfs/a/b/c"),
	      "test_repeated_slashes_collapse");
	check(join_is("a/./b/.", "/ctr/rootfs/a/b"), "test_dot_components_vanish");
	check(join_is("a/b/../c", "/ctr/rootfs/a/c"), "test_dotdot_pops");
	check(join_is("a/b/../../d", "/ctr/rootfs/d"), "test_dotdot_pops_twice");
	check(join_is("a/b/../..", "/ctr/rootfs"), "test_pop_back_to_root");

	/* The attacks: .. must never climb above the root. */
	check(join_is("../../etc/shadow", "/ctr/rootfs/etc/shadow"),
	      "test_leading_dotdot_clamped");
	check(join_is("/../..", "/ctr/rootfs"), "test_absolute_dotdot_clamped");
	check(join_is("a/../../../etc", "/ctr/rootfs/etc"),
	      "test_mixed_escape_clamped");

	/* Three dots is a normal file name, not a traversal. */
	check(join_is("...", "/ctr/rootfs/..."), "test_three_dots_is_a_name");
	check(join_is("..hidden", "/ctr/rootfs/..hidden"),
	      "test_dotdot_prefix_is_a_name");

	char tiny[8];
	check(secure_join("/ctr/rootfs", "etc", tiny, sizeof tiny) == -1,
	      "test_root_too_long_fails");
	char small[14];
	check(secure_join("/ctr/rootfs", "etc", small, sizeof small) == -1,
	      "test_truncation_fails");
	char exact[16];
	check(secure_join("/ctr/rootfs", "etc", exact, sizeof exact) == 0 &&
	      strcmp(exact, "/ctr/rootfs/etc") == 0,
	      "test_exact_fit_ok");

	return failed;
}
```

# Lesson: Pretending to Be Root — User Namespaces {#user-namespaces}

Everything so far assumed the runtime is root. User namespaces remove
that assumption — they're how `podman` runs containers for a normal user
account, and how even root-run containers keep "root inside" from being
root outside.

A user namespace gives its processes a private view of uids and gids.
The process believes it's uid 0; the kernel knows better, because every
namespace carries a **mapping table** that translates ids at every
boundary crossing — file ownership checks, sending signals, `stat`
results, all of it:

```d2
direction: right
ctr: "uids inside" {
  shape: sql_table
  "0   (root)": ""
  "1 … 65535": ""
  "65534  (nobody)": ""
}
host: "uids the kernel uses" {
  shape: sql_table
  "100000": ""
  "100001 … 165535": ""
  "any unmapped uid": ""
}
ctr."0   (root)" -> host."100000": "0 100000 65536" {
  style.stroke: "#d97706"
  style.stroke-width: 2
}
ctr."1 … 65535" -> host."100001 … 165535"
host."any unmapped uid" -> ctr."65534  (nobody)": "appears as" {
  style.stroke-dash: 4
}
```

"Root inside" is uid 100000 outside — an account with no special power
on the host. A container that escapes its filesystem finds itself owning
nothing. This is the single biggest actual-security win in the container
toolbox, which is why it's the one piece of isolation with its own
mapping bureaucracy.

## The mapping files

The table is written, not syscalled: after creating the namespace, the
*parent* writes `/proc/<child pid>/uid_map` and `gid_map`. Each line is
three numbers — and note the order, because everyone reverses it once:

```
   inside-id   outside-id   count
       │           │          │
       0        100000      65536      ← container 0..65535
                                          maps to host 100000..165535
```

The kernel enforces ceremony around these writes:

- **One shot.** Each map file can be written exactly once. A typo means
  destroying the namespace and starting over.
- **`setgroups` first.** Before an unprivileged process may write
  `gid_map`, it must write `deny` to `/proc/<pid>/setgroups` — otherwise
  a user could use "drop this group" semantics to *gain* access to files
  a group ban was protecting them from.
- **Privilege to map ranges.** An unprivileged parent may only map a
  single id (its own). Mapping a 65536-id range needs `CAP_SETUID` or
  the setuid helpers `newuidmap`/`newgidmap`, which check
  `/etc/subuid` and `/etc/subgid` for what you've been allotted.
- **Unmapped ids don't vanish** — they appear as the overflow id, 65534
  ("nobody"). A file owned by an unmapped user is readable per its mode
  bits but its ownership is meaningless inside.

Every id the kernel reports or checks goes through the table — in one
direction or the other. Translating both ways is your challenge: given
mappings like `{container_id, host_id, size}`, `idmap_to_host` sends a
container id out, `idmap_to_container` brings a host id in, and either
returns -1 for an id no range covers. runc's version of this function
answers "root inside is who outside?" every time you `docker exec`.

## Milestone 4: be root, own nothing

Watch a single process hold both identities at once. Start a namespace
and leave it running:

```sh
$ unshare -U -r --uts sh -c 'id -u; touch /tmp/made-by-fake-root; exec sleep 300'
0
```

It says uid 0. Now, from another terminal, ask the host who that
process really is and who owns the file it just created:

```
$ pgrep -x sleep
42317
$ grep -E '^Uid|^NSpid' /proc/42317/status
Uid:	1000	1000	1000	1000
NSpid:	42317
$ cat /proc/42317/uid_map
         0       1000          1
$ ls -l /tmp/made-by-fake-root
-rw-r--r-- 1 your-username your-username 0 Jul 26 11:04 /tmp/made-by-fake-root
```

Four views of one fact. Inside: uid 0. To the kernel's accounting: uid
1000, your ordinary account. The `uid_map` line is the translation
table this lesson is about, in the exact `inside outside count` order —
and `0 1000 1` is precisely the mapping your runtime will write, one
id wide, because an unprivileged parent may only map itself.

The file is the punchline. "Root" created it, and it belongs to *you* —
not to root, and not to nobody. A container that escapes this
filesystem is holding an account that can't read your neighbors' files,
can't bind port 80, and can't load a kernel module. That is why
rootless containers are the default in podman and why this course never
asks you for `sudo`.

Your challenge is the lookup that this table implies. Save it as
`src/idmap.c`; `minict` calls it to build the very `0 1000 1` line you
just read.

## Challenge: Translate Both Ways {#idmap points=10}

Implement both directions of range lookup. An id belongs to a range when
it's within `size` ids of the range's start on the relevant side; the
answer is the same offset from the other side's start.

### Starter

```c
#include <stddef.h>

/* One line of uid_map/gid_map: container-side start, host-side start,
   and how many consecutive ids the range covers. */
struct idmap {
	unsigned container_id;
	unsigned host_id;
	unsigned size;
};

/* The host id for a container id, or -1 if no range covers it. */
long long idmap_to_host(const struct idmap *maps, size_t n, unsigned id) {
	/* TODO */
	(void)maps;
	(void)n;
	(void)id;
	return -1;
}

/* The container id for a host id, or -1 if no range covers it. */
long long idmap_to_container(const struct idmap *maps, size_t n, unsigned id) {
	/* TODO */
	(void)maps;
	(void)n;
	(void)id;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>

struct idmap {
	unsigned container_id;
	unsigned host_id;
	unsigned size;
};

long long idmap_to_host(const struct idmap *maps, size_t n, unsigned id);
long long idmap_to_container(const struct idmap *maps, size_t n, unsigned id);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

int main(void) {
	/* The classic rootless map: container 0..65535 -> host 100000..165535 */
	struct idmap simple[] = {{0, 100000, 65536}};

	check(idmap_to_host(simple, 1, 0) == 100000, "test_root_maps_to_100000");
	check(idmap_to_host(simple, 1, 1000) == 101000, "test_offset_within_range");
	check(idmap_to_host(simple, 1, 65535) == 165535, "test_last_id_in_range");
	check(idmap_to_host(simple, 1, 65536) == -1, "test_one_past_range_unmapped");

	check(idmap_to_container(simple, 1, 100000) == 0, "test_reverse_root");
	check(idmap_to_container(simple, 1, 165535) == 65535, "test_reverse_last");
	check(idmap_to_container(simple, 1, 99999) == -1,
	      "test_reverse_below_range");
	check(idmap_to_container(simple, 1, 0) == -1,
	      "test_host_root_not_in_container");

	/* Two ranges: your own uid becomes container root, a subuid
	   block covers the rest. */
	struct idmap rootless[] = {
		{0, 1000, 1},
		{1, 100000, 65535},
	};
	check(idmap_to_host(rootless, 2, 0) == 1000, "test_multi_first_range");
	check(idmap_to_host(rootless, 2, 1) == 100000, "test_multi_second_range");
	check(idmap_to_host(rootless, 2, 65535) == 165534,
	      "test_multi_second_range_end");
	check(idmap_to_container(rootless, 2, 1000) == 0, "test_multi_reverse");
	check(idmap_to_container(rootless, 2, 1001) == -1,
	      "test_gap_between_ranges_unmapped");

	check(idmap_to_host(NULL, 0, 0) == -1, "test_no_mappings");

	/* A size-1 range maps exactly one id. */
	struct idmap one[] = {{5, 7, 1}};
	check(idmap_to_host(one, 1, 5) == 7 && idmap_to_host(one, 1, 4) == -1 &&
	      idmap_to_host(one, 1, 6) == -1,
	      "test_size_one_range");

	return failed;
}
```

# Lesson: Drawing the Line — cgroups v2 {#cgroups}

Namespaces control what a container *sees*; without cgroups it can still
*take* everything — fork-bomb the machine, eat all the RAM, starve every
neighbor. Control groups are the kernel's resource-accounting tree:
every process belongs to exactly one node, and limits set on a node
apply to all its processes together.

Version 2 (the only one modern systems mount, and the only one this
course teaches) is one unified tree under `/sys/fs/cgroup`, and its API
is just files:

```d2
direction: down
root: "/sys/fs/cgroup — the v2 unified tree"
slice: "system.slice · user.slice\n(everything else on the machine)"
ctr: "duck-f00d.scope — your container" {
  shape: sql_table
  procs: "cgroup.procs ← the child pid"
  mem: "memory.max ← \"268435456\""
  cpu: "cpu.max ← \"50000 100000\""
  pids: "pids.max ← \"64\""
}
root -> slice
root -> ctr: { style.stroke: "#d97706"; style.stroke-width: 2 }
```

The runtime's whole cgroup interaction is: `mkdir` a node, `write` a few
files, write the child's pid into `cgroup.procs`, and `rmdir` on
cleanup. No syscalls beyond the filesystem. Two structural rules catch
newcomers: a controller is only usable in a child if the *parent's*
`cgroup.subtree_control` enables it (`echo "+memory +pids" > ...`), and
the **no-internal-process rule** — a cgroup with child cgroups holds
processes only in the leaves. Runtimes make one leaf per container and
sidestep both.

## The three files worth knowing by heart

The OCI config carries resource limits in `linux.resources`; the
runtime's job is translating each into a file write. The formats are
almost — not quite — uniform:

```
OCI config                          file          contents
──────────────────────────────      ───────────   ─────────────────
memory: { limit: 268435456 }    →   memory.max    "268435456"
memory absent / limit: -1       →   memory.max    "max"

cpu: { quota: 50000,
       period: 100000 }         →   cpu.max       "50000 100000"
cpu: { quota: -1 }              →   cpu.max       "max 100000"

pids: { limit: 64 }             →   pids.max      "64"
```

`cpu.max` is the odd one: **two** numbers, `quota period` — "this group
may run `quota` µs of CPU time per `period` µs of wall clock." 50000 in
a 100000 period is half a core; 200000 in 100000 is two cores. An
unlimited quota is the literal string `max`, but the period stays — the
kernel's default is 100000 µs, and a runtime writes that when the config
doesn't say otherwise. "No limit configured" and "limit: -1" both mean
unlimited, and `memory.max`/`pids.max` spell it `max` with no second
number.

(Trivia with a moral: memory.max has a gentler sibling, `memory.high`,
that throttles and reclaims instead of OOM-killing, and orchestrators
increasingly set both — Kubernetes' memory QoS feature does. Formats
grow; functions that produce them should be small and testable — which
is exactly what you're about to write.)

## Rootless cgroups: the one thing you must be given

Namespaces you can create as an ordinary user. Cgroups you cannot —
`/sys/fs/cgroup` is root-owned, and `mkdir` there fails for you. What
makes rootless limits possible is **delegation**: systemd can hand your
user a subtree that you own outright. `systemd-run --user --scope -p
Delegate=yes` starts a scope whose directory is yours to `mkdir` in.

One structural rule bites immediately, and it is the
no-internal-process rule from above. Your shell is *in* the delegated
scope; the moment you create a child cgroup, the scope has children and
may no longer hold processes directly. So the first move is always:
make a leaf for yourself, step into it, and only then enable
controllers for your children.

## Milestone 5: a limit you can feel

```sh
$ systemd-run --user --scope -p Delegate=yes bash
```

Inside that shell:

```sh
$ CG=/sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)
$ mkdir "$CG/sup" && echo $$ > "$CG/sup/cgroup.procs"   # step aside
$ echo "+pids +memory +cpu" > "$CG/cgroup.subtree_control"
$ mkdir "$CG/ctr" && echo 8 > "$CG/ctr/pids.max"
$ cat "$CG/ctr/pids.max"
8
```

Now put a shell in that leaf and ask it for more processes than it is
allowed:

```sh
$ sh -c 'echo $$ > "'$CG'/ctr/cgroup.procs"
  for i in $(seq 20); do sleep 5 & done; wait'
sh: fork: retry: Resource temporarily unavailable
sh: fork: retry: Resource temporarily unavailable
sh: fork: retry: Resource temporarily unavailable
```

The kernel refused. Not the shell, not a policy daemon — `fork`
returned `EAGAIN` because the cgroup was full, and it will do that to a
fork bomb just as flatly. Check the receipt:

```sh
$ cat "$CG/ctr/pids.events"
max 4
```

Four forks denied that time. Your number will differ — the shell keeps
retrying and the `sleep`s keep exiting, so how many requests collide
with the ceiling is a race — but it is always non-zero, and the ceiling
itself never moves. `memory.max` and `cpu.max` work the same way, with
`memory.events` and `cpu.stat` as their receipts. Type `exit` to leave
the scope and systemd removes the whole subtree.

Your challenge is the three formatters that produce the strings you
just echoed by hand. Save them as `src/cgroup.c`. If delegation isn't
available on your machine, `minict` will print a note and run without
limits — every other milestone still works, and the epilogue explains
the fallback.

## Challenge: Speak cgroup {#cgroup-files points=10}

Implement the three formatters. `snprintf` into the caller's buffer;
any value ≤ 0 means unlimited; -1 only when the result doesn't fit.

### Starter

```c
#include <stddef.h>
#include <stdio.h>

/* Format "quota period" for cpu.max. quota <= 0 -> "max"; period <= 0
   -> the kernel default period, 100000. 0 on success, -1 if the result
   (plus NUL) does not fit in cap bytes. */
int cg_cpu_max(long long quota, long long period, char *out, size_t cap) {
	/* TODO */
	(void)quota;
	(void)period;
	(void)out;
	(void)cap;
	return -1;
}

/* Format memory.max: the byte count, or "max" when limit <= 0. */
int cg_memory_max(long long limit, char *out, size_t cap) {
	/* TODO */
	(void)limit;
	(void)out;
	(void)cap;
	return -1;
}

/* Format pids.max: the count, or "max" when limit <= 0. */
int cg_pids_max(long long limit, char *out, size_t cap) {
	/* TODO */
	(void)limit;
	(void)out;
	(void)cap;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>
#include <string.h>

int cg_cpu_max(long long quota, long long period, char *out, size_t cap);
int cg_memory_max(long long limit, char *out, size_t cap);
int cg_pids_max(long long limit, char *out, size_t cap);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

int main(void) {
	char out[32];

	check(cg_cpu_max(50000, 100000, out, sizeof out) == 0 &&
	      strcmp(out, "50000 100000") == 0,
	      "test_cpu_half_a_core");
	check(cg_cpu_max(200000, 100000, out, sizeof out) == 0 &&
	      strcmp(out, "200000 100000") == 0,
	      "test_cpu_two_cores");
	check(cg_cpu_max(-1, 100000, out, sizeof out) == 0 &&
	      strcmp(out, "max 100000") == 0,
	      "test_cpu_unlimited_quota");
	check(cg_cpu_max(0, 0, out, sizeof out) == 0 &&
	      strcmp(out, "max 100000") == 0,
	      "test_cpu_unset_uses_default_period");
	check(cg_cpu_max(25000, 0, out, sizeof out) == 0 &&
	      strcmp(out, "25000 100000") == 0,
	      "test_cpu_quota_with_default_period");

	check(cg_memory_max(268435456, out, sizeof out) == 0 &&
	      strcmp(out, "268435456") == 0,
	      "test_memory_bytes");
	check(cg_memory_max(-1, out, sizeof out) == 0 && strcmp(out, "max") == 0,
	      "test_memory_unlimited");
	check(cg_memory_max(0, out, sizeof out) == 0 && strcmp(out, "max") == 0,
	      "test_memory_unset");

	check(cg_pids_max(64, out, sizeof out) == 0 && strcmp(out, "64") == 0,
	      "test_pids_limit");
	check(cg_pids_max(-1, out, sizeof out) == 0 && strcmp(out, "max") == 0,
	      "test_pids_unlimited");

	char tiny[4];
	check(cg_cpu_max(50000, 100000, tiny, sizeof tiny) == -1,
	      "test_cpu_truncation_fails");
	check(cg_memory_max(268435456, tiny, sizeof tiny) == -1,
	      "test_memory_truncation_fails");
	check(cg_memory_max(-1, tiny, sizeof tiny) == 0 &&
	      strcmp(tiny, "max") == 0,
	      "test_max_fits_exactly");

	return failed;
}
```

# Lesson: The Lifecycle — create, start, kill, delete {#lifecycle}

The OCI spec doesn't just define the config format — it defines the
**operations** a runtime must expose and the **states** a container
moves through. This is the contract `containerd` programs against: it
calls `runc create`, later `runc start`, polls `runc state`, and
finally `runc delete`, and it can do that against any compliant runtime
because the state machine is nailed down:

```d2
grid-columns: 3
grid-gap: 60
creating: { shape: oval }
created: { shape: oval }
running: { shape: oval }
sp: "" { style.opacity: 0 }
gone: "deleted" {
  shape: oval
  style.stroke-dash: 4
  style.font-color: "#9ca3af"
}
stopped: { shape: oval }
creating -> created: "create()"
created -> running: "start"
created -> stopped: "kill → init dies"
running -> stopped: "process exits"
stopped -> gone: "delete"
```

(`kill` itself doesn't move the machine — see below. The freezer-based
`paused` state some runtimes add is an optional extension; this course
sticks to the mandatory core.)

Why is create/start split at all? Because a huge amount of setup must
happen *inside* the container-to-be — namespaces entered, filesystem
pivoted, cgroup joined — before the user's process is exec'd, and the
caller often needs to act in that gap: attach the console, wire up
networking (that's when CNI — Container Network Interface — plugins
run), run hooks. So `create` does
all the setup and then the init process **parks**, blocked on a pipe,
with the user's command not yet started. `start` writes one byte into
that pipe; init wakes and calls `execve`. Docker's `docker create` /
`docker start` is this seam, exposed.

The transition table, spelled out:

- `creating → created` — the create operation finishes its setup.
- `created → running` — `start`: the parked init execs the user command.
- `created/running → stopped` — the container process **exits** (on its
  own, or because a signal landed). This is the only edge that reflects
  reality rather than a request.
- `kill` — *valid* in created and running, but it only sends a signal;
  the state doesn't change until the process actually dies. A `SIGTERM`
  a process ignores leaves it `running`, correctly.
- `stopped → deleted` — `delete` removes the cgroup node, the state
  file, everything. Deleting a running container is an error (runc
  makes you `kill` first or pass `--force`).
- Everything else — starting a stopped container, deleting a running
  one, double-starting — is an error the runtime must refuse.

runc tracks the current state in a `state.json` per container; the
`state` operation reports it (`ociVersion`, container id, `status`,
`pid`, bundle path) so orchestrators can reconcile. Your challenge is
the machine itself: one function, the full table, every invalid edge
refused.

## How init parks: the exec fifo

One implementation detail deserves spelling out, because "blocked on a
pipe" glosses over a real problem. `create` exits. If the pipe's write
end lived in `create`, it would close on exit and init's `read` would
return 0 immediately — the container would start itself.

runc's answer is a **FIFO** (a named pipe) in the state directory,
and the trick is which end each side opens:

- `create` makes the fifo, opens it `O_RDWR`, and lets the cloned child
  inherit that fd. `O_RDWR` is the load-bearing flag: opening a fifo
  read-only blocks until a writer shows up, and write-only fails with
  `ENXIO` when no reader exists — `O_RDWR` does neither. Init then
  blocks in `read()`, which is the parked state.
- `start` opens the same fifo `O_WRONLY` and writes one byte. Init
  wakes and calls `execve`.

Because the child holds an inherited copy of the open file description,
the fifo keeps a reader even after `create` is long gone. The container
can sit parked for hours; `state` will keep saying `created`.

## Milestone 6: the seam, visible

Once the epilogue's `container.c` and `main.c` are in place — or if you
are reading ahead, after you finish the course — this is the sequence
that proves the state machine is real:

```
$ minict create demo ./bundle
minict: created demo — host pid 40219, pid 1 inside
$ minict state demo
{
  "ociVersion": "1.2.0",
  "status": "created",
  "pid": 40219,
  "bundle": "/home/you/minict/bundle"
}
```

The container exists. Its namespaces are made, its rootfs is pivoted,
its cgroup is set — and `/bin/sh` has not run. Prove it: `ps -p 40219`
shows the process alive, and nothing has been executed. Then try to
skip a step:

```
$ minict delete demo
minict: cannot delete a created container
```

That refusal is `lifecycle_next(ST_CREATED, EV_DELETE)` returning -1 —
your table, enforcing the spec. Now use the seam properly:

```
$ minict start demo
/ # exit
$ minict state demo | grep status
  "status": "stopped",
$ minict delete demo
minict: deleted demo
```

Save your state machine as `src/state.c`. It grows one companion in the
project — the `state.json` reader and writer — and that pairing is the
whole `state` operation.

## Challenge: The State Machine {#lifecycle-fsm points=10}

Implement `lifecycle_next(state, event)`: return the next state, the
same state for a valid-but-stateless `kill`, or -1 for any transition
the spec forbids.

### Starter

```c
enum ctr_state {
	ST_CREATING,
	ST_CREATED,
	ST_RUNNING,
	ST_STOPPED,
	ST_GONE,
};

enum ctr_event {
	EV_CREATE_DONE, /* the create operation finished its setup   */
	EV_START,       /* the start operation                       */
	EV_KILL,        /* the kill operation: send a signal         */
	EV_PROC_EXIT,   /* the container process actually died       */
	EV_DELETE,      /* the delete operation                      */
};

/* The next state, or -1 if the spec forbids this event in this state.
   EV_KILL is valid in created and running but returns the state
   unchanged — signals don't change state, exits do. */
int lifecycle_next(int state, int event) {
	/* TODO */
	(void)state;
	(void)event;
	return -1;
}
```

### Tests

```c
#include <stdio.h>

enum ctr_state {
	ST_CREATING,
	ST_CREATED,
	ST_RUNNING,
	ST_STOPPED,
	ST_GONE,
};

enum ctr_event {
	EV_CREATE_DONE,
	EV_START,
	EV_KILL,
	EV_PROC_EXIT,
	EV_DELETE,
};

int lifecycle_next(int state, int event);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

int main(void) {
	/* The happy path: create, start, exit, delete. */
	check(lifecycle_next(ST_CREATING, EV_CREATE_DONE) == ST_CREATED,
	      "test_create_finishes");
	check(lifecycle_next(ST_CREATED, EV_START) == ST_RUNNING,
	      "test_start_runs");
	check(lifecycle_next(ST_RUNNING, EV_PROC_EXIT) == ST_STOPPED,
	      "test_exit_stops");
	check(lifecycle_next(ST_STOPPED, EV_DELETE) == ST_GONE,
	      "test_delete_removes");

	/* kill is valid on created and running, and changes nothing by
	   itself — the state moves when the process actually exits. */
	check(lifecycle_next(ST_RUNNING, EV_KILL) == ST_RUNNING,
	      "test_kill_running_keeps_state");
	check(lifecycle_next(ST_CREATED, EV_KILL) == ST_CREATED,
	      "test_kill_created_keeps_state");
	check(lifecycle_next(ST_CREATED, EV_PROC_EXIT) == ST_STOPPED,
	      "test_killed_before_start_stops");

	/* Everything else is an error. */
	check(lifecycle_next(ST_RUNNING, EV_START) == -1,
	      "test_double_start_rejected");
	check(lifecycle_next(ST_STOPPED, EV_START) == -1,
	      "test_start_after_stop_rejected");
	check(lifecycle_next(ST_RUNNING, EV_DELETE) == -1,
	      "test_delete_running_rejected");
	check(lifecycle_next(ST_CREATED, EV_DELETE) == -1,
	      "test_delete_created_rejected");
	check(lifecycle_next(ST_STOPPED, EV_KILL) == -1,
	      "test_kill_stopped_rejected");
	check(lifecycle_next(ST_CREATING, EV_START) == -1,
	      "test_start_before_created_rejected");
	check(lifecycle_next(ST_GONE, EV_CREATE_DONE) == -1,
	      "test_gone_is_terminal");

	return failed;
}
```

# Lesson: A Terminal in the Box — PTYs and the Console Socket {#console}

The `-it` in `docker run -it alpine sh` looks like an afterthought —
two letters — but it flips the container into a different mode of
existence. Programs *check* what their stdio is: `isatty(0)` asks the
kernel "is this a terminal?" A shell that hears yes prints a prompt
and enables line editing and job control; `sudo` will dare to ask for
a password; `ls` picks colors. Wire the same programs to a pipe and
they all go quiet and batch-shaped. So an interactive container needs
its process to hold a *real terminal* — and since no physical teletype
has been wired to a Unix machine in decades, the kernel provides fake
ones.

## The pseudoterminal pair

A **PTY** is two connected devices pretending to be one serial line.
Open `/dev/ptmx` (the portable wrapper is `posix_openpt`) and you get
back the **master** fd, and the kernel conjures a matching **slave**
device, `/dev/pts/N` — `ptsname` tells you which N, and
`grantpt`/`unlockpt` make it openable. Bytes written on one end come
out readable on the other, but not directly: in between sits the
**line discipline**, the same kernel layer a hardware terminal gets.
It echoes keystrokes back, holds input until Enter, and — the crucial
one — turns a `0x03` byte (Ctrl-C) into a `SIGINT` delivered to the
slave's foreground process group. Window size lives here too
(`TIOCSWINSZ`), which is what the spec's optional `consoleSize` field
feeds — and "Runtimes MUST ignore `consoleSize` if `terminal` is
`false` or unset."

In config.json all of this is one field: `process.terminal` (bool,
OPTIONAL, defaults to false). The spec's gloss is exactly the plan:
"a pseudoterminal pair is allocated for the process and the
pseudoterminal pty is duplicated on the process's standard streams."

## Plugging the slave in

Where should the runtime open `/dev/ptmx`? *Inside the container* —
after the mount namespace exists and the container's own `devpts`
instance is mounted at `/dev/pts` — so the slave node lives in the
container's tree and its ownership follows the id mappings from the
user-namespaces lesson. Then init, parked before its `execve`, wires
itself up:

1. `setsid()` — become a session leader; a process that isn't one
   cannot acquire a controlling terminal.
2. `ioctl(slave, TIOCSCTTY, 0)` — adopt the slave as the
   **controlling terminal**, the thing that decides which process
   group that Ctrl-C `SIGINT` lands on.
3. `dup2` the slave onto fds 0, 1, and 2, and close the original.

From that point on the user's command isn't being fooled about having
a terminal. It has one.

## Who keeps the master? The console socket

Here is the wrinkle the lifecycle lesson set up: `create` finishes
its work and *exits* — init parks alone, and there is no long-running
runtime process. But `create` is also the code that just allocated
the PTY, so the master fd is about to die with it, taking the
terminal's far end into the void.

runc's answer — and yours — is the **console socket**. The caller
(containerd, or you at a shell) listens on an `AF_UNIX` socket and
hands its path to `create` as `--console-socket`. After allocating
the PTY, the runtime connects to that socket and sends *the master
fd itself* through it: `sendmsg` with an `SCM_RIGHTS` control
message. SCM_RIGHTS is the kernel's fd-passing mechanism — the
receiver calls `recvmsg` and finds a brand-new entry in its own fd
table referring to the *same open file*, the PTY master. Not the fd
number; the open file. The runtime can now exit in peace. Whoever
holds that socket holds the container's keyboard and screen — that
is the far end of `docker attach`.

The full handoff inside `create` — the slave stays in the box as
stdio; the master leaves through the console socket:

```d2
direction: right
rt: "your runtime\ncreate()" {
  style.stroke: "#d97706"
  style.stroke-width: 3
}
pty: "PTY pair" {
  shape: sql_table
  master: "master fd"
  slave: "/dev/pts/0"
}
init: "container init\nstdio = slave\nTIOCSCTTY"
caller: "caller holds\nconsole.sock"
rt -> pty: "posix_openpt"
pty.slave -> init: "dup2 → 0,1,2"
pty.master -> caller: "SCM_RIGHTS"
```

## Pass-through, and two rules worth refusing

With `terminal: false` there is no PTY at all: stdio is plain
**pipes** to wherever the caller pointed them. Logs flow out, EOF
flows in, nothing echoes, Ctrl-C is nobody's business. For batch
workloads that's exactly right.

What's never right is a config and a command line that disagree, and
runc refuses both directions (its `checkTerminal`, translated to your
create/start model, which always detaches):

- `terminal: true` but no console socket — error: "cannot allocate
  tty if runc will detach without setting console socket." The
  master would be orphaned the moment create returns.
- A console socket but `terminal` false or absent — error: "cannot
  use console socket if runc will not detach or allocate tty." A
  socket nobody will ever send an fd down means the caller is
  confused, and refusing beats surprising them.

One more sharp edge, an old friend from the secure-join lesson:
validate paths *at plan time*. `AF_UNIX` socket paths have a hard
kernel limit — `sun_path` is 108 bytes on Linux, NUL included — so a
console-socket path longer than 107 characters can never connect and
deserves rejection before a single syscall happens.

## An ordering problem worth seeing coming

The plan says "allocate the PTY inside the container." The console
socket, meanwhile, is a path on the *host* — `/tmp/console.sock`.
After `pivot_root` the container cannot name that path at all. So the
handoff has to straddle the pivot, and the order is forced:

1. **Before the pivot**, `connect()` to the console socket. Keep the
   fd. A connected socket is an open file description; it does not care
   what `/` points at afterwards.
2. **After the pivot**, mount the container's own `devpts` at
   `/dev/pts`, then open the multiplexor and allocate the pair.
3. Send the master down the fd from step 1, `close` it, and wire the
   slave onto 0/1/2.

Step 2 has a wrinkle that will cost you twenty minutes if nobody warns
you: `posix_openpt` opens `/dev/ptmx`, but a freshly mounted devpts
instance puts its multiplexor at `/dev/pts/ptmx`. Real container images
ship a `/dev/ptmx → pts/ptmx` symlink for exactly this reason; your
runtime creates it if the image didn't.

## Milestone 7: attach to your own container

The far end needs a program: something that listens on the socket,
receives the fd, and relays your keyboard. That's `attach.c`, printed
in full in the epilogue — about 120 lines, most of it `poll`. Run it
first, in one terminal:

```
$ minict-attach /tmp/console.sock
minict-attach: waiting on /tmp/console.sock
```

Then, in another, create a container whose config says
`"terminal": true`:

```
$ minict create demo ./bundle --console-socket /tmp/console.sock
minict: created demo — host pid 40219, pid 1 inside
$ minict start demo
```

Back in the first terminal:

```
minict-attach: got the pty master (fd 5)
/ # tty
/dev/pts/0
/ # echo isatty=$( [ -t 0 ] && echo YES || echo no )
isatty=YES
/ # exit
minict-attach: console closed
```

A prompt, because the shell asked `isatty(0)` and heard yes. Line
editing works. Ctrl-C interrupts instead of killing your terminal,
because the `SIGINT` is generated by the container's line discipline
and delivered to the container's foreground process group. `tty` names
`/dev/pts/0` — device zero of an instance that exists only inside this
container; your host's `/dev/pts/0` is somebody else's terminal
entirely.

Try the two refusals too, and watch your own validation fire:

```
$ minict create bad ./bundle           # config says terminal: true
minict: terminal/console-socket mismatch
```

Save your work as `src/console.c`.

## Challenge: Plan the Console {#console-plan points=15}

Two functions, both pure decision logic. `json_bool` teaches your
scanner the one JSON type it can't read yet — and unlike its cousins
it must report *three* outcomes, because absent (defaults to false)
and present-as-false take different paths through the rules.
`console_plan` then makes the call that `create` would act on: which
mode, which socket path, or a refusal. The tests link `json_bool`
directly, so leave it non-static.

### Starter

```c
#include <stddef.h>
#include <string.h>

#define CONSOLE_PIPES 0 /* pass-through: stdio is plain pipes      */
#define CONSOLE_PTY   1 /* new terminal: PTY master → console.sock */

/* sizeof(struct sockaddr_un.sun_path) on Linux, NUL included. */
#define SUN_PATH_MAX 108

struct console_plan {
	int mode;                       /* CONSOLE_PIPES or CONSOLE_PTY  */
	char socket_path[SUN_PATH_MAX]; /* "" when mode is CONSOLE_PIPES */
};

/* ---- given: the scanner core from earlier lessons ---- */

static int is_ws(char c) {
	return c == ' ' || c == '\t' || c == '\n' || c == '\r';
}

static const char *after_key(const char *doc, const char *key) {
	size_t klen = strlen(key);
	for (const char *p = doc; (p = strchr(p, '"')) != NULL; p++) {
		if (strncmp(p + 1, key, klen) != 0 || p[1 + klen] != '"')
			continue;
		const char *q = p + klen + 2;
		while (is_ws(*q))
			q++;
		if (*q != ':')
			continue;
		return q + 1;
	}
	return NULL;
}

/* ---- implement these ---- */

/* Three-way: 1 if `key` is present with a literal true/false value
   (*out = 1 or 0), 0 if the key is absent, -1 if the value is
   anything else. Absent and false are different answers — the
   caller needs to know which rule it is applying. */
int json_bool(const char *doc, const char *key, int *out) {
	/* TODO: after_key, skip whitespace, match "true" or "false" */
	(void)after_key;
	(void)doc;
	(void)key;
	(void)out;
	return -1;
}

/* Decide the stdio wiring for a create that will detach.
   doc is the config's process object; socket_path is the caller's
   --console-socket argument, or NULL if it wasn't given.
   terminal true → CONSOLE_PTY, socket required, path copied.
   terminal false or absent → CONSOLE_PIPES, socket must be NULL.
   Malformed terminal, missing/empty/oversized socket path: -1. */
int console_plan(const char *doc, const char *socket_path,
                 struct console_plan *p) {
	/* TODO: json_bool, then the two runc rules */
	(void)json_bool;
	(void)doc;
	(void)socket_path;
	(void)p;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>
#include <string.h>

#define CONSOLE_PIPES 0
#define CONSOLE_PTY   1
#define SUN_PATH_MAX  108

struct console_plan {
	int mode;
	char socket_path[SUN_PATH_MAX];
};

int json_bool(const char *doc, const char *key, int *out);
int console_plan(const char *doc, const char *socket_path,
                 struct console_plan *p);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

int main(void) {
	struct console_plan p;
	int b;

	/* json_bool: three outcomes, not two. */
	b = -7;
	check(json_bool("{ \"terminal\": true }", "terminal", &b) == 1 &&
	      b == 1, "test_bool_true");
	b = -7;
	check(json_bool("{ \"terminal\": false }", "terminal", &b) == 1 &&
	      b == 0, "test_bool_false");
	check(json_bool("{ \"tty\": true }", "terminal", &b) == 0,
	      "test_bool_absent_key");
	check(json_bool("{ \"terminal\": null }", "terminal", &b) == -1,
	      "test_bool_null_is_malformed");
	check(json_bool("{ \"terminal\": \"true\" }", "terminal", &b) == -1,
	      "test_bool_string_is_malformed");
	b = -7;
	check(json_bool("{ \"terminal\" \t:\n\ttrue }", "terminal", &b) == 1 &&
	      b == 1, "test_bool_whitespace");

	/* The two happy paths. */
	memset(&p, 0x55, sizeof p);
	check(console_plan("{ \"terminal\": true }",
	                   "/run/ctr/console.sock", &p) == 0 &&
	      p.mode == CONSOLE_PTY &&
	      strcmp(p.socket_path, "/run/ctr/console.sock") == 0,
	      "test_terminal_and_socket_is_pty");
	memset(&p, 0x55, sizeof p);
	check(console_plan("{ \"args\": [ \"sh\" ] }", NULL, &p) == 0 &&
	      p.mode == CONSOLE_PIPES && p.socket_path[0] == '\0',
	      "test_absent_terminal_is_pipes");
	check(console_plan("{ \"terminal\": false }", NULL, &p) == 0 &&
	      p.mode == CONSOLE_PIPES,
	      "test_terminal_false_is_pipes");
	check(console_plan("{ \"args\": [ \"sh\", \"-c\" ], \"terminal\": true }",
	                   "/s", &p) == 0 && p.mode == CONSOLE_PTY,
	      "test_terminal_after_other_keys");

	/* runc's two refusals. */
	check(console_plan("{ \"terminal\": true }", NULL, &p) == -1,
	      "test_tty_without_socket_rejected");
	check(console_plan("{ \"terminal\": false }", "/tmp/c.sock", &p) == -1,
	      "test_socket_without_tty_rejected");
	check(console_plan("{}", "/tmp/c.sock", &p) == -1,
	      "test_socket_with_absent_terminal_rejected");

	/* A malformed config refuses, whatever the command line says. */
	check(console_plan("{ \"terminal\": \"true\" }", "/tmp/c.sock", &p) == -1,
	      "test_string_terminal_rejected");
	check(console_plan("{ \"terminal\": 1 }", "/tmp/c.sock", &p) == -1,
	      "test_numeric_terminal_rejected");

	/* sun_path is 108 bytes with the NUL: 107 chars fit, 108 don't. */
	char longp[120];
	memset(longp, 'a', sizeof longp);
	longp[0] = '/';
	longp[107] = '\0';
	memset(&p, 0x55, sizeof p);
	check(console_plan("{ \"terminal\": true }", longp, &p) == 0 &&
	      strcmp(p.socket_path, longp) == 0,
	      "test_107_char_path_fits");
	longp[107] = 'a';
	longp[108] = '\0';
	check(console_plan("{ \"terminal\": true }", longp, &p) == -1,
	      "test_108_char_path_rejected");
	check(console_plan("{ \"terminal\": true }", "", &p) == -1,
	      "test_empty_socket_path_rejected");

	return failed;
}
```

# Final Challenge: plan_container {#final points=50}

Every piece you've built is one column of the same translation: the
config document on one side, kernel-ready values on the other. The last
step is the function that owns that translation end to end — the one a
`create` implementation calls before it dares touch a syscall:

```d2
direction: down
cfg: "config.json — the author's intent" { shape: package }
plan: "struct plan" {
  shape: sql_table
  "rootfs ← secure_join": ""
  "hostname ← json_str": ""
  "clone_flags ← nsflags": ""
  "cpu · memory · pids ← cg fns": ""
}
sys: "create + start — clone · sethostname · cgroup writes · pivot_root · execve"
cfg -> plan: "plan_container()" {
  style.stroke: "#d97706"
  style.stroke-width: 2
}
plan -> sys
```

Planning before executing isn't just tidiness. A runtime that validates
*everything* first can reject a bad config having touched nothing — no
half-made namespaces to unwind, no orphaned cgroup nodes. runc calls
this phase "loading the spec," and it's where most config attacks die.
(Your console plan stays a separate call beside this one — its input
includes the `--console-socket` flag, which lives on the command
line, not in the document `plan_container` translates.)

`plan_container(bundle, doc, p)` handles this config subset:

```json
{
  "ociVersion": "1.2.0",
  "hostname": "duck",
  "root": { "path": "rootfs" },
  "linux": {
    "namespaces": [ { "type": "pid" }, { "type": "mount" } ],
    "resources": {
      "memory": { "limit": 268435456 },
      "cpu": { "quota": 50000, "period": 100000 },
      "pids": { "limit": 64 }
    }
  }
}
```

The rules — each one enforced by the tests:

- **Version gate.** `ociVersion` must exist and start with `"1."`; a
  missing or 0.x/2.x version is `-1`. (Majors may break meaning; a
  runtime that guesses is a runtime that runs something other than what
  was asked.)
- **Rootfs.** `root.path` is required, and is joined to the bundle
  directory with `secure_join` — a config whose path says `../../etc`
  lands harmlessly *inside* the bundle. (In this subset, `"path"`
  appears only under `"root"`, so a whole-document scan finds the right
  one.)
- **Hostname** is optional; default to `""`.
- **Namespaces.** No `"namespaces"` key means zero clone flags. If the
  array exists, collect every entry's `"type"` and fold them with
  `nsflags` — so unknown or duplicated types are rejected, exactly as in
  lesson one.
- **Resources.** All optional, defaulting to unlimited: quota/period →
  `cpu_max`, memory limit → `memory_max`, pids limit → `pids_max`, via
  your lesson-six formatters.

Two parsing problems are genuinely new, and they're the meat of the
challenge:

**Integers.** `json_int` is `json_str`'s sibling for numeric values:
find the key (the given `after_key` helper does the find-a-real-key scan
and returns a pointer just past the `:`), skip whitespace, accept an
optional `-`, then digits. No digits where a number should be is an
error.

**Scoped scans.** Both `memory` and `pids` carry a key named `"limit"` —
a whole-document scan for `"limit"` would hand you the memory limit
twice. The fix uses position: `after_key(doc, "memory")` returns a
pointer *into the document at the memory section*; scanning for
`"limit"` from there finds memory's own. Scan for `"pids"` first and
you find its `"limit"` instead. The same trick reads the namespace
array: find `"namespaces"`, find the `[`, and collect each `"type"`
that appears before the closing `]`. One caution: that closing `]` is
doing real work, and the resource sections need the same courtesy.
Scan from `"memory"` with nothing to stop you, and an empty
`"memory": {}` sends you sailing into the pids section to borrow *its*
limit. The objects in this subset are flat, so the first `}` after the
section opens is its end — a `"limit"` found beyond it belongs to the
next section, and a section without its own simply defaults to
unlimited. (This position game is exactly the
order-independence a real parser gives you for free — build it once by
hand and you'll never wonder what `encoding/json` is doing for you
again.)

Wire it all together and the returned plan is everything a `create`
needs: a jailed rootfs path, a hostname, a clone mask, and three
cgroup file payloads.

### Starter

```c
#include <stddef.h>
#include <stdio.h>
#include <string.h>

#define CLONE_NEWNS     0x00020000u
#define CLONE_NEWCGROUP 0x02000000u
#define CLONE_NEWUTS    0x04000000u
#define CLONE_NEWIPC    0x08000000u
#define CLONE_NEWUSER   0x10000000u
#define CLONE_NEWPID    0x20000000u
#define CLONE_NEWNET    0x40000000u

/* Everything create() needs to know, decided before any syscall. */
struct plan {
	char rootfs[256];        /* absolute, jailed inside the bundle  */
	char hostname[64];       /* "" if the config doesn't set one    */
	unsigned long clone_flags;
	char cpu_max[32];        /* contents for cpu.max                */
	char memory_max[32];     /* contents for memory.max             */
	char pids_max[32];       /* contents for pids.max               */
};

/* ---- given: the pieces you built in earlier lessons ---- */

static int is_ws(char c) {
	return c == ' ' || c == '\t' || c == '\n' || c == '\r';
}

int json_str(const char *doc, const char *key, char *out, size_t cap) {
	size_t klen = strlen(key);
	if (cap == 0)
		return -1;
	for (const char *p = doc; (p = strchr(p, '"')) != NULL; p++) {
		if (strncmp(p + 1, key, klen) != 0 || p[1 + klen] != '"')
			continue;
		const char *q = p + klen + 2;
		while (is_ws(*q))
			q++;
		if (*q != ':')
			continue;
		q++;
		while (is_ws(*q))
			q++;
		if (*q != '"')
			return -1;
		q++;
		size_t len = 0;
		while (*q && *q != '"') {
			char c = *q++;
			if (c == '\\' && (*q == '"' || *q == '\\'))
				c = *q++;
			if (len + 1 >= cap)
				return -1;
			out[len++] = c;
		}
		if (*q != '"')
			return -1;
		out[len] = '\0';
		return 0;
	}
	return -1;
}

int nsflags(const char *const names[], size_t n, unsigned long *out) {
	unsigned long flags = 0;
	for (size_t i = 0; i < n; i++) {
		unsigned long f;
		if (strcmp(names[i], "pid") == 0)
			f = CLONE_NEWPID;
		else if (strcmp(names[i], "network") == 0)
			f = CLONE_NEWNET;
		else if (strcmp(names[i], "mount") == 0)
			f = CLONE_NEWNS;
		else if (strcmp(names[i], "ipc") == 0)
			f = CLONE_NEWIPC;
		else if (strcmp(names[i], "uts") == 0)
			f = CLONE_NEWUTS;
		else if (strcmp(names[i], "user") == 0)
			f = CLONE_NEWUSER;
		else if (strcmp(names[i], "cgroup") == 0)
			f = CLONE_NEWCGROUP;
		else
			return -1;
		if (flags & f)
			return -1;
		flags |= f;
	}
	*out = flags;
	return 0;
}

int secure_join(const char *root, const char *path, char *out, size_t cap) {
	size_t rlen = strlen(root);
	if (rlen + 1 > cap)
		return -1;
	memcpy(out, root, rlen + 1);
	size_t len = rlen;
	const char *p = path;
	while (*p) {
		while (*p == '/')
			p++;
		if (!*p)
			break;
		const char *start = p;
		while (*p && *p != '/')
			p++;
		size_t clen = (size_t)(p - start);
		if (clen == 1 && start[0] == '.')
			continue;
		if (clen == 2 && start[0] == '.' && start[1] == '.') {
			while (len > rlen && out[len - 1] != '/')
				len--;
			if (len > rlen)
				len--;
			out[len] = '\0';
			continue;
		}
		if (len + 1 + clen + 1 > cap)
			return -1;
		out[len++] = '/';
		memcpy(out + len, start, clen);
		len += clen;
		out[len] = '\0';
	}
	return 0;
}

int cg_cpu_max(long long quota, long long period, char *out, size_t cap) {
	int n;
	if (period <= 0)
		period = 100000;
	if (quota <= 0)
		n = snprintf(out, cap, "max %lld", period);
	else
		n = snprintf(out, cap, "%lld %lld", quota, period);
	return (n < 0 || (size_t)n >= cap) ? -1 : 0;
}

int cg_memory_max(long long limit, char *out, size_t cap) {
	int n;
	if (limit <= 0)
		n = snprintf(out, cap, "max");
	else
		n = snprintf(out, cap, "%lld", limit);
	return (n < 0 || (size_t)n >= cap) ? -1 : 0;
}

int cg_pids_max(long long limit, char *out, size_t cap) {
	int n;
	if (limit <= 0)
		n = snprintf(out, cap, "max");
	else
		n = snprintf(out, cap, "%lld", limit);
	return (n < 0 || (size_t)n >= cap) ? -1 : 0;
}

/* A pointer just past the ':' after `"key"` used as a key at or after
   doc, or NULL if it never appears. json_str's opening scan, split out
   so number parsing and section scoping can reuse it. */
static const char *after_key(const char *doc, const char *key) {
	size_t klen = strlen(key);
	for (const char *p = doc; (p = strchr(p, '"')) != NULL; p++) {
		if (strncmp(p + 1, key, klen) != 0 || p[1 + klen] != '"')
			continue;
		const char *q = p + klen + 2;
		while (is_ws(*q))
			q++;
		if (*q != ':')
			continue;
		return q + 1;
	}
	return NULL;
}

/* ---- new: implement these ---- */

/* Parse the first integer value of `key` at or after doc into *out.
   0 on success; -1 if the key never appears or its value does not
   start with an optional '-' and at least one digit. */
static int json_int(const char *doc, const char *key, long long *out) {
	/* TODO: after_key, skip whitespace, optional '-', digits */
	(void)after_key;
	(void)doc;
	(void)key;
	(void)out;
	return -1;
}

/* Fold every "type" in the "namespaces" array into clone flags via
   nsflags. No "namespaces" key at all: *out = 0 and success. A
   missing '[', a "type" that isn't a short string, or anything
   nsflags rejects: -1. */
static int ns_clone_flags(const char *doc, unsigned long *out) {
	/* TODO: after_key("namespaces"), find '[' and ']', collect each
	   "type" value that appears between them, hand them to nsflags */
	(void)doc;
	(void)out;
	return -1;
}

/* Validate doc and fill the plan. 0 on success, -1 on any rule
   violation described in the challenge text. */
int plan_container(const char *bundle, const char *doc, struct plan *p) {
	/* TODO: version gate, rootfs, hostname, namespaces, resources */
	(void)json_int;
	(void)ns_clone_flags;
	(void)bundle;
	(void)doc;
	(void)p;
	return -1;
}
```

### Tests

```c
#include <stddef.h>
#include <stdio.h>
#include <string.h>

#define CLONE_NEWNS     0x00020000u
#define CLONE_NEWCGROUP 0x02000000u
#define CLONE_NEWUTS    0x04000000u
#define CLONE_NEWIPC    0x08000000u
#define CLONE_NEWUSER   0x10000000u
#define CLONE_NEWPID    0x20000000u
#define CLONE_NEWNET    0x40000000u

struct plan {
	char rootfs[256];
	char hostname[64];
	unsigned long clone_flags;
	char cpu_max[32];
	char memory_max[32];
	char pids_max[32];
};

int plan_container(const char *bundle, const char *doc, struct plan *p);

static int failed;

static void check(int ok, const char *name) {
	if (ok) {
		printf("--- PASS: %s\n", name);
	} else {
		printf("--- FAIL: %s\n", name);
		failed++;
	}
}

static const char *full_config =
	"{\n"
	"  \"ociVersion\": \"1.2.0\",\n"
	"  \"hostname\": \"duck\",\n"
	"  \"root\": { \"path\": \"rootfs\" },\n"
	"  \"process\": { \"args\": [\"/bin/sh\"] },\n"
	"  \"linux\": {\n"
	"    \"namespaces\": [\n"
	"      { \"type\": \"pid\" },\n"
	"      { \"type\": \"mount\" },\n"
	"      { \"type\": \"uts\" },\n"
	"      { \"type\": \"network\" }\n"
	"    ],\n"
	"    \"resources\": {\n"
	"      \"memory\": { \"limit\": 268435456 },\n"
	"      \"cpu\": { \"quota\": 50000, \"period\": 100000 },\n"
	"      \"pids\": { \"limit\": 64 }\n"
	"    }\n"
	"  }\n"
	"}\n";

static const char *minimal_config =
	"{ \"ociVersion\": \"1.0.2\", \"root\": { \"path\": \"rootfs\" } }";

int main(void) {
	struct plan p;

	memset(&p, 0, sizeof p);
	check(plan_container("/run/bundle", full_config, &p) == 0,
	      "test_full_config_accepted");
	check(strcmp(p.rootfs, "/run/bundle/rootfs") == 0,
	      "test_rootfs_joined_to_bundle");
	check(strcmp(p.hostname, "duck") == 0, "test_hostname_extracted");
	check(p.clone_flags == (CLONE_NEWPID | CLONE_NEWNS |
	                        CLONE_NEWUTS | CLONE_NEWNET),
	      "test_clone_flags_from_namespaces");
	check(strcmp(p.cpu_max, "50000 100000") == 0, "test_cpu_max_formatted");
	check(strcmp(p.memory_max, "268435456") == 0, "test_memory_max_formatted");
	check(strcmp(p.pids_max, "64") == 0, "test_pids_max_formatted");

	memset(&p, 0, sizeof p);
	check(plan_container("/b", minimal_config, &p) == 0,
	      "test_minimal_config_accepted");
	check(strcmp(p.rootfs, "/b/rootfs") == 0, "test_minimal_rootfs");
	check(p.hostname[0] == '\0', "test_hostname_defaults_empty");
	check(p.clone_flags == 0, "test_no_namespaces_means_no_flags");
	check(strcmp(p.cpu_max, "max 100000") == 0, "test_cpu_defaults_to_max");
	check(strcmp(p.memory_max, "max") == 0, "test_memory_defaults_to_max");
	check(strcmp(p.pids_max, "max") == 0, "test_pids_defaults_to_max");

	/* root.path pointing outside the bundle must be clamped, not obeyed. */
	const char *escape =
		"{ \"ociVersion\": \"1.2.0\","
		"  \"root\": { \"path\": \"../../../etc\" } }";
	memset(&p, 0, sizeof p);
	check(plan_container("/run/bundle", escape, &p) == 0 &&
	      strcmp(p.rootfs, "/run/bundle/etc") == 0,
	      "test_escaping_rootfs_clamped");

	const char *no_version =
		"{ \"root\": { \"path\": \"rootfs\" } }";
	check(plan_container("/b", no_version, &p) == -1,
	      "test_missing_version_rejected");

	const char *bad_version =
		"{ \"ociVersion\": \"0.9.0\", \"root\": { \"path\": \"rootfs\" } }";
	check(plan_container("/b", bad_version, &p) == -1,
	      "test_wrong_major_version_rejected");

	const char *no_root = "{ \"ociVersion\": \"1.2.0\" }";
	check(plan_container("/b", no_root, &p) == -1,
	      "test_missing_root_rejected");

	const char *bad_ns =
		"{ \"ociVersion\": \"1.2.0\", \"root\": { \"path\": \"r\" },"
		"  \"linux\": { \"namespaces\": [ { \"type\": \"banana\" } ] } }";
	check(plan_container("/b", bad_ns, &p) == -1,
	      "test_unknown_namespace_rejected");

	const char *dup_ns =
		"{ \"ociVersion\": \"1.2.0\", \"root\": { \"path\": \"r\" },"
		"  \"linux\": { \"namespaces\": ["
		" { \"type\": \"pid\" }, { \"type\": \"pid\" } ] } }";
	check(plan_container("/b", dup_ns, &p) == -1,
	      "test_duplicate_namespace_rejected");

	/* cpu limits without a period: the default period fills in. */
	const char *quota_only =
		"{ \"ociVersion\": \"1.1.0\", \"root\": { \"path\": \"r\" },"
		"  \"linux\": { \"resources\": {"
		" \"cpu\": { \"quota\": 25000 } } } }";
	memset(&p, 0, sizeof p);
	check(plan_container("/b", quota_only, &p) == 0 &&
	      strcmp(p.cpu_max, "25000 100000") == 0,
	      "test_quota_without_period");

	/* A section with no limit of its own defaults to unlimited — it
	   must not borrow the next section's. Bound each scan at the
	   section's closing '}'. */
	const char *empty_memory =
		"{ \"ociVersion\": \"1.2.0\", \"root\": { \"path\": \"r\" },"
		"  \"linux\": { \"resources\": {"
		" \"memory\": {},"
		" \"pids\": { \"limit\": 64 } } } }";
	memset(&p, 0, sizeof p);
	check(plan_container("/b", empty_memory, &p) == 0 &&
	      strcmp(p.memory_max, "max") == 0 &&
	      strcmp(p.pids_max, "64") == 0,
	      "test_empty_memory_does_not_borrow_pids_limit");

	return failed;
}
```
# Lesson: Epilogue: Run It for Real {#run-it-for-real}

Every challenge in this course graded a decision, and every milestone
let you watch the kernel obey one. This epilogue is where the two
halves meet: the ~350 lines of syscall choreography that turn your
nine solutions into a program you can run. Nothing below is graded and
there is nothing to submit. What there is instead: a working OCI
runtime, on your machine, built by you, that needs no daemon and no
root.

## What's already yours

Six files hold all nine of your solutions — paste each one in under the
header shown, and that file is done:

```
src/oci.c       json_str · nsflags · secure_join · plan_container
                (plus json_int, section_int, ns_clone_flags, after_key
                from the final challenge's starter)
src/nspid.c     nspid_parse
src/idmap.c     idmap_to_host · idmap_to_container
src/cgroup.c    cg_cpu_max · cg_memory_max · cg_pids_max
src/state.c     lifecycle_next
src/console.c   json_bool · console_plan
```

Four of those need a companion the challenges couldn't ask for,
because each one touches the filesystem. They are short, and they are
listed below with the file they belong in.

## The shared header

`src/minict.h` — every declaration in one place, so the files above
compile against the same types:

```c
#ifndef MINICT_H
#define MINICT_H

#include <stddef.h>

#define CLONE_NEWNS_     0x00020000u
#define CLONE_NEWCGROUP_ 0x02000000u
#define CLONE_NEWUTS_    0x04000000u
#define CLONE_NEWIPC_    0x08000000u
#define CLONE_NEWUSER_   0x10000000u
#define CLONE_NEWPID_    0x20000000u
#define CLONE_NEWNET_    0x40000000u

#define CONSOLE_PIPES 0
#define CONSOLE_PTY   1
#define SUN_PATH_MAX  108

struct plan {
	char rootfs[256];
	char hostname[64];
	unsigned long clone_flags;
	char cpu_max[32];
	char memory_max[32];
	char pids_max[32];
};

struct console_plan {
	int mode;
	char socket_path[SUN_PATH_MAX];
};

struct idmap {
	unsigned container_id;
	unsigned host_id;
	unsigned size;
};

enum ctr_state { ST_CREATING, ST_CREATED, ST_RUNNING, ST_STOPPED, ST_GONE };
enum ctr_event { EV_CREATE_DONE, EV_START, EV_KILL, EV_PROC_EXIT, EV_DELETE };

/* --- oci.c : your lesson 1/2/4 solutions --- */
int json_str(const char *doc, const char *key, char *out, size_t cap);
int json_bool(const char *doc, const char *key, int *out);
int nsflags(const char *const names[], size_t n, unsigned long *out);
int secure_join(const char *root, const char *path, char *out, size_t cap);
int plan_container(const char *bundle, const char *doc, struct plan *p);
const char *after_key(const char *doc, const char *key);

/* --- cgroup.c : your lesson 6 solutions + the writes --- */
int cg_cpu_max(long long quota, long long period, char *out, size_t cap);
int cg_memory_max(long long limit, char *out, size_t cap);
int cg_pids_max(long long limit, char *out, size_t cap);
int cgroup_apply(const char *cg, const struct plan *p, int pid);
int cgroup_remove(const char *cg);

/* --- idmap.c : your lesson 5 solution --- */
long long idmap_to_host(const struct idmap *maps, size_t n, unsigned id);
long long idmap_to_container(const struct idmap *maps, size_t n, unsigned id);

/* --- nspid.c : your lesson 3 solution --- */
int nspid_parse(const char *line, int *out, size_t cap);
int nspid_of(int pid, int *out, size_t cap);

/* --- state.c : your lesson 7 solution + state.json --- */
int lifecycle_next(int state, int event);
int state_save(const char *dir, int st, int pid, const char *bundle);
int state_load(const char *dir, int *st, int *pid, char *bundle, size_t cap);
const char *state_name(int st);

/* --- console.c : your lesson 8 solution + the PTY handoff --- */
int console_plan(const char *doc, const char *socket_path,
                 struct console_plan *p);
int pty_open(char *slave_name, size_t cap);
int console_connect(const char *sock_path);
int console_send(int sock_fd, int master_fd);

/* --- container.c --- */
int container_create(const char *id, const char *bundle,
                     const char *console_sock);
int container_start(const char *id);
int container_kill(const char *id, int sig);
int container_delete(const char *id);
int container_state(const char *id);

char *read_file(const char *path);
const char *run_dir(const char *id);

#endif
```

## The heart: container.c

This is the file the whole course has been building toward. The first
half runs *inside* the new namespaces and is, line for line, the
choreography from lessons 3 through 8:

```c
#define _GNU_SOURCE
#include "minict.h"

#include <errno.h>
#include <fcntl.h>
#include <sched.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mount.h>
#include <sys/stat.h>
#include <sys/syscall.h>
#include <sys/wait.h>
#include <unistd.h>

static char child_stack[512 * 1024];

struct child_args {
	const struct plan *plan;
	const struct console_plan *con;
	int con_fd;    /* connected console socket, opened pre-pivot     */
	int mapped[2]; /* parent -> child: "your id maps are written"    */
	int start_fd;  /* the exec fifo, opened before the pivot     */
	char **argv;
};

static void fail(const char *what) {
	fprintf(stderr, "minict: %s: %s\n", what, strerror(errno));
	_exit(1);
}

/* Everything below already runs inside the new namespaces. */
static int child_main(void *arg) {
	struct child_args *a = arg;
	char b;

	close(a->mapped[1]);
	if (read(a->mapped[0], &b, 1) != 1)
		fail("waiting for id maps");
	close(a->mapped[0]);

	if (a->plan->hostname[0] != '\0' &&
	    sethostname(a->plan->hostname, strlen(a->plan->hostname)) != 0)
		fail("sethostname");

	/* The console socket path is a HOST path — reach it while we still
	   can. The connected fd outlives the pivot; the path would not. */
	if (a->con->mode == CONSOLE_PTY) {
		a->con_fd = console_connect(a->con->socket_path);
		if (a->con_fd < 0)
			fail("connect to console socket");
	}

	/* --- lesson 4: the pivot_root dance --- */
	if (mount(NULL, "/", NULL, MS_REC | MS_PRIVATE, NULL) != 0)
		fail("make mounts private");
	if (mount(a->plan->rootfs, a->plan->rootfs, NULL, MS_BIND, NULL) != 0)
		fail("bind rootfs onto itself");
	if (chdir(a->plan->rootfs) != 0)
		fail("chdir rootfs");
	if (syscall(SYS_pivot_root, ".", ".") != 0)
		fail("pivot_root");
	/* /proc must be mounted while the old root is still stacked: inside a
	   user namespace the kernel grants a fresh procfs only when a
	   fully-visible one already exists in this mount namespace. */
	if (mount("proc", "proc", "proc", 0, NULL) != 0)
		fail("mount /proc");
	if (umount2(".", MNT_DETACH) != 0)
		fail("detach old root");
	if (chdir("/") != 0)
		fail("chdir /");

	/* --- lesson 8: allocate the terminal HERE, inside the box --- */
	if (a->con->mode == CONSOLE_PTY) {
		/* The container needs its own devpts instance: the slave node
		   must live in this mount namespace for the path to mean
		   anything, and its ownership follows our id mappings. */
		if (mount("devpts", "/dev/pts", "devpts", 0,
		          "newinstance,ptmxmode=0666") != 0)
			fail("mount devpts");
		/* posix_openpt opens /dev/ptmx, but a fresh devpts instance
		   puts its multiplexor at /dev/pts/ptmx. Every real container
		   image carries this symlink for exactly this reason. */
		unlink("/dev/ptmx");
		if (symlink("pts/ptmx", "/dev/ptmx") != 0)
			fail("symlink /dev/ptmx");

		char slave_name[64];
		int master = pty_open(slave_name, sizeof slave_name);
		if (master < 0)
			fail("allocate pty");
		/* The master leaves through the console socket; whoever holds
		   the far end now holds this container's keyboard and screen. */
		if (console_send(a->con_fd, master) != 0)
			fail("send pty master");
		close(master);
		close(a->con_fd);

		if (setsid() < 0)
			fail("setsid");
		int slave = open(slave_name, O_RDWR);
		if (slave < 0)
			fail("open pty slave");
		if (ioctl(slave, TIOCSCTTY, 0) != 0)
			fail("TIOCSCTTY");
		dup2(slave, 0);
		dup2(slave, 1);
		dup2(slave, 2);
		if (slave > 2)
			close(slave);
	}

	/* --- lesson 7: park here. created, but not yet running. --- */
	if (read(a->start_fd, &b, 1) != 1)
		_exit(0); /* killed before start ever came */
	close(a->start_fd);

	execv(a->argv[0], a->argv);
	fail("execv");
	return 1;
}
```

Read that against the lessons and there are no surprises left in it:
the id-map handshake (lesson 5), the six-step pivot (lesson 4), the
devpts-and-SCM_RIGHTS handoff (lesson 8), the park (lesson 7). The one
line no lesson predicted is the `/proc` mount landing *before* the old
root is detached — inside a user namespace the kernel grants a fresh
procfs only when a fully-visible one is still present in the mount
namespace, so the order is forced.

The second half is the parent: it plans, clones, writes the maps the
child is waiting on, applies the cgroup, and implements the four
lifecycle operations on top of your state machine.

```c
static int write_file(const char *path, const char *text) {
	int fd = open(path, O_WRONLY);
	if (fd < 0)
		return -1;
	ssize_t n = write(fd, text, strlen(text));
	close(fd);
	return n < 0 ? -1 : 0;
}

/* The rootless map: exactly one id — ours — becomes container root. */
static int write_id_maps(int pid) {
	char path[64], line[64];
	struct idmap uid = {0, getuid(), 1};
	struct idmap gid = {0, getgid(), 1};

	snprintf(path, sizeof path, "/proc/%d/setgroups", pid);
	if (write_file(path, "deny") != 0)
		return -1;
	snprintf(path, sizeof path, "/proc/%d/uid_map", pid);
	snprintf(line, sizeof line, "%u %u %u", uid.container_id, uid.host_id,
	         uid.size);
	if (write_file(path, line) != 0)
		return -1;
	snprintf(path, sizeof path, "/proc/%d/gid_map", pid);
	snprintf(line, sizeof line, "%u %u %u", gid.container_id, gid.host_id,
	         gid.size);
	return write_file(path, line);
}

/* process.args[0]: scan to the array, take the first quoted string. */
static int first_arg(const char *doc, char *out, size_t cap) {
	const char *proc = after_key(doc, "process");
	if (proc == NULL)
		return -1;
	const char *args = after_key(proc, "args");
	if (args == NULL)
		return -1;
	const char *q = strchr(args, '"');
	if (q == NULL)
		return -1;
	size_t i = 0;
	for (q++; *q && *q != '"'; q++) {
		if (i + 1 >= cap)
			return -1;
		out[i++] = *q;
	}
	out[i] = '\0';
	return *q == '"' ? 0 : -1;
}

static void cgroup_path(const char *id, char *out, size_t cap) {
	const char *base = getenv("MINICT_CGROUP");
	snprintf(out, cap, "%s/ctr-%s", base ? base : "/sys/fs/cgroup", id);
}

int container_create(const char *id, const char *bundle,
                     const char *console_sock) {
	char cfg_path[512];
	snprintf(cfg_path, sizeof cfg_path, "%s/config.json", bundle);
	char *doc = read_file(cfg_path);
	if (doc == NULL) {
		fprintf(stderr, "minict: cannot read %s\n", cfg_path);
		return 1;
	}

	/* ---- the final challenge: decide everything, touch nothing ---- */
	struct plan plan;
	if (plan_container(bundle, doc, &plan) != 0) {
		fprintf(stderr, "minict: invalid config.json\n");
		return 1;
	}
	const char *proc = after_key(doc, "process");
	struct console_plan con;
	if (console_plan(proc ? proc : doc, console_sock, &con) != 0) {
		fprintf(stderr, "minict: terminal/console-socket mismatch\n");
		return 1;
	}
	char cmd[256];
	if (first_arg(doc, cmd, sizeof cmd) != 0) {
		fprintf(stderr, "minict: config has no process.args\n");
		return 1;
	}
	char *argv[] = {cmd, NULL};

	const char *dir = run_dir(id);
	if (mkdir(dir, 0755) != 0 && errno != EEXIST) {
		perror("minict: state dir");
		return 1;
	}

	struct child_args a = {.plan = &plan, .con = &con, .argv = argv};

	/* The exec fifo — runc's trick. Init blocks reading it; `start`
	   unblocks it by opening the write end. Opening O_RDWR never
	   blocks and never hits ENXIO, and the child inherits the fd, so
	   the fifo keeps a reader even after this process exits. */
	char fifo[512];
	snprintf(fifo, sizeof fifo, "%s/exec.fifo", dir);
	unlink(fifo);
	if (mkfifo(fifo, 0600) != 0) {
		perror("minict: mkfifo");
		return 1;
	}
	a.start_fd = open(fifo, O_RDWR);
	if (a.start_fd < 0 || pipe(a.mapped) != 0)
		return 1;

	pid_t pid = clone(child_main, child_stack + sizeof child_stack,
	                  plan.clone_flags | SIGCHLD, &a);
	if (pid < 0) {
		perror("minict: clone");
		return 1;
	}

	if (write_id_maps(pid) != 0) {
		fprintf(stderr, "minict: cannot write id maps\n");
		kill(pid, SIGKILL);
		return 1;
	}

	char cg[512];
	cgroup_path(id, cg, sizeof cg);
	if (cgroup_apply(cg, &plan, pid) != 0)
		fprintf(stderr,
		        "minict: note: cgroup limits not applied (%s) — see the "
		        "delegation note in the epilogue\n",
		        strerror(errno));

	close(a.mapped[0]);
	if (write(a.mapped[1], "x", 1) != 1)
		return 1;
	close(a.mapped[1]);

	int nspids[8];
	int levels = nspid_of(pid, nspids, 8);
	printf("minict: created %s — host pid %d", id, pid);
	if (levels >= 2)
		printf(", pid %d inside", nspids[levels - 1]);
	printf("\n");

	state_save(dir, lifecycle_next(ST_CREATING, EV_CREATE_DONE), pid, bundle);
	free(doc);
	return 0;
}

int container_start(const char *id) {
	const char *dir = run_dir(id);
	int st, pid;
	char bundle[256];
	if (state_load(dir, &st, &pid, bundle, sizeof bundle) != 0) {
		fprintf(stderr, "minict: no such container: %s\n", id);
		return 1;
	}
	int next = lifecycle_next(st, EV_START);
	if (next < 0) {
		fprintf(stderr, "minict: cannot start a %s container\n",
		        state_name(st));
		return 1;
	}

	char fifo[512];
	snprintf(fifo, sizeof fifo, "%s/exec.fifo", dir);
	int fd = open(fifo, O_WRONLY);
	if (fd < 0 || write(fd, "x", 1) != 1) {
		fprintf(stderr, "minict: cannot wake init: %s\n", strerror(errno));
		return 1;
	}
	close(fd);
	state_save(dir, next, pid, bundle);

	/* We are not init's parent (create was), so we cannot waitpid it.
	   Poll /proc until the pid is gone, then record the exit. */
	char proc[64];
	snprintf(proc, sizeof proc, "/proc/%d", pid);
	while (access(proc, F_OK) == 0)
		usleep(50000);
	state_save(dir, lifecycle_next(next, EV_PROC_EXIT), pid, bundle);
	return 0;
}

int container_kill(const char *id, int sig) {
	const char *dir = run_dir(id);
	int st, pid;
	char bundle[256];
	if (state_load(dir, &st, &pid, bundle, sizeof bundle) != 0) {
		fprintf(stderr, "minict: no such container: %s\n", id);
		return 1;
	}
	if (lifecycle_next(st, EV_KILL) < 0) {
		fprintf(stderr, "minict: cannot kill a %s container\n",
		        state_name(st));
		return 1;
	}
	if (kill(pid, sig) != 0) {
		perror("minict: kill");
		return 1;
	}
	/* The signal is sent; the state moves only when the process dies. */
	char proc[64];
	snprintf(proc, sizeof proc, "/proc/%d", pid);
	for (int i = 0; i < 40 && access(proc, F_OK) == 0; i++)
		usleep(50000);
	if (access(proc, F_OK) != 0)
		state_save(dir, lifecycle_next(st, EV_PROC_EXIT), pid, bundle);
	return 0;
}

int container_delete(const char *id) {
	const char *dir = run_dir(id);
	int st, pid;
	char bundle[256];
	if (state_load(dir, &st, &pid, bundle, sizeof bundle) != 0) {
		fprintf(stderr, "minict: no such container: %s\n", id);
		return 1;
	}
	if (lifecycle_next(st, EV_DELETE) < 0) {
		fprintf(stderr, "minict: cannot delete a %s container\n",
		        state_name(st));
		return 1;
	}
	char path[512], cg[512];
	cgroup_path(id, cg, sizeof cg);
	cgroup_remove(cg);
	snprintf(path, sizeof path, "%s/state.json", dir);
	unlink(path);
	snprintf(path, sizeof path, "%s/exec.fifo", dir);
	unlink(path);
	rmdir(dir);
	printf("minict: deleted %s\n", id);
	return 0;
}

int container_state(const char *id) {
	const char *dir = run_dir(id);
	char path[512];
	snprintf(path, sizeof path, "%s/state.json", dir);
	char *doc = read_file(path);
	if (doc == NULL) {
		fprintf(stderr, "minict: no such container: %s\n", id);
		return 1;
	}
	fputs(doc, stdout);
	free(doc);
	return 0;
}
```

Two things there are worth pausing on. `write_id_maps` uses your
`struct idmap` to build the literal `0 1000 1` line you read in
milestone 4 — the type from the challenge is the type the kernel wants.
And `container_start` polls `/proc/<pid>` instead of calling `waitpid`:
`start` is a *different process* from `create`, so it is not init's
parent and has no child to wait on. That is the price of the
create/start split, and every runtime pays some version of it.

## The CLI

`src/main.c` — argument dispatch, plus the two helpers the other files
use for reading a file and locating a container's state directory:

```c
#define _GNU_SOURCE
#include "minict.h"

#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <unistd.h>

/* Slurp a whole file into a NUL-terminated buffer the caller frees. */
char *read_file(const char *path) {
	FILE *f = fopen(path, "rb");
	if (f == NULL)
		return NULL;
	if (fseek(f, 0, SEEK_END) != 0) {
		fclose(f);
		return NULL;
	}
	long n = ftell(f);
	rewind(f);
	if (n < 0) {
		fclose(f);
		return NULL;
	}
	char *buf = malloc((size_t)n + 1);
	if (buf == NULL) {
		fclose(f);
		return NULL;
	}
	size_t got = fread(buf, 1, (size_t)n, f);
	fclose(f);
	buf[got] = '\0';
	return buf;
}

/* Where a container's state.json and exec.fifo live. Rootless, so
   under $XDG_RUNTIME_DIR rather than /run/minict. */
const char *run_dir(const char *id) {
	static char dir[768];
	const char *base = getenv("XDG_RUNTIME_DIR");
	if (base == NULL)
		base = "/tmp";
	char parent[512];
	snprintf(parent, sizeof parent, "%s/minict", base);
	mkdir(parent, 0700);
	snprintf(dir, sizeof dir, "%s/%s", parent, id);
	return dir;
}

static int usage(void) {
	fputs("usage:\n"
	      "  minict create <id> <bundle> [--console-socket PATH]\n"
	      "  minict start  <id>\n"
	      "  minict state  <id>\n"
	      "  minict kill   <id> [SIGNAL]\n"
	      "  minict delete <id>\n",
	      stderr);
	return 2;
}

int main(int argc, char **argv) {
	if (argc < 3)
		return usage();
	const char *cmd = argv[1], *id = argv[2];

	if (strcmp(cmd, "create") == 0) {
		if (argc < 4)
			return usage();
		const char *sock = NULL;
		for (int i = 4; i + 1 < argc; i++)
			if (strcmp(argv[i], "--console-socket") == 0)
				sock = argv[i + 1];
		return container_create(id, argv[3], sock);
	}
	if (strcmp(cmd, "start") == 0)
		return container_start(id);
	if (strcmp(cmd, "state") == 0)
		return container_state(id);
	if (strcmp(cmd, "delete") == 0)
		return container_delete(id);
	if (strcmp(cmd, "kill") == 0) {
		int sig = SIGTERM;
		if (argc > 3) {
			if (strcmp(argv[3], "KILL") == 0 || strcmp(argv[3], "9") == 0)
				sig = SIGKILL;
			else if (strcmp(argv[3], "INT") == 0)
				sig = SIGINT;
		}
		return container_kill(id, sig);
	}
	return usage();
}
```

Note where the state lives: `$XDG_RUNTIME_DIR/minict/<id>`, not
`/run/minict`. Rootless runtimes keep their bookkeeping somewhere the
user can actually write, and podman does exactly this.

## The four companions

Each goes at the bottom of the file that already holds your solution.

**`src/cgroup.c`** — the writes your formatters feed:

```c
/* ---- new: the four file operations that use them ---- */

static int write_str(const char *dir, const char *file, const char *val) {
	char path[512];
	if ((size_t)snprintf(path, sizeof path, "%s/%s", dir, file) >= sizeof path)
		return -1;
	FILE *f = fopen(path, "w");
	if (f == NULL)
		return -1;
	int ok = fputs(val, f) >= 0;
	if (fclose(f) != 0)
		ok = 0;
	return ok ? 0 : -1;
}

/* mkdir the leaf, write the three limits, move pid into it. Writing a
   "max" that was already max is harmless, so there is no special case. */
int cgroup_apply(const char *cg, const struct plan *p, int pid) {
	if (mkdir(cg, 0755) != 0 && access(cg, F_OK) != 0)
		return -1;
	if (write_str(cg, "cpu.max", p->cpu_max) != 0)
		return -1;
	if (write_str(cg, "memory.max", p->memory_max) != 0)
		return -1;
	if (write_str(cg, "pids.max", p->pids_max) != 0)
		return -1;
	char buf[32];
	snprintf(buf, sizeof buf, "%d", pid);
	return write_str(cg, "cgroup.procs", buf);
}

int cgroup_remove(const char *cg) {
	return rmdir(cg);
}
```

**`src/nspid.c`** — hand your parser the real file:

```c
/* ---- new: feed it the real file ---- */

int nspid_of(int pid, int *out, size_t cap) {
	char path[64], line[256];
	snprintf(path, sizeof path, "/proc/%d/status", pid);
	FILE *f = fopen(path, "r");
	if (f == NULL)
		return -1;
	int n = -1;
	while (fgets(line, sizeof line, f) != NULL) {
		if (strncmp(line, "NSpid:", 6) == 0) {
			n = nspid_parse(line, out, cap);
			break;
		}
	}
	fclose(f);
	return n;
}
```

**`src/state.c`** — the `state.json` the spec asks a runtime to keep.
It is read back with *your own* JSON scanner, which is a quietly
satisfying moment: the parser you wrote in lesson 2 is now parsing this
runtime's own output.

```c
/* ---- new: the state.json the OCI spec asks a runtime to keep ---- */

static const char *const names[] = {"creating", "created", "running",
                                    "stopped", "deleted"};

const char *state_name(int st) {
	if (st < 0 || st > ST_GONE)
		return "unknown";
	return names[st];
}

int state_save(const char *dir, int st, int pid, const char *bundle) {
	char path[512];
	snprintf(path, sizeof path, "%s/state.json", dir);
	FILE *f = fopen(path, "w");
	if (f == NULL)
		return -1;
	fprintf(f,
	        "{\n  \"ociVersion\": \"1.2.0\",\n"
	        "  \"status\": \"%s\",\n"
	        "  \"pid\": %d,\n"
	        "  \"bundle\": \"%s\"\n}\n",
	        state_name(st), pid, bundle);
	return fclose(f) == 0 ? 0 : -1;
}

int state_load(const char *dir, int *st, int *pid, char *bundle, size_t cap) {
	char path[512];
	snprintf(path, sizeof path, "%s/state.json", dir);
	char *doc = read_file(path);
	if (doc == NULL)
		return -1;

	char status[32];
	int rc = -1;
	if (json_str(doc, "status", status, sizeof status) == 0 &&
	    json_str(doc, "bundle", bundle, cap) == 0) {
		*st = -1;
		for (int i = 0; i <= ST_GONE; i++)
			if (strcmp(status, names[i]) == 0)
				*st = i;
		const char *p = after_key(doc, "pid");
		if (*st >= 0 && p != NULL && sscanf(p, " %d", pid) == 1)
			rc = 0;
	}
	free(doc);
	return rc;
}
```

**`src/console.c`** — the PTY and the fd handoff:

```c
/* ---- new: the syscalls the plan authorizes ---- */

/* Allocate a PTY pair from the CONTAINER's own devpts instance and
   write back the slave's path. Called after the pivot, so /dev/pts/N
   names a node inside the container and its ownership follows the id
   mappings. */
int pty_open(char *slave_name, size_t cap) {
	int m = posix_openpt(O_RDWR | O_NOCTTY);
	if (m < 0)
		return -1;
	if (grantpt(m) != 0 || unlockpt(m) != 0) {
		close(m);
		return -1;
	}
	if (ptsname_r(m, slave_name, cap) != 0) {
		close(m);
		return -1;
	}
	return m;
}

/* Connect to the caller's console socket. This must happen BEFORE
   pivot_root: the path is a host path, and after the pivot the
   container cannot name it. The connected fd survives the pivot —
   an open file description does not care what / points at. */
int console_connect(const char *sock_path) {
	int s = socket(AF_UNIX, SOCK_STREAM, 0);
	if (s < 0)
		return -1;
	struct sockaddr_un addr = {.sun_family = AF_UNIX};
	if (strlen(sock_path) + 1 > sizeof addr.sun_path) {
		close(s);
		return -1;
	}
	strcpy(addr.sun_path, sock_path);
	if (connect(s, (struct sockaddr *)&addr, sizeof addr) != 0) {
		close(s);
		return -1;
	}
	return s;
}

/* Send the master fd down an already-connected console socket as
   SCM_RIGHTS. The byte in iov is not payload — a control message needs
   at least one byte of ordinary data to ride along with. */
int console_send(int s, int master_fd) {
	char byte = 'C';
	struct iovec iov = {.iov_base = &byte, .iov_len = 1};
	union {
		char buf[CMSG_SPACE(sizeof(int))];
		struct cmsghdr align;
	} u = {0};
	struct msghdr msg = {
		.msg_iov = &iov,
		.msg_iovlen = 1,
		.msg_control = u.buf,
		.msg_controllen = sizeof u.buf,
	};
	struct cmsghdr *cm = CMSG_FIRSTHDR(&msg);
	cm->cmsg_level = SOL_SOCKET;
	cm->cmsg_type = SCM_RIGHTS;
	cm->cmsg_len = CMSG_LEN(sizeof(int));
	memcpy(CMSG_DATA(cm), &master_fd, sizeof(int));

	return sendmsg(s, &msg, 0) == 1 ? 0 : -1;
}
```

`console_send` is worth reading twice. The `SCM_RIGHTS` control message
is the payload; the single byte in `iov` exists only because a control
message needs at least one byte of ordinary data to travel with. What
arrives on the other side is not the number 5 — it is a new entry in
the receiver's fd table pointing at the same open file.

## The other side of the socket

`src/attach.c` is a separate program: the caller, standing in for
containerd or `docker attach`. It listens, receives the master fd, puts
your terminal in raw mode (so the container's line discipline is the
only one interpreting keys), and pumps bytes both ways until the far
end closes.

```c
/* minict-attach — the CALLER's side of the console socket.
   Listens on a unix socket, receives the PTY master fd that `minict
   create` sends, then relays your terminal to it until the container
   exits. This is, in miniature, what `docker attach` does. */
#define _GNU_SOURCE

#include <errno.h>
#include <poll.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <termios.h>
#include <unistd.h>

static struct termios saved;
static int raw_active;

static void restore(void) {
	if (raw_active)
		tcsetattr(STDIN_FILENO, TCSANOW, &saved);
}

/* Put OUR terminal in raw mode: the container's line discipline is the
   one that should echo and interpret Ctrl-C, not ours. */
static void go_raw(void) {
	if (tcgetattr(STDIN_FILENO, &saved) != 0)
		return;
	struct termios raw = saved;
	cfmakeraw(&raw);
	if (tcsetattr(STDIN_FILENO, TCSANOW, &raw) == 0) {
		raw_active = 1;
		atexit(restore);
	}
}

static int recv_fd(int conn) {
	char byte;
	struct iovec iov = {.iov_base = &byte, .iov_len = 1};
	union {
		char buf[CMSG_SPACE(sizeof(int))];
		struct cmsghdr align;
	} u = {0};
	struct msghdr msg = {
		.msg_iov = &iov,
		.msg_iovlen = 1,
		.msg_control = u.buf,
		.msg_controllen = sizeof u.buf,
	};
	if (recvmsg(conn, &msg, 0) < 1)
		return -1;
	for (struct cmsghdr *c = CMSG_FIRSTHDR(&msg); c; c = CMSG_NXTHDR(&msg, c)) {
		if (c->cmsg_level == SOL_SOCKET && c->cmsg_type == SCM_RIGHTS) {
			int fd;
			memcpy(&fd, CMSG_DATA(c), sizeof fd);
			return fd;
		}
	}
	return -1;
}

int main(int argc, char **argv) {
	if (argc != 2) {
		fprintf(stderr, "usage: minict-attach <console.sock>\n");
		return 2;
	}
	const char *path = argv[1];
	unlink(path);

	int srv = socket(AF_UNIX, SOCK_STREAM, 0);
	struct sockaddr_un addr = {.sun_family = AF_UNIX};
	if (strlen(path) + 1 > sizeof addr.sun_path) {
		fprintf(stderr, "minict-attach: socket path too long\n");
		return 1;
	}
	strcpy(addr.sun_path, path);
	if (bind(srv, (struct sockaddr *)&addr, sizeof addr) != 0 ||
	    listen(srv, 1) != 0) {
		perror("minict-attach: listen");
		return 1;
	}
	fprintf(stderr, "minict-attach: waiting on %s\n", path);

	int conn = accept(srv, NULL, NULL);
	if (conn < 0) {
		perror("minict-attach: accept");
		return 1;
	}
	int master = recv_fd(conn);
	close(conn);
	close(srv);
	unlink(path);
	if (master < 0) {
		fprintf(stderr, "minict-attach: no fd in that message\n");
		return 1;
	}
	fprintf(stderr, "minict-attach: got the pty master (fd %d)\n", master);

	go_raw();
	struct pollfd fds[2] = {
		{.fd = STDIN_FILENO, .events = POLLIN},
		{.fd = master, .events = POLLIN},
	};
	for (;;) {
		if (poll(fds, 2, -1) < 0 && errno != EINTR)
			break;
		char buf[4096];
		if (fds[0].revents & POLLIN) {
			ssize_t n = read(STDIN_FILENO, buf, sizeof buf);
			if (n <= 0 || write(master, buf, (size_t)n) != n)
				break;
		}
		if (fds[1].revents & POLLIN) {
			ssize_t n = read(master, buf, sizeof buf);
			if (n <= 0)
				break; /* container closed the far end */
			if (write(STDOUT_FILENO, buf, (size_t)n) != n)
				break;
		}
		if (fds[1].revents & (POLLHUP | POLLERR))
			break;
	}
	restore();
	fprintf(stderr, "\r\nminict-attach: console closed\n");
	return 0;
}
```

## Building it

```make
CFLAGS = -std=gnu11 -Wall -Wextra -O1 -g
OBJS = src/oci.o src/cgroup.o src/idmap.o src/nspid.o src/state.o \
       src/console.o src/container.o src/main.o

all: minict minict-attach

minict: $(OBJS)
	$(CC) $(CFLAGS) -o $@ $(OBJS)

minict-attach: src/attach.o
	$(CC) $(CFLAGS) -o $@ src/attach.o

$(OBJS) src/attach.o: src/minict.h

clean:
	rm -f minict minict-attach src/*.o
```

```
$ make
$ ls
Makefile  bundle  minict  minict-attach  src
```

## The whole thing, running

Batch mode first — no terminal, just a command and its output:

```
$ minict create demo ./bundle
minict: created demo — host pid 40219, pid 1 inside
$ minict state demo | grep status
  "status": "created",
$ minict start demo
duck
$ minict delete demo
minict: deleted demo
```

(That `duck` is `/bin/hostname` running inside, printing the hostname
your `config.json` asked for and your `sethostname` applied.)

Now interactive. Terminal one:

```
$ minict-attach /tmp/console.sock
minict-attach: waiting on /tmp/console.sock
```

Terminal two — set `"terminal": true` in `bundle/config.json` and
`"args": ["/bin/sh"]`:

```
$ minict create demo ./bundle --console-socket /tmp/console.sock
minict: created demo — host pid 41533, pid 1 inside
$ minict start demo
```

Terminal one comes alive:

```
minict-attach: got the pty master (fd 5)
/ # hostname
duck
/ # id
uid=0(root) gid=0(root) groups=65534(nobody),65534(nobody),65534(nobody),65534(nobody),0(root)
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    4 root      0:00 ps
/ # ls /
bin    etc    lib    proc   sys    usr
dev    home   lib64  root   tmp    var
/ # tty
/dev/pts/0
/ # exit
minict-attach: console closed
```

Six answers, six lessons. The hostname came from your UTS namespace.
The uid came from your user namespace — and on the host that same
process belongs to your ordinary account. Those repeated `nobody`
entries are the overflow id doing its job: your supplementary groups
have no mapping in this namespace, so the kernel reports each as 65534
rather than leaking a host group number. `ps` shows two processes
because `/proc` is a fresh procfs in a fresh pid namespace. `ls /` is
the unpacked image, because the old root was pivoted away and
unmounted. `/dev/pts/0` is a terminal in a devpts instance that exists
nowhere else. And it all happened without `sudo`.

With cgroups delegated, the limits are real too:

```
$ systemd-run --user --scope -p Delegate=yes bash
$ export MINICT_CGROUP=/sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)
$ mkdir "$MINICT_CGROUP/sup" && echo $$ > "$MINICT_CGROUP/sup/cgroup.procs"
$ echo "+pids +memory +cpu" > "$MINICT_CGROUP/cgroup.subtree_control"
$ minict create cg1 ./bundle
minict: created cg1 — host pid 41746, pid 1 inside
$ cat "$MINICT_CGROUP/ctr-cg1/pids.max"
64
$ cat "$MINICT_CGROUP/ctr-cg1/memory.max"
268435456
$ cat "$MINICT_CGROUP/ctr-cg1/cpu.max"
50000 100000
```

Those three values travelled from `config.json`, through
`plan_container`, through your three formatters, into kernel state.
That is the entire thesis of this course in one command.

## When it doesn't work

- **`clone: Operation not permitted`** — unprivileged user namespaces
  are off. Check `sysctl kernel.unprivileged_userns_clone` (Debian) or
  `user.max_user_namespaces` (any distro); on Arch and Fedora they are
  on by default. Confirm with `unshare -U -r id`.
- **`mount /proc: Operation not permitted`** — you dropped `"user"`
  from the namespace list but kept `"mount"`. Without a user namespace
  an ordinary user may not mount anything; add it back.
- **`cgroup limits not applied (Permission denied)`** — you are not in
  a delegated scope. Harmless: every other part still works. Use the
  `systemd-run` recipe above to get limits.
- **`cgroup.subtree_control: Device or resource busy`** — the
  no-internal-process rule, catching you exactly as advertised: some
  process is still sitting in the cgroup you're trying to enable
  controllers on. Usually it's a subshell your own command line spawned
  a moment earlier. Move your shell into the `sup` leaf *first*, then
  enable controllers; if it still complains, `cat cgroup.procs` in that
  directory and see who's loitering.
- **`execv: No such file or directory`** — the path in `process.args`
  doesn't exist *inside the rootfs*, or the binary is dynamically
  linked against libraries the image lacks. Busybox is static; that's
  why this course uses it.
- **The container starts itself before you call `start`** — the exec
  fifo was opened with the wrong flags. It must be `O_RDWR` in
  `create`; anything else either blocks or returns EOF immediately.
- **`minict create ... | grep something` never returns.** Nothing is
  broken, and the container was created fine. The parked init inherited
  stdout, so the pipe has a writer that will not close until the
  container exits — and `grep` waits for EOF. This is the same reason
  `docker run -d` detaches its stdio rather than inheriting yours.
  Redirect `create`'s output to a file, or don't pipe it.

## What a real runtime does that yours doesn't

Honest accounting, so you know the size of the remaining gap:

- **Mounts.** The spec's `mounts` array (`/proc`, `/dev`, `/sys`,
  tmpfs, volumes, bind mounts with options) — yours hardcodes `/proc`
  and `devpts`. This is where `secure_join` would earn its keep for
  real, on every destination.
- **Capabilities, seccomp, LSMs, rlimits, `no_new_privs`.** The whole
  second layer of confinement. runc's seccomp profile alone blocks
  ~40 syscalls.
- **Networking.** Yours shares your host's network namespace. A real
  stack creates a netns and hands it to a CNI plugin for veth pairs,
  bridges, and NAT.
- **`exec` into a running container** (`setns` on each namespace),
  `ps`, `events`, `checkpoint`, and the `paused` state via the freezer.
- **Hooks** — `createRuntime`, `startContainer`, `poststop`, etc.
- **A real JSON parser**, so key names may nest and repeat, and so a
  hostile document can't confuse a positional scan.

None of those change the shape of what you built. They are more of the
same job: read the spec, decide before you act, and refuse anything you
cannot honor. You have written that runtime's spine — the next commit
is just more spec.
