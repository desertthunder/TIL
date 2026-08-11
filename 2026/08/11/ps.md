---
title: Inspecting Processes with ps
description: Process selection, custom output, state codes, and portability across Unix systems
tags:
  - unix
  - command-line
  - processes
  - linux
---

`ps`, short for process status, reports a snapshot of selected processes. It can answer
questions like who owns a PID, when it started, and which process launched it. The
output does not update so tools like `top`, `htop`, or another monitor can watch changes
over time.[^1]

Running `ps` without options usually shows processes owned by the current effective user
and attached to the current terminal. The exact columns vary, but normally include the
PID, terminal, accumulated CPU time, and command. Daemons and programs launched from
another terminal may therefore be absent from plain `ps` output.

## Three option styles

Linux's procps-ng implementation accepts three option styles:[^1]

| Style      | Form       | Examples                   |
| ---------- | ---------- | -------------------------- |
| Unix/POSIX | one dash   | `-e`, `-f`, `-p 123`, `-o` |
| BSD        | no dash    | `a`, `x`, `u`, `aux`       |
| GNU-style  | two dashes | `--sort`, `--forest`       |

On Linux, `ps` normally comes from procps-ng. Its manual calls the double-dash form
“GNU-style”; `ps` itself is not part of GNU Coreutils. Mixing styles can change both the
selected processes and the default columns. In scripts, explicit Unix-style selection
and `-o` output are easier to reason about.

Common forms have different meanings:

- `ps -e` or `ps -A` selects every process.
- `ps -ef` selects every process and requests the Unix full format.
- `ps aux` uses BSD options: `a` includes other users' terminal processes, `x` includes
  processes without a controlling terminal, and `u` selects a user-oriented format.
- `ps -aux` should be avoided. On procps-ng it is formally a request involving the user
  named `x`, with a compatibility fallback when that user does not exist.[^1]

POSIX standardizes `-A`, `-e`, `-a`, `-f`, `-l`, several selectors, and `-o`. It does not
standardize BSD's dashless bundles or procps-ng long options.[^2] BSD descendants also
retain historical conflicts: FreeBSD documents that options such as `-e`, `-f`, `-g`,
and `-u` do not all have their POSIX meanings.[^3]

## Select processes and columns

Selection options choose processes. `-o` chooses and orders columns. Some
implementations also support sorting.

```sh
ps -A                    # every process
ps -p 123,456            # these PIDs
ps -t pts/2              # processes attached to a terminal
ps -u alice              # effective user, on POSIX/procps-ng ps
```

On procps-ng, multiple positive selectors are additive. A process appears if it matches
any of them. Supplying a selector also discards the narrow default selection. Thus,
`ps -p 123 -u alice` selects PID 123 or Alice's processes.[^1]

`-o` produces a narrower, predictable listing:

```sh
ps -p 123 -o user,pid,ppid,pgid,stat,lstart,etime,time,args
ps -A -o user,pid,ppid,tty,etime,time,args
```

`-o` accepts a comma- or space-separated list. It may be repeated, and a field can be
given a custom header with `field=HEADER`. An empty header suppresses that heading; if
every heading is empty, `ps` omits the header row. That is handy when a script needs one
value:

```sh
ps -p 123 -o ppid=
```

Programs that need reliable process metadata should use `/proc`, a system API, or a
service manager. `ps` output can change with the operating system, locale, terminal
width, selected personality, and version.

## Reading the common fields

