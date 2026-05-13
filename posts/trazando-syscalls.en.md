# Tracing Syscalls

I just felt like playing around with syscalls a bit and taking the opportunity to explain in more detail how they actually work.

## Minitrace

What’s the conceptual idea we’re going to try here? We fork a process, the child marks itself as traceable with ptrace, and then executes the command. Once that’s done, the parent enters a loop to read and print syscalls.

> [!INFO]
> We’ll take it step by step and understand what’s happening along the way — I think that’s the best approach.

### Printing syscalls

Let’s start simple by printing `syscall!` every time a system call happens, without worrying yet about which syscall it actually is.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "usage: %s <command> [args...]\n", argv[0]);
        return 1;
    }

    pid_t child = fork();

    if (child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execvp(argv[1], &argv[1]);
        perror("execvp");
        exit(1);
    }

    int status;
    waitpid(child, &status, 0);
    ptrace(PTRACE_SETOPTIONS, child, 0, PTRACE_O_TRACESYSGOOD);

    while (1) {
        ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        waitpid(child, &status, 0);

        if (WIFEXITED(status))
            break;

        if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80))
            printf("syscall!\n");
    }

    return 0;
}

```

Let’s pause for a second and look at what we’re actually doing here so it makes sense. I’m skipping some code details because they’re more about making the tool prettier than useful for what we’re trying to understand.

```c
    pid_t child = fork();

    if (child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execvp(argv[1], &argv[1]);
        perror("execvp");
        exit(1);
    }
```

In the parent process, `child` contains the PID of the child process, while in the child itself `child` equals 0. Knowing that, what we’re doing is: if `child == 0`, meaning we’re inside the child process, we call `ptrace` to tell the kernel that this process is ready to be debugged.

From this point on, the child becomes the `tracee`, and the parent becomes the `tracer` ([ptrace manual](https://man7.org/linux/man-pages/man2/ptrace.2.html)).

Cool, but what about the rest? Well, we call `execvp` using the argument passed to our binary, for example:

```bash
./minitrace cat /etc/hostname
```

In this specific case we’d get `cat`. What we’re basically doing is replacing the child process we created with the command we passed in, in this case the `cat` binary.

Everything else is just error handling and exiting the program.

> [!INFO]
> IMPORTANT! When using `PTRACE_TRACEME`, the child sends a `SIGTRAP` signal that the parent can use to start tracing.

Why use `execvp` instead of `execve` like the manual mentions? Simple convenience. This way we can search the binary in the system PATH without needing the absolute path.

Let’s continue.

```c
    int status;
    waitpid(child, &status, 0);
    ptrace(PTRACE_SETOPTIONS, child, 0, PTRACE_O_TRACESYSGOOD);
```

Here we use [waitpid](https://man7.org/linux/man-pages/man3/waitpid.3p.html), which waits for a process PID, a status variable, and some options. Basically, we’re catching that first `SIGTRAP` from the kernel so we can start controlling the `tracee`.

Then we call `ptrace` again, this time with `SETOPTIONS`, enabling the `PTRACE_O_TRACESYSGOOD` option.

Right now there’s a problem: both normal instructions and syscalls would trigger a `SIGTRAP`, so we wouldn’t be able to tell which stops are actual syscalls. That’s why we enable this option — from now on, syscall entry and exit stops get marked with the `0x80` bit. We’ll understand this better in a second.

```c
    while (1) {
        ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        waitpid(child, &status, 0);

        if (WIFEXITED(status))
            break;

        if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80))
            printf("syscall!\n");
    }
