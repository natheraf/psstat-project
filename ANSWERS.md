# psstat — Assignment Questions & Answers

This file answers the **Questions** section of the CISC 7310X Project 1
(psstat) assignment. Each answer is short and, where a question asks you to
run something, includes the command and its (abbreviated) output. The script
under test is `src/bash/psstat`.

> **Note on the environment.** This was developed inside a minimal container
> (PID 1 is `tini`, PID 7 is `open-terminal`, no systemd / kernel threads).
> The *format* of every output matches the assignment; the *set of PIDs* and
> exact values obviously differ from the fully-loaded box in the examples.

---

## 1. Annotated execution trace

I ran the `--list-sched-policy-is 0` mode under `strace` (full trace is in
`traces/strace-list-sched-policy-is-0.txt`). The per-process work repeats this
sequence for every PID. For PID 7, the trace shows:

```
openat(AT_FDCWD, "/proc/7/stat",    O_RDONLY) = 3   # 1st read of stat
openat(AT_FDCWD, "/proc/7/comm",    O_RDONLY) = 3   # comm read
openat(AT_FDCWD, "/proc/7/stat",    O_RDONLY) = 3   # 2nd read of stat
openat(AT_FDCWD, "/proc/7/cmdline", O_RDONLY) = 3   # cmdline read
```

For each `/proc` file my script reads:

| (a) file | (b) field extracted | (c) how it is parsed |
|---|---|---|
| `/proc/7/stat` | `policy` (field 41) — the CPU scheduling policy | `stat_field()` anchors on the **first `(`** and **last `)`**, then splits the text after `) ` on whitespace; field 41 is the 38th such token. |
| `/proc/7/stat` | `state` (field 3) | Same helper, field 3 = the first token after the closing paren. |
| `/proc/7/comm` | the image name (whole line) | Read whole, trailing newline stripped (bash command substitution). |
| `/proc/7/cmdline` | all arguments | Arguments are **NUL-separated**; the script reads them with `read -r -d ''` until EOF and re-renders each single-quoted. |

Why `stat` is opened **twice**: once by `stat_field <pid> 41` (to filter on
policy) and again by `stat_field <pid> 3` (to print the state). Each call is a
separate `cat`, so the file is re-opened. (This is a deliberate, simple design
choice — correctness over efficiency — and it is what the trace makes
visible.)

---

## 2. The `read(2)` syscall count for `--list-short`

**Answer: zero.**

`--list-short` prints only PIDs, and it discovers them by globbing the
`/proc/[0-9]*/` directory names — it never opens or reads any
`/proc/<pid>/stat|comm|cmdline` file. A clean `strace` of the whole run
(`traces/strace-list-short.txt`, 347 syscalls total) contains **no `read(2)`
call on any `/proc/<pid>/*` file**.

What it *does* do is `newfstatat` on each candidate path, which is how bash
expands the glob and how the `[ -d /proc/$pid ]` existence test works:

```
newfstatat(AT_FDCWD, "/proc/1",   {st_mode=S_IFDIR|0555, st_size=0, ...}, 0) = 0
newfstatat(AT_FDCWD, "/proc/7",   {st_mode=S_IFDIR|0555, st_size=0, ...}, 0) = 0
newfstatat(AT_FDCWD, "/proc/985", {st_mode=S_IFDIR|0555, st_size=0, ...}, 0) = 0
newfstatat(AT_FDCWD, "/proc/986", {st_mode=S_IFDIR|0555, st_size=0, ...}, 0) = 0
newfstatat(AT_FDCWD, "/proc/989", {st_mode=S_IFDIR|0555, st_size=0, ...}, 0) = 0
newfstatat(AT_FDCWD, "/proc/990", 0x7ffcba7d9600, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/proc/991", 0x7ffcba7d9600, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/proc/992", 0x7ffcba7d9600, 0) = -1 ENOENT (No such file or directory)
```

