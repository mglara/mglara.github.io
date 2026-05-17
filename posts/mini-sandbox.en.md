# Mini Sandbox

It’s a pleasure to have you back for another day. What we’re covering today is really based on the following article: [Tracing Syscalls](/blog/tracing-syscalls). I’d strongly recommend checking that one out before we begin.

## Introduction

Picking up where we left off, we had a tiny little strace clone working. It still had plenty of gaps and definitely couldn’t be turned into a proper production-ready tool, but it was more than enough to keep poking around Linux a bit more.

Today’s plan is to build a mini sandbox environment. Maybe calling it a sandbox is overselling it a little, but what we’re going to do is allow any program to run through our binary while we control the actions it can perform.

Like I said, don’t expect something production-grade that you’ll actually use in the real world. In fact, the way we’re doing this isn’t even the modern approach anymore. Still, understanding these foundations is incredibly important, so let’s jump in.

## Phase 0 — Kill the process

What are we going to do? Define a forbidden syscall without making huge changes to our code. Remember, you can see how the code looked in [Tracing Syscalls](/blog/tracing-syscalls).

```c
#include <stdbool.h>

struct syscall_entry {
    long        number;
    const char *name;
    int         str_arg;
    bool        blacklist; /*0 means not blacklisted, 1 means blacklisted*/
};

static const struct syscall_entry syscall_table[] = {
    {   0, "read",   0, 0 },
    {   1, "write",  2, 0 },
    {   3, "close",  0, 0 },
    {   5, "fstat",  0, 0 },
    {   9, "mmap",   0, 0 },
    {  12, "brk",    0, 0 },
    {  59, "execve", 1, 1 }, 
    { 257, "openat", 2, 0 }, 
};
```

First of all, the quickest solution I came up with without heavily modifying the code — not necessarily the best one — is the one you’re seeing here. We add a boolean to the structure. That’s all we need: if it’s `0`, the syscall is allowed; if it’s `1`, the syscall is blacklisted.

For this test I blacklisted `execve`. Why, you ask?

Why not?

Okay, serious answer now: `execve` allows execution, and in certain environments or scenarios that may be exactly what we want to prevent.

> [!DANGER]
> CAREFUL: you absolutely cannot do this system-wide unless you want to break everything.

```c
if (!entry) {
    fprintf(stderr, "syscall_%ld(0x%llx)", nr, regs.rdi);
} else if (entry->blacklist == 1) {
    kill(child, SIGKILL);
    fprintf(stderr, "%s(\"%s\")", entry->name, str);
    fprintf(stderr, "\nBLOCKED\n");
    return 0;
```

And we only need to add a new `else if`. This checks the `blacklist` value, and if it’s `true`, it kills the process outright with `kill(child, SIGKILL)`.

Brutal? Absolutely.

Elegant? Not really.

Effective? Definitely.

The `return 0` is there so the parent exits immediately too. Let’s see the result:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace env ls
brk(0x0) = 94961438253056
mmap(0x0) = 133098360590336
syscall_21(0x790d608465b0) = -2
...
syscall_302(0x0) = 0
syscall_11(0x790d607fb000) = 0
syscall_318(0x790d6060a178) = 8
brk(0x0) = 94961438253056
brk(0x565deecd7000) = 94961438388224
openat(-100, "/usr/lib/locale/locale-archive") = 3
fstat(0x3) = 0
mmap(0x0) = 133098341662720
close(0x3) = 0
execve("/usr/lib/locale/locale-archive")
BLOCKED
```

Looks like it works. Good enough for now...

## Phase 1 — Silent failure

Let’s make things a bit more interesting. Simply blocking the syscall isn’t enough anymore. This time, we want to choose the exact error code returned to the process.

```c
#include <errno.h>
```

Remember to include the error library so we can reference error codes directly.

```c
...
int in_syscall = 0;
bool sandbox_blocked = 0;