```

Here’s the crown jewel of what we’ve built so far.

We enter an “infinite” loop, and the first thing we do is resume execution of the child process, telling it to stop at the next syscall entry or exit. Then `waitpid` blocks the parent until the child stops again.

Now we get to the two conditionals:

The first one just exits the loop when the child process finishes.

The second one is the important one, and the whole reason we enabled `PTRACE_O_TRACESYSGOOD`. Let’s break it down.

* `WIFSTOPPED` from `sys/wait.h` becomes true if the child stopped because of a `SIGTRAP`.
* `WSTOPSIG` extracts the signal number that caused the stop, and this is where we compare things...
* `SIGTRAP | 0x80` → meaning the `0x80` bit OR’d with `SIGTRAP` (`0x5`), giving us `0x85` (`133`), which is what we expect to receive.

It’s important to remember that `|` in C is the bitwise OR operator. Let’s visualize it — in this case it behaves like an addition because the bits don’t overlap.

```binary
  5:   00000000 00000000 00000000 00000101
128:   00000000 00000000 00000000 10000000
OR:    00000000 00000000 00000000 10000101  = 133
```

So `WSTOPSIG(status)` equals `133` whenever we’re dealing with a syscall stop, because that `0x80` bit gets added by the option we enabled earlier.

I temporarily added a couple of variables to show it more clearly.

![SIGTRAP variables](/trazando-syscalls/sigtrap_variables.png)

For now, once we enter the conditional, we simply print `syscall!`.

> [!WARNING]
> Remember that we’ll receive both the syscall entry and syscall exit events.

Let’s test it. Compile and see what happens...

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
syscall!
syscall!
syscall!
...
syscall!
demo-ubuntu
syscall!
syscall!
...
syscall!
```

A few syscalls later and the hostname appears — voilà, first part done.

---

### Printing and mapping syscall numbers

Now we want to know which syscalls are actually being called, so let’s work on that.

```c
struct syscall_entry {
    long number;
    const char *name;
};

static const struct syscall_entry syscall_table[] = {
    {   0, "read"    },
    {   1, "write"   },
    {   3, "close"   },
    {   5, "fstat"   },
    {   9, "mmap"    },
    {  12, "brk"     },
    {  59, "execve"  },
    { 257, "openat"  },
};

static const char *syscall_name(long nr) {
    for (size_t i = 0; i < sizeof(syscall_table) / sizeof(syscall_table[0]); i++) {
        if (syscall_table[i].number == nr)
            return syscall_table[i].name;
    }
    return NULL;
}
```

First we create a `struct` defining the types, then we build an array called `syscall_table` that we’ll query using syscall numbers.

> [!WARNING]
> These numbers belong to the x86_64 architecture. There’s a full list at filippo.io/linux-syscall-table/

Then we have a simple lookup function: pass in the syscall number, iterate over the table, and return the name.

```c
if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child, NULL, &regs);
            long nr = (long)regs.orig_rax;
            const char *name = syscall_name(nr);
            if (name)
                printf("%s\n", name);
            else
                printf("syscall_%ld\n", nr);
        }
```

Now we modify our second conditional so instead of printing `syscall!`, we print the mapped syscall name if we know it.

`user_regs_struct` comes from `sys/user.h` and lets us access all CPU registers (`rax`, `rbx`, `rcx`, `rdx`, `rsp`, `rip`, etc...). We use `ptrace` again, this time to fetch the registers.

One important detail: on x86_64, the syscall number is stored in the `orig_rax` register.

Let’s confirm this with radare:

```bash
miguel@demo-ubuntu:~/low-level/minitrace$ r2 -d /bin/cat 
[0x7ed619ee6540]> ood /etc/hostname
child received signal 9
File dbg:///usr/bin/cat /etc/hostname reopened in read-write mode
[0x7dec35315540]> dcs
Running child until next syscall
--> SN 0x7dec3531a9cb syscall 12 unknown (0x0 0x7dec3532ca18 0x0)
[0x7dec3531a9cb]> dr
rax = 0xffffffffffffffda
rbx = 0x7dec35316af0
rcx = 0x7dec3531a9cb
rdx = 0x00000000
r8 = 0x00000000
r9 = 0x7dec3532c440
r10 = 0x0000037f
r11 = 0x00000246
r12 = 0x7ffe3494cd50
r13 = 0x00000000
r14 = 0x7dec352f6000
r15 = 0x7dec352f6590
rsi = 0x7dec3532ca18
rdi = 0x00000000
rsp = 0x7ffe3494cc88
rbp = 0x7ffe3494ccc0
rip = 0x7dec3531a9cb
rflags = 0x00000246
orax = 0x0000000c
[0x7dec3531a9cb]> ?vi `dr orax`
12
```