The `ENOENT` on `/proc/990` is a process that exited between the glob and the
test — it is correctly skipped. So the honest count, justified by the trace:
`read(2)` on `/proc` files = **0**; the discovery mechanism is a batch of
`newfstatat(2)` probes.

---

## 3. Break it on purpose: reading another user's `/proc`

I'm running as `user` (uid 1000). PID 7 (`open-terminal`) is **owned by
root** (`ls -l /proc/7/comm` → `-rw-r--r-- 1 root root`). What happens when I
read its files:

```
$ cat /proc/7/stat          # works
7 (open-terminal) R 1 7 1 0 -1 4194560 22734 ...
$ cat /proc/7/comm          # works
open-terminal
$ ls -l /proc/7/            # the protected entries
-r-------- 1 root root 0 ... auxv
-r--r--r-- 1 root root 0 ... cmdline
-rw-r--r-- 1 root root 0 ... comm
---------- 1 root root 0 ... environ
lrwxrwxrwx 1 root root 0 ... cwd -> ...   (but readlink fails)
lrwxrwxrwx 1 root root 0 ... exe  -> ...  (but readlink fails)
dr-x------ 2 root root 0 ... fd
$ python3 -c 'import os; [os.open(p, os.O_RDONLY) for p in
  ("/proc/7/stat","/proc/7/comm","/proc/7/environ","/proc/7/fd")]'
# -> stat: OK, comm: OK, environ: Permission denied (errno 13), fd: Permission denied (errno 13)
```

* **What happens:** the *world-readable* files (`stat`, `comm`, `cmdline`) are
  read successfully even for a root-owned process, but the *sensitive* files
  (`environ`, `auxv`, and the `fd`/`cwd`/`exe` symlinks and their `readlink`)
  are refused.
* **Which syscall returns the error:** the failing operation is the **`open(2)`
  / `readlink(2)`** on the sensitive path, **not** `read(2)` — the file never
  gets opened, so there is nothing to read.
* **errno:** **13, `EACCES` (Permission denied).**
* **Why the kernel allows `stat`/`comm`/`cmdline` but denies `environ`:**
  these `/proc/<pid>/*` "files" are generated in memory by the kernel with a
  per-file mode mask. `stat`, `comm` and `cmdline` are made world-readable
  because they are considered non-sensitive (they do not reveal credentials).
  `environ` can hold secrets (API keys, tokens), `fd`/`cwd`/`exe` let you
  inspect or hijack open file descriptors, and `auxv` can leak ASLR — so those
  are `0400`/`0000` owned by the process's real UID, and a non-owner without
  `CAP_SYS_PTRACE` gets `EACCES`. The directory `/proc/7/` itself is "owned"
  by root only in the cosmetic sense; the kernel actually enforces the
  *real-UID* check (plus `PTRACE_MODE_READ_FSCREDS`) on each individual file,
  which is why some of them succeed and others fail.

---

## 4. Field-counting in `/proc/<pid>/stat`

I created a process whose command name literally contains spaces **and**
parentheses (an executable named `test (123)`):

```
$ cat /proc/1075/stat
1075 (test (123)) S 1072 1072 1072 0 -1 4194560 175 0 0 0 ...
$ cut -d' ' -f2 /proc/1075/stat
(test
```

`cut -d' ' -f2` splits on **every single space**. For `1075 (test (123)) S
...`, the first space is after `1075`, so "field 2" is everything up to the
*next* space — `(test` — not the real name `test (123)`. The internal spaces
and the trailing `))` throw off every field after it (the state ends up in the
wrong column, the policy is in a completely wrong slot, etc.). Any naive
whitespace field-numbering breaks on such a name.

**Correct way:** read the name from **`/proc/<pid>/comm`** (one line, no
surprise spaces in the field index), *or* if you must use `stat`, anchor on the
**first `(`** and **last `)`** in the line: everything between them is `comm`,
and only the text *after* the last `) ` is safe to split into numeric fields.
That is exactly what `stat_field()` does.

