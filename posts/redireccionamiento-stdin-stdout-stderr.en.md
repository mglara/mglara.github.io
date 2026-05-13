# FD Redirection

We already know Linux file descriptors, so let’s have a bit of fun with them live.

| Column 1 | Column 2 |
| -------- | -------- |
| 0        | STDIN    |
| 1        | STDOUT   |
| 2        | STDERR   |

In Linux, everything is a file, and communication with the outside world happens through file descriptors. Let’s take a look from the terminal using a process with `PID = 213408`.

As I mentioned, everything is a file, and all the information we need about that process lives under `/proc/PID`. More specifically, we care about `/proc/PID/fd`, because that’s where the file descriptors are.

```bash
miguel@demo-ubuntu:~/tmp/std$ ls -lah /proc/213408/fd
total 0
dr-x------ 2 miguel miguel  4 may 12 17:05 .
dr-xr-xr-x 9 miguel miguel  0 may 12 17:05 ..
lrwx------ 1 miguel miguel 64 may 12 17:05 0 -> /dev/pts/15
lrwx------ 1 miguel miguel 64 may 12 17:08 1 -> /dev/pts/15
lrwx------ 1 miguel miguel 64 may 12 17:08 2 -> /dev/pts/15
lr-x------ 1 miguel miguel 64 may 12 17:08 255 -> /home/miguel/tmp/std/test.sh
```

Let’s ignore `255` for a moment, but notice how `0`, `1`, and `2` — STDIN, STDOUT, and STDERR — all have a sort of symbolic link to `/dev/pts/15`.

What exactly is that?

We can jump into any terminal and run:

```bash
miguel@demo-ubuntu:~/tmp/std$ tty
/dev/pts/19
```

Aaaah, right. Everything is a file, remember?

Now it starts making sense. Looks like this process was launched from terminal `tty 15`, or at least everything related to it is tied to that terminal.

It’s possible to redirect where stdin, stdout, and stderr point to. But to do that, we can’t just edit the symbolic link directly — we need to use some more advanced tricks.

If you’re already lost and have no idea what we’re talking about, I’d recommend reading <a href="/stdin-stdout-stderr">the post about stdin-stdout-stderr</a> first.

Like I said, we can’t simply repoint the symbolic link to another terminal:

```bash
miguel@demo-ubuntu:~/tmp/std$ ln -sf /proc/213408/fd/1 /dev/pts/17
ln: failed to create symbolic link '/dev/pts/17': Permission denied
miguel@demo-ubuntu:~/tmp/std$ sudo ln -sf /proc/213408/fd/1 /dev/pts/17
ln: failed to create symbolic link '/dev/pts/17': Operation not permitted
```

But we *can* do it another way...

---

## Redirecting with GDB

Using GDB, we can pull this off like this.

First, we need the PID of the process whose redirection we want to change. As an example, here’s the following script:

```bash
#!/bin/bash
echo "This is the pid: $$"
echo "This is stdout:"; sudo ls -l /proc/$$/fd/0;
n=0; while true; do ((n++)); echo $n; sleep 1; done
```

This script simply prints its PID, shows the standard input, and then enters an infinite loop counting numbers every second.

In fact, the process we were looking at earlier is exactly this script.

To modify the output of this script, we grab its PID and open GDB like this:

```bash
# open gdb
miguel@demo-ubuntu:~/tmp/std$ sudo gdb

# First attach to the process. Once attached, the process pauses
# and lets us execute code in its address space.
attach PID

# Next, open the target terminal in read/write mode
call open("/dev/pts/17", 66, 0666)

# This returns a number which we'll use next.
# The first argument is the source FD, the second one is 1,
# meaning stdout.
call dup2(3, 1)

# Finally, let it continue and quit
detach
quit
```

Here’s a quick video showing it in action:

<localvideo src="/redireccionamiento-stdin-stdout-stderr/TerminalChange2.mp4" />

Now that we know this is possible, we could build our own little tool to automate it, or something similar.

But today is not that day 😄

That said, before wrapping up, let’s build something slightly cooler. I’m not leaving you hanging halfway through. What each of you decides to do with this knowledge afterward... I don’t need to know 😈

---

## Terminal Mirroring

Once you understand how this works and combine it with a few extra concepts, we can build a mirror — meaning we can watch, in real time, what’s happening in another terminal from our own terminal.

Maybe two people logged into the same machine with the same user?...

A simple `dup` isn’t enough here, because once we redirect stdout we lose the original reference, meaning we can’t send output to two terminals at the same time. STDOUT is unique.

Creating a regular pipe wouldn’t really help either. Not because it’s theoretically impossible, but because we’d need a previously allocated memory pointer.

So let’s think a bit differently...

What if we use a FIFO?

A FIFO is basically a pipe, but with its own name — and we already know that naming things gives them superpowers.

A regular pipe can only be used between related processes, while a FIFO persists in the filesystem and can be used by unrelated processes too.

Creating one is easy:

```bash
miguel@demo-ubuntu:~/tmp/std$ mkfifo /tmp/mirror
miguel@demo-ubuntu:~/tmp/std$ ls -lh /tmp/mirror
prw-rw-r-- 1 miguel miguel 0 may 12 19:37 /tmp/mirror
```

Perfect. With that ready, we jump back into GDB:

```bash
(gdb) call open("/tmp/mirror", 1, 0)
$4 = 4
(gdb) call dup2(4, 1)  
```

Okay, I’m cheating a little bit here. There was an annoying issue in the middle...

When we run `call open("/tmp/mirror_pipe", 1, 0)`, that `1` means write-only mode.

But opening a FIFO in write-only mode blocks the call until another process opens the FIFO for reading.

A quick fix is to already have something listening on it, open it in read/write mode, or — if you already had everything running like I did — simply do a `cat /tmp/mirror` and problem solved.

Now we only need the final piece: using that FIFO to display the output on two terminals simultaneously.

For that we can use `tee`, a binary specifically designed for this:

```bash
cat /tmp/mirror | tee /dev/pts/15 /dev/pts/17
```

Here’s the video, trimming a few steps since you already saw the full process earlier:

<localvideo src="/redireccionamiento-stdin-stdout-stderr/Mirrorpipe.mp4" />

Beyond the huge amount of things you could do with this information when used properly, at the end of the day we’re just learning a bit more about the inner workings of the wonderful world of Linux.