What we’re basically doing is running the same binary and asking radare to stop before the next syscall. It already tells us it’s syscall number 12, but since we don’t trust it blindly, we inspect all the registers and confirm that `orig_rax` (`orax = 0x0000000c`) indeed equals 12.

And syscall number 12 is the first syscall we see, which in our table maps to `brk`. So now, when we run our tracer, look at the first syscall.

Here’s the radare panel view as well, although I’ll warn you already — it’s not exactly pleasant.

![radare orax](/trazando-syscalls/radare_orax.png)

> [!INFO]
> As a small detail, notice that when a syscall isn’t mapped, we now append the number so we can map or search it manually later.

Let’s run it and see what happens:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk
brk
mmap
mmap
syscall_21
syscall_21
openat
openat
...
read
read
write
demo-ubuntu
write
read
read
...
close
close
close
close
syscall_231
```

Some interesting things already show up here.

For starters, every syscall appears twice. That’s because when using `PTRACE_SYSCALL`, the process stops both on syscall entry and syscall exit, so we print the same syscall name twice. For now we’ll leave it like that.

### Printing the first argument

Let’s keep moving forward by printing the first syscall argument.

```c
if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child, NULL, &regs);
            long nr = (long)regs.orig_rax;
            const char *name = syscall_name(nr);
            if (name)
                printf("%s(%llu)\n", name, regs.rdi);
            else
                printf("syscall_%ld(%llu)\n", nr, regs.rdi);
        }
    }
```

The change here is that we added an `unsigned long long` (`llu`) printing the value stored in `rdi`.

And what is `rdi` exactly?

On x86_64, syscall arguments are usually passed in registers following this order:

```text
rdi, rsi, rdx, r10, r8, r9
```

So `rdi` is simply the first argument passed to the syscall.

Let’s run it and see:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk(0)
brk(0)
mmap(0)
mmap(0)
syscall_21(131031247156656)
syscall_21(131031247156656)
openat(4294967196)
openat(4294967196)
fstat(3)
fstat(3)
...
read(3)
read(3)
write(1)
demo-ubuntu
write(1)
read(3)
read(3)
...
close(3)
close(3)
close(1)
close(1)
close(2)
close(2)
syscall_231(0)
```

Nice, this is starting to take shape.

One of the first things we notice is that the first argument is identical on syscall entry and exit. That’s because the kernel restores the register before returning from the syscall.

