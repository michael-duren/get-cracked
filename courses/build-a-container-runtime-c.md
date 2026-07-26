---
course: build-a-container-runtime
title: Build an OCI Container Runtime in C
language: c
description: >
  Containers are not virtual machines — they're ordinary Linux processes
  wearing a disguise the kernel provides. Build the brain of an OCI runtime
  in C: parse config.json with your own scanner, turn namespace lists into
  clone flags, jail paths so a hostile config can't escape the rootfs, map
  container root to an unprivileged user, translate resource limits into
  cgroup v2 writes, and drive the create/start/kill/delete lifecycle — the
  same decisions runc makes before every container on earth starts.
duration_hours: 6
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

## What you'll build, and how the grading works

A real runtime needs root (or user namespaces) to do its work, and the
grader that checks your code runs in a sandbox that has neither. So this
course splits honestly: **lessons** walk through the privileged syscall
choreography you'd run on your own machine; **challenges** grade the
decision logic around those syscalls — the parsing, validation, mapping,
and formatting that *is* most of a real runtime's code. Every piece you
write here has a direct counterpart inside runc.

First piece: the OCI config lists namespaces by name — `"pid"`,
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
networking (that's when CNI plugins run), run hooks. So `create` does
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
that appears before the closing `]`. (This position game is exactly the
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

	return failed;
}
```