| Field            | Meaning                                                     |
| ---------------- | ----------------------------------------------------------- |
| `PID`            | Process ID                                                  |
| `PPID`           | Parent process ID                                           |
| `PGID`           | Process group ID, used for job control and group signaling  |
| `SID`            | Session ID                                                  |
| `TTY`            | Controlling terminal; `?` or `-` commonly means none        |
| `STAT`           | Current state plus optional modifiers                       |
| `START`/`LSTART` | Start time in a compact/full representation                 |
| `ETIME`          | Wall-clock time since the process started                   |
| `TIME`           | Accumulated user plus system CPU time                       |
| `%CPU`           | Implementation-defined CPU-use calculation                  |
| `%MEM`           | Resident memory as a percentage of physical memory on Linux |
| `RSS`            | Resident set size, normally in KiB                          |
| `VSZ`            | Virtual address-space size, normally in KiB                 |
| `COMM`           | Executable or process name                                  |
| `ARGS`           | Command line, possibly modified, unavailable, or truncated  |

`TIME` and `ETIME` answer different questions. A process can have existed for a day
(`ETIME`) while using only seconds of CPU (`TIME`) because it spent most of that day
sleeping or waiting.

On Linux, `%CPU` is CPU time divided by the process's lifetime. BSD implementations may
use a decaying recent average instead. Use a live monitor for current CPU load.[^1][^3]

`RSS` estimates pages currently resident in physical memory. `VSZ` measures the virtual
address space, which can include reserved but untouched mappings and shared libraries.
RSS values also include shared pages in more than one process, so adding every process's
RSS can overcount physical memory. These columns are useful for finding processes worth
further inspection, but not for complete memory accounting.

A process can change the title that tools display. Arguments may also be unavailable or
truncated. Put `args` last and request wide output during interactive investigation.

## Process state

The first character of `STAT` is the main state. These codes are common on procps-ng:

| Code | State                                                   |
| ---- | ------------------------------------------------------- |
| `R`  | Running or runnable                                     |
| `S`  | Interruptible sleep, waiting for an event               |
| `D`  | Uninterruptible sleep, usually waiting for I/O on Linux |
| `T`  | Stopped by job control                                  |
| `t`  | Stopped while being traced                              |
| `Z`  | Zombie: exited, but not yet reaped by its parent        |
| `I`  | Idle kernel thread on Linux                             |

Later characters are modifiers. On Linux, `<` and `N` indicate raised and reduced
scheduling priority, `s` marks a session leader, `l` a multithreaded process, and `+` a
member of the terminal's foreground process group.[^1] The vocabulary is not identical
across systems; for example, FreeBSD also uses `I` for a process idle for more than about
20 seconds.[^3]

A zombie is no longer executing. Its parent has not collected its termination status
with a `wait` operation, so the kernel keeps a small process-table record. Investigate
the parent that failed to reap it.

## Practical queries on Linux

procps-ng adds sorting, trees, and thread views:

```sh
# Highest lifetime CPU ratios first
ps -eo pid,ppid,user,stat,etime,%cpu,%mem,rss,args --sort=-%cpu

# Largest resident sets first
ps -eo pid,user,%mem,rss,vsz,args --sort=-rss

# Show parent-child relationships
ps -e -o pid,ppid,stat,etime,args --forest

# Show lightweight processes (threads)
ps -eLf
```

`--sort` accepts comma-separated keys, with `-` for descending and `+` for ascending.
These long options and the exact thread flags are procps-ng features, so check `man ps`
before using them on macOS or another BSD-derived system.

Use `pgrep` instead of `ps ... | grep name` for process discovery. The pattern can match
the `grep` command itself, command lines are mutable, and the target can exit between
being listed and acted upon. Any subsequent action still has a race unless the operating
system provides a stable process handle.

The system continues running while `ps` constructs its listing. Processes can start,
exit, change state, or change resource use between reads. Each row may already be stale
by the time it is printed.[^3]

[^1]: procps-ng, [`ps(1)` Linux manual page](https://man7.org/linux/man-pages/man1/ps.1.html).

[^2]: IEEE and The Open Group, [`ps` in the POSIX Programmer's Manual](https://man7.org/linux/man-pages/man1/ps.1p.html).

[^3]: The FreeBSD Project, [`ps(1)` manual page](https://man.freebsd.org/cgi/man.cgi?query=ps&sektion=1&manpath=FreeBSD+16.0-CURRENT).