For example, `syscall_21` has a huge number as its first argument. Looking it up in [the table](https://filippo.io/linux-syscall-table/), it turns out to be `access`, and that giant value is probably a memory address. That immediately tells us that if we want more meaningful tracing later on, hexadecimal output would be much easier to work with visually.

Let’s inspect it more closely.

![radare rdi](/trazando-syscalls/radare_rdi_memory.png)

The first register, `rdi`, contains a specific hexadecimal value. Just like we suspected, it looks like a memory address — so let’s follow it.

![radare rdi](/trazando-syscalls/radare_rdi_memory_2.png)

And yeah, it makes perfect sense. The process is trying to access the `ld.so.preload` library, so mystery solved.

Ignoring some of the less interesting results, I want to focus on the smaller arguments like `1` and `3`.

For example, `read(3)` means it’s reading from file descriptor 3, which in this case is `/etc/hostname`.

And then we have `write(1)`, BEAUTIFUL, because it connects directly with the article [What are STDIN, STDOUT, STDERR?](/blog/stdin-stdout-stderr), where we already covered this:

| Column 1 | Column 2 |
| -------- | -------- |
| 0        | STDIN    |
| 1        | STDOUT   |
| 2        | STDERR   |

So we’re writing to `STDOUT`, and since we’re executing the binary directly, we see the output of `/etc/hostname` printed in the terminal.

And finally we see `close(1,2,3)` because the process closes all its file descriptors before exiting.

### What if they take a string? Let’s print it too

Let’s grab even more information about what’s happening inside each syscall.

```c
struct syscall_entry {
    long        number;
    const char *name;
    int         str_arg; /* which positional argument is a string: 0=none, 1=rdi, 2=rsi */
};

static const struct syscall_entry syscall_table[] = {
    {   0, "read",   0 },
    {   1, "write",  2 },
    {   3, "close",  0 },
    {   5, "fstat",  0 },
    {   9, "mmap",   0 },
    {  12, "brk",    0 },
    {  59, "execve", 1 }, 
    { 257, "openat", 2 }, 
};

static void read_child_string(pid_t pid, unsigned long long addr, char *buf, size_t size) {
    struct iovec local  = { .iov_base = buf,          .iov_len = size - 1 };
    struct iovec remote = { .iov_base = (void *)addr, .iov_len = size - 1 };
    ssize_t n = process_vm_readv(pid, &local, 1, &remote, 1, 0);
    if (n < 0) {
        buf[0] = '?';
        buf[1] = '\0';
        return;
    }
    buf[n] = '\0';
}
```

What’s this?

A helper function that lets us read a string directly from the memory of the `tracee` without needing to use `PTRACE_PEEKDATA`.

To do this, it relies on the `process_vm_readv` syscall, which allows transferring memory between the child and parent processes.

We use two `iovec` structures from `sys/uio.h`, where each one defines a buffer address and its size in bytes:

* `local` → points to the buffer where we’ll store the data, and `size - 1` leaves room for the null terminator.
* `remote` → points to the memory location containing the string we want to read.

> [!DANGER]
> Remember this line: `buf[n] = '\0'`, because it appends the null byte after the bytes we read.

We also updated our syscall mapping structure so it can indicate which argument is expected to be a string.

```c
if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
      struct user_regs_struct regs;
      ptrace(PTRACE_GETREGS, child, NULL, &regs);
      long nr = (long)regs.orig_rax;
      const struct syscall_entry *entry = syscall_lookup(nr);

      char str[STR_MAX];
      const char *name = entry ? entry->name : NULL;

      if (!entry) {
          printf("syscall_%ld(%llu)\n", nr, regs.rdi);
      } else if (entry->str_arg == 1) {
          read_child_string(child, regs.rdi, str, sizeof(str));
          printf("%s(\"%s\")\n", name, str);
      } else if (entry->str_arg == 2) {
          read_child_string(child, regs.rsi, str, sizeof(str));
          printf("%s(%d, \"%s\")\n", name, (int)regs.rdi, str);
      } else {
          printf("%s(%llu)\n", name, regs.rdi);
      }
  }
```

More changes to our main conditional.

Before we were calling `syscall_name`, but now we need to go one step further and call `syscall_lookup`, which returns a pointer to the full syscall entry.

Now we handle three possible cases:

* We don’t know the syscall → `!entry`
* The first argument is a string → use `read_child_string()`
* The second argument is a string and the first one is an integer → also use `read_child_string()`
* Everything else → treat arguments as integers

Let’s compare the second and third cases:

```c
# Second case
else if (entry->str_arg == 1) {
  read_child_string(child, regs.rdi, str, sizeof(str));

# Third case
else if (entry->str_arg == 2) {
  read_child_string(child, regs.rsi, str, sizeof(str));
```

We manually check whether the first or second argument is a string, and then read the appropriate register. Remember the order we mentioned earlier: first `rdi`, then `rsi`.

Let’s run it. I’ll only show the interesting parts:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk(0)
brk(0)
...
openat(-100, "/etc/ld.so.cache")
openat(-100, "/etc/ld.so.cache")
...
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")
...
openat(-100, "/usr/lib/locale/locale-archive")
openat(-100, "/usr/lib/locale/locale-archive")
...
openat(-100, "/etc/hostname")
openat(-100, "/etc/hostname")
...
syscall_231(0)
```

We can already see several `openat` calls happening before the program even touches `/etc/hostname`.

First it checks the library cache to resolve paths (`ld.so.cache`), then opens the standard C library (`/lib/x86_64-linux-gnu/libc.so.6`), and it also loads locale data (`/usr/lib/locale/locale-archive`).

Notice that all of them use descriptor `-100`.

Earlier we saw it like this:

```bash
openat(4294967196)
```

That happened because we previously printed `regs.rdi` directly, but now we changed it slightly to `(int)regs.rdi`. That huge number `4294967196` is simply the unsigned representation of `-100`.

And what is `-100`?

That’s `AT_FDCWD`, which basically means “current working directory”.

### Let’s distinguish syscall entry from exit

```c
int in_syscall = 0;

    while (1) {
        ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        waitpid(child, &status, 0);

        if (WIFEXITED(status))
            break;

        if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child, NULL, &regs);

            if (!in_syscall) {
                long nr = (long)regs.orig_rax;
                const struct syscall_entry *entry = syscall_lookup(nr);
                char str[STR_MAX];

                if (!entry) {
                    printf("syscall_%ld(%llu)", nr, regs.rdi);
                } else if (entry->str_arg == 1) {
                    read_child_string(child, regs.rdi, str, sizeof(str));
                    printf("%s(\"%s\")", entry->name, str);
                } else if (entry->str_arg == 2) {
                    read_child_string(child, regs.rsi, str, sizeof(str));
                    printf("%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
                } else {
                    printf("%s(%llu)", entry->name, regs.rdi);
                }
            } else {
                long ret = (long)regs.rax;
                if (ret < 0)
                    printf(" = %ld\n", ret);
                else
                    printf(" = %ld\n", ret);
            }

            in_syscall = !in_syscall;
        }
    }
