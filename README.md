# psstat — process information from `/proc`

`psstat` is a Bash shell script that lists process information using **only**
the `/proc` file system. It does not call `ps` (or similar wrappers) to gather
process data.

## Running

```sh
./src/bash/psstat --help
./src/bash/psstat --list-short
./src/bash/psstat --list-long
./src/bash/psstat --list-name-has bash
./src/bash/psstat --list-pid-is <pid>
./src/bash/psstat --list-sched-policy-is 0
```

The script is pure Bash + coreutils and reads `/proc/<pid>/{comm,stat,cmdline}`
directly. It degrades gracefully: if a process exits between discovery and read,
that process is skipped (see *ANSWERS.md*, Design note).

## Repository layout

```
.
├── README.md                      this file
├── .gitignore
├── ANSWERS.md                     answers to the assignment Questions
└── src/
    └── bash/
        └── psstat                 the deliverable shell script
```

## How it works (short)

- **Discovery.** PIDs are enumerated from the numeric directory entries under
  `/proc` (`/proc/[0-9]*/`), not from any process-listing tool.
- **`comm` (image name).** Read from `/proc/<pid>/comm` (one line, newline
  trimmed). This is the name that can contain spaces and parentheses.
- **`stat` fields.** `state` (field 3), `minflt` (10), `majflt` (12), `utime`
  (14), `stime` (15), `num_threads` (20), `vsize` (23), `rss` (24),
  `rt_priority` (40) and `policy` (41) are parsed from `/proc/<pid>/stat`.
  Because the `comm` field is wrapped in parentheses and may itself contain
  spaces/parentheses, the script anchors on the *first* `(` and *last* `)` in
  the line before splitting the remaining whitespace-separated fields.
- **`cmdline` (CMD_ARGS).** Read from `/proc/<pid>/cmdline`, where arguments are
  NUL-separated; NULs are rendered as spaces between single-quoted arguments to
  match the reference output.
- **Scheduling policy name.** The numeric `policy` value is mapped to a name
  (`SCHED_NORMAL`, `SCHED_FIFO`, …) using the constants from
  `/usr/include/linux/sched.h`.

## Note on the reference output

The examples on the assignment page come from a fully-loaded Linux box (systemd,
kernel threads, etc.). This repository was developed in a minimal container, so
the set of PIDs and the exact values differ, but the *format* of every mode
matches the specification.
