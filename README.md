<!---
{
  "id": "ec9e6408-a2c2-4ef3-9546-0212928a9abb",
  "teaches": "Debugging: Backtrace After a Segfault and Core Dumps",
  "depends_on": [
    "14d07a93-2c67-4dba-ab7b-4ae61fbd884b",
    "c32afdd3-48c3-4ff2-b5f7-a2179a2f8093"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-05",
  "keywords": [
    "gdb",
    "segfault",
    "SIGSEGV",
    "backtrace",
    "frame",
    "core dump",
    "ulimit",
    "coredumpctl",
    "NULL pointer"
  ]
}
--->

# Debugging: Backtrace After a Segfault and Core Dumps

> In this exercise a program dies with *Segmentation fault* and nothing else. You will run it under GDB to see the exact line, the value of the offending pointer and the chain of calls that led there. Then you will do the same *after the fact*, from a core dump, without having been there when it crashed. No `sudo` is required.

## Introduction

A segmentation fault is the operating system telling you that the program touched memory it does not own. The message carries no line number and no variable. `DBG` cannot help, because the program dies before `stderr` is flushed, and a log only tells you what happened *before*.

GDB catches the signal. At that moment the program is frozen, every variable still has its value, and the stack still records which function called which. `backtrace` prints that chain; `frame N` moves to one of the callers so you can inspect *its* variables. If the crash happened on a machine without GDB, a **core dump** saves the same frozen state to a file that you can open later.

### 1.1) Further Readings and Other Sources

* [GDB Manual: Examining the Stack](https://sourceware.org/gdb/current/onlinedocs/gdb/Stack.html)
* [core(5) man page](https://man7.org/linux/man-pages/man5/core.5.html)
* [coredumpctl(1) man page](https://man7.org/linux/man-pages/man1/coredumpctl.1.html)
* [signal(7) man page: SIGSEGV and friends](https://man7.org/linux/man-pages/man7/signal.7.html)

## Tasks

### Task 1: The Program

```bash
cd ~/dbg-gdb
```

```c
// file: crash.c
#include <stdio.h>

struct node { int value; struct node *next; };

int list_len(struct node *n) {
    int len = 0;
    while (n->next != NULL) {
        len++;
        n = n->next;
    }
    return len + 1;
}

int main(void) {
    struct node c = {3, NULL}, b = {2, &c}, a = {1, &b};
    printf("len = %d\n", list_len(&a));
    printf("len of empty = %d\n", list_len(NULL));
    return 0;
}
```

```bash
gcc -Wall -Wextra -O0 -g crash.c -o crash
./crash; echo "exit status $?"
```

Note the exit status. 139 is 128 + 11, and 11 is the signal number of `SIGSEGV`.

### Task 2: Catch It in GDB

```bash
gdb -q ./crash
```

```
(gdb) run
(gdb) backtrace
(gdb) print n
(gdb) print *n
```

GDB shows the line, `n=0x0`, and `print *n` explains the crash: you cannot dereference address zero.

### Task 3: Walk Up the Stack

```
(gdb) frame 1
(gdb) list
(gdb) info locals
(gdb) print a
(gdb) print *a.next
(gdb) frame 0
```

Frame 1 is `main`. Its local `a` is intact; the list is fine. The bug is the *call* `list_len(NULL)`, not the list.

### Task 4: Decide and Fix

Two valid fixes exist: forbid `NULL` (document it, add `assert(n != NULL)`), or allow it and return 0. Pick one, implement it, rebuild, and run under GDB again to confirm the program exits normally (`[Inferior 1 (process N) exited normally]`).

### Task 5: A Core Dump Without GDB

Undo the fix for a moment, then:

```bash
ulimit -c unlimited
./crash
ls -l core* 2>/dev/null || echo "no core file here"
```

On many modern systems the kernel hands the dump to `systemd-coredump` instead of writing `core` into the directory. Find out which:

```bash
cat /proc/sys/kernel/core_pattern
```

If it starts with `|/usr/lib/systemd/systemd-coredump`, use:

```bash
coredumpctl list ./crash
coredumpctl debug ./crash
```

Otherwise:

```bash
gdb -q ./crash core
```

In both cases GDB opens the frozen state. Run `backtrace` and `print n`. The program is long dead; the information is the same.

### Task 6: A Crash With No Debug Info

```bash
gcc -O2 crash.c -o crash_stripped && strip crash_stripped
gdb -q ./crash_stripped
```

```
(gdb) run
(gdb) backtrace
```

Addresses instead of lines, `??` instead of names. This is what a crash report from a customer looks like when the binary was shipped without symbols. Look up `-g` plus `objcopy --only-keep-debug` to learn how projects keep the symbols separately.

### Task 7: Other Signals

Temporarily add `int z = 0; printf("%d\n", 1 / z);` to `main` (a literal `1 / 0` would be caught by the compiler), rebuild and run under GDB. Which signal is it? Then try `abort();` and `assert(0)`. For each: which signal, and does `backtrace` still work?

## Questions

- What does the exit status 139 encode, and how would a shell script detect a crash?
- Why does `print *n` fail with "Cannot access memory at address 0x0" but `print n` works?
- What is the difference between frame 0 and frame 1 here, and why is `a` not visible in frame 0?
- A core dump of a program using 2 GB of memory is 2 GB. What does `ulimit -c` control, and why is it 0 by default on most systems?
- In Task 6, `backtrace` still showed *something*. Where does that information come from if the symbols are stripped?
- `assert(n != NULL)` versus `if (n == NULL) return 0;`: which one makes the bug in `main` visible, which one hides it?

## Advice

`run`, `backtrace`, `frame`, `print` is the four-command recipe for any crash. It turns "Segmentation fault" into a line number and a value in under a minute, and it is the reason to keep `-g` builds around even for programs you consider finished.

Core dumps extend this to crashes you did not witness. Enable them on your development machine (`ulimit -c unlimited` in your shell profile), learn whether your system uses `coredumpctl`, and you will never again have to reproduce a crash just to see where it happened.