```

Now we’re getting rid of duplicated syscall lines by distinguishing entry and exit.

This isn’t the most perfect solution in the world, but it works fine for our purposes. We simply toggle an `in_syscall` flag.

We also removed the newline from the first print so we can append more information afterward.

```c
else {
                long ret = (long)regs.rax;
                if (ret < 0)
                    printf(" = %ld\n", ret);
                else
                    printf(" = %ld\n", ret);
            }
```

Here we read the value of `rax`, which contains the syscall return value.

So now we have a lot more information:

* file descriptors
* strings
* return values
* and no duplicated entry/exit lines

Let’s run it:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk(0) = 107374942257152
mmap(0) = 130344951947264
syscall_21(130344952182192) = -2
openat(-100, "/etc/ld.so.cache") = 3
fstat(3) = 0
mmap(0) = 130344951877632
close(3) = 0
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6") = 3
read(3) = 832
syscall_17(3) = 784
fstat(3) = 0
syscall_17(3) = 784
mmap(0) = 130344948858880
mmap(130344949022720) = 130344949022720
mmap(130344950628352) = 130344950628352
mmap(130344950951936) = 130344950951936
mmap(130344950976512) = 130344950976512
close(3) = 0
mmap(0) = 130344951865344
syscall_158(4098) = 0
syscall_218(130344951867920) = 102625
syscall_273(130344951867936) = 0
syscall_334(130344951869536) = 0
syscall_10(130344950951936) = 0
syscall_10(107374131417088) = 0
syscall_10(130344952209408) = 0
syscall_302(0) = 0
syscall_11(130344951877632) = 0
syscall_318(130344950997368) = 8
brk(0) = 107374942257152
brk(107374942392320) = 107374942392320
openat(-100, "/usr/lib/locale/locale-archive") = 3
fstat(3) = 0
mmap(0) = 130344934178816
close(3) = 0
fstat(1) = 0
openat(-100, "/etc/hostname") = 3
fstat(3) = 0
syscall_221(3) = 0
mmap(0) = 130344951726080
read(3) = 31
demo-ubuntu
write(1) = 31
read(3) = 0
syscall_11(130344951726080) = 0
close(3) = 0
close(1) = 0
close(2) = 0
syscall_231(0)
```