while (1) {
...
```

Right before the `while`, we add a convenience boolean that tells us when we decided to block a syscall. You’ll see why in a second.

```c
if (!entry) {
    fprintf(stderr, "syscall_%ld(0x%llx)", nr, regs.rdi);
} else if (entry->blacklist == 1) { 
    fprintf(stderr, "%s(0x%llx)", entry->name, regs.rdi);
    fprintf(stderr, "BLOCKED");
    regs.orig_rax = -1; 
    ptrace(PTRACE_SETREGS, child, 0, &regs);
    sandbox_blocked = 1;   
```

I changed a few things here.

For starters, I don’t want to overcomplicate things, so we’re no longer printing arguments based on our array.

I *do* still want it to warn me with `BLOCKED`, though.

Now for the interesting part: before, we killed the process immediately. Now we’re just going to lie to it.

Here’s what’s happening:

`regs.orig_rax = -1` followed by `ptrace(PTRACE_SETREGS, child, 0, &regs);` changes the syscall number into an invalid syscall before the kernel actually executes it.

Why?

Because if we’re blocking something like a write operation, we don’t want the write to happen *before* we inject our fake error code.

Let’s rewind for a second. Syscalls enter the kernel, perform their action, then return to userspace. That’s why in [Tracing Syscalls](/blog/tracing-syscalls) we originally saw duplicated outputs.

If we only modify the *return value* after the syscall executes, then even though we can “fake” the output, the action itself already happened.

Turning the syscall into an invalid syscall is our safeguard against that.

And of course we set our blocking boolean to `1` so the syscall exit handler knows what happened.

```c
if (sandbox_blocked) {
    regs.rax = -EACCES;                
    ptrace(PTRACE_SETREGS, child, 0, &regs);
    sandbox_blocked = 0;
}
```

Now we inject the return error. In this case it’s as simple as modifying `rax` with whatever error we want.

I’ll show outputs using both `-EACCES` and `-EPERM` so the difference becomes obvious.

**Output using -EACCES**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
brk(0x0) = 104202677182464
mmap(0x0) = 125737236832256
writev(0x725b7b59b5b0) = -2
openat(0xffffff9c)BLOCKED = -38
...
access(0x2)cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Permission denied
 = 104
exit_group(0x7f)
```

**Output using -EPERM**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
brk(0x0) = 99751971921920
mmap(0x0) = 133053564239872
writev(0x7902f27255b0) = -2
openat(0xffffff9c)BLOCKED = -38
...
access(0x2)cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Operation not permitted
 = 110
exit_group(0x7f)
```

You can see how one says `Permission denied` while the other says `Operation not permitted`.

But even more interestingly, the number of syscalls changes depending on the error returned.

Think about what just happened:

From the parent process, we lied to the child process by inventing an error code. The child accepted it as if it came directly from the kernel — essentially lying to poor `cat` through the channel it trusts the most.

That caused it to behave differently depending on the error it received, entirely due to its own internal error handling logic.

Let’s tweak the code slightly before our syscall-entry `if`:

```c
read_child_string(child, regs.rsi, str, sizeof(str));
fprintf(stderr, "%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
```

And run it again.

**Output using -EACCES**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
...
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")BLOCKED = -38
...
cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Permission denied
```

**Output using -EPERM**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
...
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")BLOCKED = -38
...
cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Operation not permitted
```

When using `-EPERM`, the binary basically says: “If you won’t allow this operation, I’m not wasting more time.”

With `-EACCES`, though, it says: “No problem, if I can’t read `/lib/x86_64-linux-gnu/libc.so.6`, I’ll just keep searching other directories.”

That can be *very* interesting when trying to understand how binaries handle errors — or even exploit that behavior.

What happens if I discover I have write permissions on one of those fallback directories, and that binary runs with SUID or through a cron job under another user? 😈

> [!WARNING]
> Did you notice we’re printing `-38` as the error?

`-38` is actually `-ENOSYS`, meaning “Function not implemented.”

That’s because we capture the return value *before* overwriting it:

```c
else {
    /* exit: rax contains the return value */
    long ret = (long)regs.rax;
    if (sandbox_blocked) {
        regs.rax = -EPERM;                
        ptrace(PTRACE_SETREGS, child, 0, &regs);
        sandbox_blocked = 0;
    }
    if (ret < 0)
        fprintf(stderr, " = %ld\n", ret);
    else
        fprintf(stderr, " = %ld\n", ret);
}
```

When we print at the end, we’re still printing the real kernel return value after our syscall replacement.

Fixing this is easy, but it’s useful because it shows the kernel genuinely returned a real error code.

Still, there’s already an important lesson here:

Building blocking policies this way is *extremely* fragile.

We didn’t even let `cat` reach the `/etc/passwd` access we cared about, because before it got there it needed to load a library — which we also blocked.

And this is why low-level system protection is so delicate: one mistake can bring down an entire system.

## Phase 2 — What if we want more granular policies?

Right now our logic is basically:

“You’re either allowed or blocked.”

But what if I only want to block *specific* cases?

We already saw that the previous approach completely destroyed the binary’s normal behavior. So let’s try blocking only certain paths — for example, allowing `openat` except for specific files.

> [!WARNING]
> Before you yell at me: yes, I know this code isn’t exactly clean. That’s not the point here. The goal is understanding the concepts behind it.

```c
#include <string.h>
```

We’ll need this for `strcmp`.

```c
struct blacklist_path {
    const char *name;
};

static const struct blacklist_path blacklist_path_table[] = {
    { "/etc/passwd" },
    { NULL }
};
```

I made a separate structure mostly in case we want to extend it later.

And yes, I know — I’m lazy. I used a `NULL` sentinel at the end of the array so I don’t have to manually track the array size.

```c
bool is_blacklisted(const char *path) {
    for (const struct blacklist_path *p = blacklist_path_table; p->name != NULL; p++) {
        if (strcmp(path, p->name) == 0){
            return 1;
        }
    }
    return 0;
}
```

This function compares the input string against our blacklist array.

Nothing fancy here:

* stop iterating once we hit `NULL`
* compare paths one by one
* return whether the path is blacklisted

```c
if (!entry) {
    fprintf(stderr, "syscall_%ld(0x%llx)", nr, regs.rdi);
} else if (entry->blacklist == 1) {
    read_child_string(child, regs.rsi, str, sizeof(str));
    if (is_blacklisted(str)) {
        fprintf(stderr, "%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
        fprintf(stderr, "BLOCKED");
        regs.rax = -1; 
        ptrace(PTRACE_SETREGS, child, 0, &regs);
        sandbox_blocked = 1;
    }
    else {
        read_child_string(child, regs.rsi, str, sizeof(str));
        fprintf(stderr, "%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
    }
```

And of course we plugged this into our condition.

Like I said, the code is pretty messy because we’re laser-focused on testing path filtering.

That’s why we directly read `rsi` without additional checks — because we already know that’s where `openat` stores the path argument.

We send that to our function, and now we only block the syscall if the specific path is blacklisted.

Let’s test it:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
...
openat(-100, "/etc/passwd")BLOCKED = 3
...
: Permission denied
```

Now we can see plenty of `openat` calls going through normally. Only the attempt to open `/etc/passwd` gets blocked.

Let’s try another file:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /tmp/test.txt
...
openat(-100, "/tmp/test.txt") = 3
...
Hello!!!!!
Hola!!!!!
```

Reading other files works perfectly fine.

> [!DANGER]
> But... there’s still a massive problem here... does TOCTOU ring a bell?

## TOCTOU

And now we hit the real problem.

Did we actually cut access using our tiny binary?

We successfully stop the syscall *if* the path matches our blacklist.

But this is exactly where race conditions — TOCTOU — destroy us.

Let me show you this output:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace ../exploit 2>/dev/null
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

I redirected file descriptor 2 into the void just to clean the output and expose a huge problem:

If access to `/etc/passwd` is blocked through our binary... how did we bypass it?

```c
#include <fcntl.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>

static char path[256] = "/tmp/whatever.txt";

void *flipper(void *arg) {
    for (;;) {
        memcpy(path, "/tmp/innocent.txt", 18);
        usleep(1);
        memcpy(path, "/etc/passwd", 12);
        usleep(1);
    }
    return NULL;
}

int main() {
    pthread_t t;
    pthread_create(&t, NULL, flipper, NULL);
    for (int i = 0; i < 10000; i++) {
        int fd = openat(AT_FDCWD, path, O_RDONLY);
        if (fd >= 0) {
            char buf[128];
            ssize_t n = read(fd, buf, sizeof(buf));
            if (n > 0) write(1, buf, n);
            close(fd);
        }
    }
    return 0;
}
```

Let’s break this down.

```c
static char path[256] = "/tmp/whatever.txt";
```

This is a shared buffer that any thread can read or modify.

```c
void *flipper(void *arg) {
    for (;;) {
        memcpy(path, "/tmp/whatever.txt", 18);
        usleep(1);
        memcpy(path, "/etc/passwd", 12);
        usleep(1);
    }
    return NULL;
}

int main() {
    pthread_t t;
    pthread_create(&t, NULL, flipper, NULL);
```

This infinite loop continuously swaps the buffer contents between an innocent file and `/etc/passwd`.

Then, once `main` starts, we create a thread running that flipper forever.

```c
for (int i = 0; i < 10000; i++) {
    int fd = openat(AT_FDCWD, path, O_RDONLY);
    if (fd >= 0) {
        char buf[128];
        ssize_t n = read(fd, buf, sizeof(buf));
        if (n > 0) write(1, buf, n);
        close(fd);
    }
}
```

Meanwhile, the main thread continuously calls `openat()` on the changing path buffer.

If it succeeds, it reads the file and prints the contents.

In short:

The exploit continuously attempts to read a forbidden path while rapidly swapping the filename between an allowed and forbidden target.

What’s actually happening is that it exploits the tiny timing window between:

1. our userspace check of the string
2. the moment the kernel actually resolves and opens the path

And this is the giant problem with trying to implement security this way using `ptrace`.

The window between:

* grabbing the registers
* reading child memory
* making a decision
* executing the syscall

...is only microseconds.

But that’s already enough.

And honestly, you don’t even need complicated code to exploit this. You can reproduce the same thing directly from the shell.

One terminal:

```bash
while true; do 
ln -sf /tmp/innocent.txt /tmp/toctou 
ln -sf /etc/passwd /tmp/toctou 
done
```

An infinite loop swapping a symlink between an allowed file and a forbidden one.

And another terminal:

```bash
while true; do 
./minitrace cat /tmp/toctou 2>/dev/null
done
```

Continuously attempting the read until it succeeds.

> [!DANGER]
> And it gets even worse...

There’s an even simpler bypass:

```bash
ln -sf /etc/passwd /tmp/toctou
./minitrace cat /tmp/toctou
```

Well then.

All that effort, and the whole thing collapses the moment you look at it too hard...

Now imagine this:

* far more code
* much deeper logic
* massively more complexity

...and suddenly it becomes obvious why ancient software keeps getting new CVEs every year.

We’re operating entirely in userspace here — specifically on what the process *tells* the kernel.

Underneath that, the kernel is doing tons of additional work:

* path normalization
* symlink resolution
* namespace handling
* descriptor resolution
* context tracking
* and much more

In fact, there are many more ways to bypass this protection, all based on the same fundamental principle.

And yes, we *could* try adding more checks and safeguards.

But the whole reason I wanted to show the TOCTOU issue is precisely because no matter how many safeguards we add, that race window will always exist.

So what’s the verdict?

Any security mechanism that separates the security decision into an external process is fundamentally vulnerable to this problem.

And this idea applies to far more than ptrace:

* sidecar validation
* userspace vs kernel checks
* client-side vs server-side validation
* and many other architectures

So in this particular case, we now know we *must* work directly in the kernel and use other mechanisms instead.

But that’s a topic for another day — it’s getting late.
