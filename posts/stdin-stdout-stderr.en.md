# What Are STDIN, STDOUT, and STDERR?

When we talk about these concepts, we’re referring to the standard data streams that handle how programs receive input and produce output.

## STDIN

**STDIN**, or standard input, is the stream a program uses to receive data. By default, that data comes from the keyboard, but we can also redirect input from a file or other sources.

For example, if we create a file called `commands` with the following content:

```bash
echo "echo hello" > commands
echo "netstat -tonap" >> commands
echo "exit" >> commands
```

We can launch a shell using the contents of that file as STDIN:

```bash
$ sh < ./commands

hello
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0    392 192.168.0.22:22         192.168.0.11:54945      ESTABLISHED -                    on (0,04/0/0)
tcp6       0      0 :::139                  :::*                    LISTEN      -                    off (0.00/0/0)
tcp6       0      0 :::22                   :::*                    LISTEN      -                    off (0.00/0/0)
tcp6       0      0 :::445                  :::*                    LISTEN      -                    off (0.00/0/0)
```

As you can see, we managed to make the program read its STDIN directly from the file we created.

## STDOUT

STDOUT, or standard output, is where a program sends its results. By default, this output is displayed in the terminal, but it can also be redirected elsewhere. If you’ve worked with Linux before, chances are you’ve already used redirections without even realizing what you were doing.

For example, if we want to run a ping and save the results to a file, we can do this:

```bash
$ ping -c 3 127.0.0.1 1> result
$ cat result

PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.035 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.037 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.022 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2037ms
rtt min/avg/max/mdev = 0.022/0.031/0.037/0.006 ms
```

To redirect the STDOUT stream, we commonly use `1>` or simply `>`.

## STDERR

**STDERR**, or standard error, is the stream used to handle a program’s error messages. By default, these messages are also sent to the terminal, but they can be redirected elsewhere too.

> [!WARNING]
> Even though STDERR is technically the standard error output, be careful with that definition because it’s often used for much more than just errors: diagnostic messages, status updates, warnings, etc. Many binaries take advantage of this to keep pipes `|` and output redirections `>` clean.

To redirect STDERR, we use `2>`. For example:

```bash
$ cat idontexist 2> result
$ cat result

cat: idontexist: No such file or directory
```

In fact, we can redirect both outputs — STDOUT and STDERR — to different places:

```bash
./myprogram.sh 1> stdoutResults 2> stderrResults

# Or even to the same place
./myprogram.sh > output 2>&1
```

In the last example, `2>&1` means STDERR is being redirected to `output`. What you’re really saying is that you want STDERR (`2>`) redirected to file descriptor `1`, which is why the `&` operator is used.

| Descriptor | Output |
| ---------: | :----- |
|          0 | STDIN  |
|          1 | STDOUT |
|          2 | STDERR |

Something that has always caught my attention is how some console-based programs, especially on Linux, send part of their regular program information through STDERR instead of STDOUT — even when those messages aren’t actually errors.

That way, if you redirect STDOUT, that extra information doesn’t end up mixed into the final output, keeping things much cleaner.

In fact, we can build our own workflow in code to do exactly this. Let’s say I have a program that performs a port scan, but besides the actual scan results, it also prints some extra information — call it DEBUG info or just progress updates so we can see what the program is doing.

```bash
#!/bin/bash

# Handle interruption signal (Ctrl+C)
trap ctrl_c INT

# Function executed on interruption
function ctrl_c() {
    echo -e "\n\n[*] CANCELLING... [*]\n" >&2
    exit 0
}

# Check whether an argument was provided (IP or domain)
if [ -z "$1" ]; then
    echo "Usage: $0 <IP or domain>" >&2
    exit 1
fi

# Scan startup info
echo "[*] Starting port scan against $1... [*]" >&2

# Port scan
for port in $(seq 0 65535); do
    echo "[*] Scanning port $port... [*]" >&2
    timeout 1 bash -c "echo '' < /dev/tcp/$1/$port" 2>/dev/null && echo "$port;UP" &
done

# Wait for all scan processes to finish
wait

# Scan finished info
echo "[*] Scan completed. [*]" >&2
```

Let’s take a look at the output from the script above:

```bash
$ ./scan.sh 127.0.0.1
[*] Starting port scan against 127.0.0.1... [*]
[*] Scanning port 0... [*]
[*] Scanning port 1... [*]
[*] Scanning port 2... [*]
[*] Scanning port 3... [*]
...
[*] Scanning port 50... [*]
22;UP
```

That’s way too much information. Ideally, the script would overwrite the previous line instead of spamming the terminal so much, but remember, we’re doing this for demo purposes 😜.

Now, if we launch the scan like this:

```bash
$ ./scan.sh 127.0.0.1 > ports 2> debug
```

And then check both `ports` and `debug`:

```bash
$ cat ports

22;UP

139;UP

445;UP

$ cat debug
[*] Starting port scan against 127.0.0.1... [*]
[*] Scanning port 0... [*]
[*] Scanning port 1... [*]
[*] Scanning port 2... [*]
[*] Scanning port 3... [*]
...
```

Exactly — you’ve probably already noticed what happened. Since we explicitly decided what should count as standard output and what should belong to the error stream, we can keep the final output clean without needing to implement a separate data storage section inside our little script. As I said before, this technique is widely used in many console applications.

## Conclusion

Understanding how standard data streams work makes it much easier to use commands in all kinds of situations, especially when dealing with errors. This article barely scratches the surface — there’s a lot more to explore.

In fact, I’d recommend checking out [File Descriptor Redirection](/blog/redireccionamiento-stdin-stdout-stderr) if you want to dive a little deeper into these concepts.