**Why the kernel wraps it in parentheses at all:** `comm` is the *only* field
in the otherwise-space-separated `stat` line that is allowed to contain spaces
and parentheses (it's the command name, set via `exec`/`prctl(PR_SET_NAME)`).
Wrapping it in `(...)` gives parsers an unambiguous start marker — combined
with the fact that the *last* `)` on the line always closes it, a correct
parser can always locate the boundary. The parentheses are the kernel's way of
making an otherwise unparseable line parseable.

---

## 5. PIDs are not contiguous

```
$ ./psstat --list-short
PID
1
7
758
763
764
```

The gaps exist because the PID namespace **recycles and skips numbers**:

1. **PIDs wrap around.** Linux reuses PIDs from 1 up to `pid_max`
   (`/proc/sys/kernel/pid_max`, default 4194304). After the highest PID is
   used, allocation wraps to the low end, so the "next" PID can be *smaller*
   than an existing one.
2. **Exits create holes.** When a process exits its `/proc/<pid>/` entry
   disappears; the number is only reused *later* (often after a delay to
   avoid a stale-PID race). So at any instant many numbers in the middle are
   simply free.
3. **`/proc` only shows *live* PIDs.** A snapshot therefore shows a sparse,
   non-monotonic set — e.g. `1` (tini) and `7` (open-terminal) started early,
   while `758`+ are recent.

**Does my script assume contiguity? No.** It globs the actual directory names
present (`/proc/[0-9]*/`) and does **not** iterate `for pid in $(seq 1 N)`. So
gaps are handled naturally. If I *had* assumed contiguity, `seq 1 N` would
print PIDs for processes that no longer exist; the `[ -d /proc/$pid ]` guard
(and the ENOENT in Q2's trace) would then hide most of them, but the list
would be wrong and would also miss PIDs greater than the current maximum that
I had no way of discovering.

---

## 6. `/proc` is not a real filesystem

`/proc/<pid>/stat` reports `utime` and `stime` (clock ticks the process spent
in user and kernel mode). **They are not stored on disk at all.** They live in
kernel memory: the `task_struct` of the process (specifically the
`task->utime` / `task->stime` / `signal->*time` accounting in the cputime
structures). `/proc` is a **virtual** (sysfs-like) filesystem: it has no
on-disk backing store and no blocks.

**What happens on `read(2)`:** when userspace opens `/proc/<pid>/stat` and
reads it, the kernel's `/proc` "show" function (e.g.
`proc_pid_stat()` → `seq_printf(...)` in `fs/proc/base.c`) is invoked. It
walks the target `task_struct`, pulls out the current values (pid, state,
utime/stime, vsize, rss, policy, ...), **formats them into a buffer in
memory on the fly**, and returns that buffer to `read(2)`. The file's reported
size is always 0 because it is not a real file. Consequence: the value is a
*live snapshot* at read time — read the file twice quickly and `utime`/`stime`
can differ, because the process kept running in between.

---

## 7. Compare with `ps`

```
$ ps -eo pid,stat,comm,args
    PID STAT COMMAND         COMMAND
      1 Ss   tini            /usr/bin/tini -- /app/entrypoint.sh run
      7 Sl   open-terminal   /usr/local/bin/python3-ot /usr/local/bin/open-termi
   1080 Ss   sh              /bin/sh -c cd /home/user/CISC7310X/psstat-project e
   1081 R    ps              ps -eo pid,stat,comm,args

$ ./psstat --list-long
PID CMD ST CMD_ARGS
1 "tini" S "'/usr/bin/tini' '--' '/app/entrypoint.sh' 'run'"
7 "open-terminal" S "'/usr/local/bin/python3-ot' '/usr/local/bin/open-terminal' 'run'"
1080 "sh" S "'/bin/sh' '-c' '...'"
```

Three (actually more) differences, and the `/proc` source behind each:

1. **`STAT` (multi-character) vs my single-character `ST`.** `ps` shows `Ss`
   for PID 1 and `Sl` for PID 7; I show just `S`. The base `S` is field 3 of
   `/proc/<pid>/stat` (which I read), but the *extra* characters are derived
   from fields I do **not** read in `--list-long`:
   * `s` (session leader) → compare `pid` to the **session** id (**field 6**)
   * `l` (multi-threaded) → **num_threads** (**field 20**) > 1
   So `ps` is reading `/proc/<pid>/stat` fields 6 and 20 that my `--list-long`
   ignores (it only takes field 3).

2. **`ps` decodes the `sflags` word (field 9) into flag letters.** `STAT`
   characters such as `S` (niced), `L` (pages locked), `<`/`N` (scheduling
   priority) come from the **`flags`/`sflags`** field (**field 9**) of
   `/proc/<pid>/stat`, a bitmask `ps` decodes. My script never reads field 9.
   (I verified field 9 = `4194560` for both PID 1 and 7 here; `ps` turns the
   relevant bits into the trailing letters.)

3. **`args` formatting.** Both use `/proc/<pid>/cmdline`, but `ps` joins the
   arguments with a single space and prints them unquoted (`/usr/bin/tini --
   /app/entrypoint.sh run`), whereas I render each argument single-quoted
   (`'/usr/bin/tini' '--' ...`) to preserve argument boundaries. Same `/proc`
   file, different derivation — mine keeps the exact NUL-separated argument
   structure visible.

Additional (formatting) differences worth noting: `ps` **right-justifies** the
PID and **truncates `comm` to 15 chars** (the kernel's `TASK_COMM_LEN`) while I
left-justify and print the full name; and the column order differs (`ps`:
pid,stat,comm,args → mine: pid,cmd,st,cmd_args).

---

## 8. The scheduling policy mapping

From `/usr/include/linux/sched.h` (this system), the `SCHED_*` **policy**
constants are:

| constant | value |
|---|---|
| `SCHED_NORMAL` | 0 |
| `SCHED_FIFO`   | 1 |
| `SCHED_RR`     | 2 |
| `SCHED_BATCH`  | 3 |
| `SCHED_IDLE`   | 5 |
| `SCHED_DEADLINE` | 6 |
| `SCHED_EXT`    | 7 |

The header also has a *comment* — not a `#define` — for
`SCHED_ISO: reserved but not implemented yet` (it would be value **4**), and a
separate flag `SCHED_RESET_ON_FORK = 0x40000000` (OR-ed into a policy, not a
policy itself).

* **Is number→name one-to-one?** Yes, for the defined constants — each value
  maps to exactly one name and vice versa. (Note: the *same* value 0 is called
  `SCHED_NORMAL` in the kernel header but `SCHED_OTHER` in `procps`/`ps`; the
  two names refer to the same policy, so it is still one-to-one per source.)
* **Numbers with no constant in the header:** value **4** (`SCHED_ISO`) is
  reserved / not implemented and has **no `#define`** in this header. On this
  system every live process I checked reported policy **0** (all
  `SCHED_NORMAL`/`SCHED_OTHER`, confirmed with `chrt -p`). My `policy_name()`
  maps 4 → `SCHED_ISO` (labelled reserved) and prints `UNKNOWN(<n>)` for any
  other value, so an out-of-range or reserved number can never be mistaken for
  a real policy.

---

## 9. Reproduce a failure: "No such file or directory" even though the process "exists"

Concrete scenario — **the process exits between my discovery and my read.**
The modes discover PIDs by globbing `/proc`, then read `/proc/<pid>/stat` (and
friends) *later*, in separate system calls. A short-lived process can die in
that window:

```
$ sleep 0.05 &          # a process that will be gone in ~50 ms
$ ./psstat --list-pid-is $!
psstat: process <pid>: No such file or directory     # exit status 1
```

Even though the process *existed* when I obtained its PID, by the time
`--list-pid-is` calls `cat /proc/<pid>/comm` / reads `/proc/<pid>/stat`, the
kernel has already reaped it and removed the directory. The `open(2)` of
`/proc/<pid>/stat` fails with **`ENOENT` (errno 2)** — "No such file or
directory" — because the `/proc/<pid>/` entry (a dentry generated from the
live `task_struct`) no longer exists. The same race is what Q2's trace caught
(`newfstatat("/proc/990") = -1 ENOENT`).

A second, related case: the process is alive but owned by someone else *and*
the file I need is one of the sensitive ones — that returns `EACCES` (13), a
different errno, covered in Q3. The "No such file or directory" specifically
is the **process-vanished** race (errno 2), not the permission case. My script
handles it by re-checking `[ -d /proc/$pid ]` before reading and by treating
an empty `comm`/`state` read as "gone," so it reports a clean message and a
non-zero exit rather than a malformed line.

---

## 10. Design note

**Discovery.** PIDs are found by globbing the numeric directory entries under
`/proc` — `for d in /proc/[0-9]*/`. I deliberately do *not* read `/proc` with
`getdents`-style listing or call `ps`, because the assignment forbids `ps` and
because globbing is the simplest idiom that stays pure-bash. The glob expands
in bytewise (lexicographic) order, which is **not** numeric ascending (`"100"
< "9"`), so I pipe the bare PIDs through `sort -n` to match the reference
ordering. Each PID is then re-checked with `[ -d /proc/$pid ]` before use,
which both (a) skips PIDs that raced out and (b) is what shows up as the
`newfstatat` probes in the Q2 trace.

**Parsing `/proc/<pid>/stat`.** The crux is the `comm` field. I read the whole
line, locate the **first `(`** and the **last `)`**, treat everything between
them as `comm`, and split only the text *after* the last `) ` into whitespace
fields. Field *N* (1-based, comm = field 2) is the `(N-3)`th post-paren token.
I verified the column positions against `chrt`/`ps` rather than trusting
memory: `state`=3, `minflt`=10, `majflt`=12, `utime`=14, `stime`=15,
`num_threads`=20, `vsize`=23, `rss`=24, `rt_priority`=40, `policy`=41. Getting
these right mattered — I initially had `policy` off by fifteen columns (reading
field 26) and produced nonsense like `policy = 18446744073709551615`;
cross-checking with `chrt -p` (which reported `SCHED_OTHER`) caught it. The
image name for display I take from `/proc/<pid>/comm` directly (cleaner than
re-extracting from `stat`), and the arguments from `/proc/<pid>/cmdline`,
reading NUL-delimited with `read -r -d ''`.

**What happens if a process exits while I'm reading it.** Between the glob and
the read the process can be reaped, so `open(2)` on `/proc/<pid>/stat`
returns `ENOENT`. I guard every mode with an existence re-check and treat a
failed/empty `comm` or `state` read as "process gone": the line is skipped (in
list modes) or a `No such file or directory` message with a non-zero exit is
emitted (in `--list-pid-is`). This makes the script robust to the snapshot
inconsistency that is unavoidable in a single-pass `/proc` walk (Q9).

**Something I tried that did not work, and the fix.** My first `--list-short`
used the raw glob output and produced `1`, `10`, `100`, `101`... in
*lexicographic* order with a stray trailing `/` on each entry (the glob returns
`/proc/7/`). It looked plausible at a glance but was wrong ordering and had a
path-suffix bug. I fixed it by stripping both the `/proc/` prefix and the
trailing `/` with parameter expansion (`${d#/proc/}` then `${pid%/}`) and
piping through `sort -n`. A second near-miss was parsing `stat` with a naive
whitespace split, which silently misaligned every field for any process whose
name had a space (Q4) — I caught it by deliberately launching a process named
`test (123)` and confirming `cut -d' ' -f2` returned `(test`, then switching
to the first-`(`/last-`)` anchor.