Does all of this start looking familiar?

Let’s compare it against a much more well-known command: `strace`.

```bash
miguel@demo-ubuntu:~/minitrace$  strace cat /etc/hostname
execve("/usr/bin/cat", ["cat", "/etc/hostname"], 0x7ffdc2d6b3e8 /* 52 vars */) = 0
brk(NULL)                               = 0x59f6f8137000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x79f191157000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=67127, ...}) = 0
mmap(NULL, 67127, PROT_READ, MAP_PRIVATE, 3, 0) = 0x79f191146000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
fstat(3, {st_mode=S_IFREG|0755, st_size=2125328, ...}) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 2170256, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x79f190e00000
...
```

Looks somewhat similar, right?

Sure, ours took a while to build and is extremely simplified, while `strace` is a mature production-grade tool, but you can already see the resemblance.

## Some extra details

Haven’t you wondered why I never printed the string argument for `write`? Let’s look at it.

```c
static const struct syscall_entry syscall_table[] = {
   ...
    {   1, "write",  2 },
  ...
};
```

The string lives in the second argument.

So what happens if we enable it?

```bash
read(0x3) = 31
write(1, "demo-ubuntu
demo-ubuntu
") = 31
read(0x3) = 0
```

We hit a small issue here.

What’s happening is that we merged syscall entry and exit into a single line, but before the syscall returns, the command itself already writes its output to stdout.

So we get:

```text
write(1, "demo-ubuntu
```

then the actual program output:

```text
demo-ubuntu
```

and finally the syscall return:

```text
") = 31
```

Wait… are we the only ones hitting this problem?

```bash
miguel@demo-ubuntu:~/minitrace$  strace cat /etc/hostname
execve("/usr/bin/cat", ["cat", "/etc/hostname"], 0x7ffdc2d6b3e8 /* 52 vars */) = 0
...
read(3, "demo-ubuntu", 131072) = 31
write(1, "demo-ubuntu", 31demo-ubuntu
) = 31
read(3, "", 131072)                     = 0
...
```

Looks like `strace` has the exact same issue.

But actually, `strace` already solved it — it’s just invisible to us right now.

If we remember again [What are STDIN, STDOUT, STDERR?](/blog/stdin-stdout-stderr):

We have `stdout` for anything “useful” that we might want to pipe elsewhere, while `stderr`, despite what many people think, is not just for errors. It’s also meant for extra information that programs may want to display without interfering with normal output.

We know stdout is file descriptor `1`, so let’s try something:

```bash
miguel@demo-ubuntu:~/minitrace$  strace cat /etc/hostname 1>/dev/null
execve("/usr/bin/cat", ["cat", "/etc/hostname"], 0x7ffdc2d6b3e8 /* 52 vars */) = 0
...
read(3, "demo-ubuntu\n", 131072) = 31
write(1, "demo-ubuntu\n", 31) = 31
read(3, "", 131072)                     = 0
...
```

By redirecting stdout into the infinite void with `1>/dev/null`, the problem disappears and the trace becomes much easier to read.

As a small improvement, we could do the same thing in our own tracer.

```c
fprintf(stderr, ...)
```

We simply replace `printf` with `fprintf(stderr, ...)` so our tracer writes to stderr instead of stdout.

```bash
miguel@demo-ubuntu:~/minitrace$  ./minitrace cat /etc/hostname 1>/dev/null
...
read(0x3) = 31
write(1, "demo-ubuntu
") = 31
read(0x3) = 0
...
```

Well... we got rid of stdout interference, but there’s still a newline issue.

That’s because `write` is literally doing this:

```c
write(1, "demo...\n", 31)
```

You can actually see it clearly in the `strace` output:

```text
"demo-ubuntu\n"
```

Still, this doesn’t really concern us right now. We’re not trying to build a fully production-ready `strace` clone from scratch — the goal is simply to understand what’s happening under the hood.

And there are plenty of ways to solve this properly anyway.

I think that’s enough for today. See you in the next one.
