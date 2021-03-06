---
title: "The Many Meanings of the Dot"
date: 2018-04-01T16:52:28-04:00
draft: false
---

# The Many Meanings of the Dot
That little `.` character sure gets a lot of use on our keyboard. Naturally, in
prose as a full stop, but I'm talking about in _software_. www.google.com. `cd
..`. The list goes on. Behind each seemingly innocuous dot, hides some pretty
fun abstractions. Let's dive into a few:

## In file paths
If you've spent any time in a shell, you've probably used `.` or `..` to
reference the current and parent directory, respectively. It's a handy shortcut!
How and why it works under the hood is more subtle.

To begin understanding the dot in relation to file paths we need to introduce
the two different types of paths: absolute and relative. An __absolute path__
begins with a `/`, the root directory. A __relative path__ is any path that does
not begin with `/`.  It is relative to your current directory &mdash; more on
that later. All relative paths can be 'resolved' to an absolute path.

For example, `/tmp` is an absolute path. If your current directory was `/`, you
could also reference the same directory as `tmp`, `./tmp`, or even
`./././././tmp`, as `.` in this context will always refer to the current
directory.

But what provides this concept of a 'current directory'? The shell? The file
system? Turns out it's the kernel. Each process has associated with it a working
directory that can change throughout the life of the process (see [`man 2
getcwd`][getcwd] / [`man 2 chdir`][chdir]). Since the kernel tracks a processes'
working directory, we hypothesize it also provides the `.` and `..` abstractions
for navigating file hierarchies. A "quick" read through the [POSIX
standard][filesystem-dot] confirms this:

> The special filename dot shall refer to the directory specified by its
> predecessor. The special filename dot-dot shall refer to the parent directory
> of its predecessor directory. As a special case, in the root directory,
> dot-dot may refer to the root directory itself.

An interesting property of this functionality being built in at such a low level
is how it simplifies programs and scripts. When writing code, you don't
necessarily need to know (though you can always find out) where in the
filesystem you are &mdash; a relative path is often just as good as an absolute
path.

Aside: another common (but distinct) usage of the dot in file paths is to hide
files whose name begins with a `.`. I'll leave this one for you. Is there
anything special about so-called dotfiles?

## Shells
In POSIX-compliant shells (think bash, zsh, etc.), the command `.` followed by a
filename executes commands from a file in the current shell.

Don't take my word for it:

```
$ help .
.: . filename [arguments]
Execute commands from a file in the current shell.

Read and execute commands from FILENAME in the current shell.  The
entries in $PATH are used to find the directory containing FILENAME.
If any ARGUMENTS are supplied, they become the positional parameters
when FILENAME is executed.

Exit Status:
Returns the status of the last command executed in FILENAME; fails if
FILENAME cannot be read.
```

Note that we have to use `help` here, not `man` because the `.` is a bash
[builtin][], meaning it's literally built into bash, which itself is [just a C
program][bash].

Normally, when a shell executes an external program it will do so in a
__subprocess__. In order to execute a new command, the shell first asks the
kernel to create a copy of itself using something like the `fork` system call
([`man 2 fork`][fork]). This copy is identical, except for its process ID
(PID). The new subprocess then replaces itself with the program you supplied
([`man 2 execve`][execve]). This "program" is just a file (in *nix systems,
everything is just a file) that has its executable permission bit set (if you've
ever done a `chmod +x`).

Using the `.` (or, in many shells, its equivalent command `source`) skips
fork-exec and instead reads commands from a file directly into your shell as if
you were typing them at an interactive prompt. Somewhat counterintuitively,
however, any programs themselves will still be executed in a subprocess; if you
want an external program to replace the current shell there's a builtin for
that, too (see `help exec`).

Let's break this down with an example. There's one more thing to understand,
first: also builtin to bash are a few [special variables][special-param] that we
can reference. One of which, `$$`, will tell us our current PID.

Assume I'm in a bash shell with PID 76735:

```
$ echo $$
76735
```

Take this simple script:

```
$ cat /tmp/getmypid
#!/bin/bash

echo "My PID is $$"
echo "My parent's PID is $PPID"
```

If we run it normally as an external program you'll notice that the PID is new
(a subprocess) but the parent's PID is in fact our shell:

```
$ chmod +x /tmp/getmypid
$ /tmp/getmypid
My PID is 77128
My parent's PID is 76735
```

Now notice the distinction when prefaced with a `.`:

```
$ . /tmp/getmypid
My PID is 76735
My parent's PID is 75379
```

The script is being executed in the shell and the parent PID is that of whatever
process spawned the shell.

A few asides:

- You may also find it interesting to experiment with how the above output
  changes after running `set -x` in your shell. Not sure what that does? It's a
  builtin; you can find out by running `help set`. If you have a complex prompt
  setup, consider dropping into a new shell without your customizations using
  `bash --norc` first to limit the output of `set -x`.
- `echo` is a builtin, but it also has an executable counterpart,
  `/bin/echo`. In this particular example it doesn't matter which echo we're
  using.
- The file you are trying to source needs to be a plaintext shell script (a
  set of commands that you'd type at an interactive shell), as opposed to any
  old executable file.
- The first line of our script is a [shebang][], which tells `execve` how we'd
  like our program to be run; in this case it's as a bash script.

## Networks
DNS is the protocol responsible for translating the human-readable
[www.google.com](https://www.google.com) into an IP address. IP addresses (like
74.125.141.147) are essential to routing requests across the Internet. Both DNS
and IPv4 use the dot as a separator, but each in a unique way.

### DNS
In DNS, even though you don't see it, fully qualified domain names actually end
with a dot. `www.google.com` is actually `www.google.com.`. The reason for this
is that DNS resolution works recursively. The authoritative answer for

> What is www.google.com's IP address?

actually comes from answering a series of questions generated by traversing the
domain name from right to left.

The rightmost `.` represents the [DNS root zone][root] &mdash; yep, that's a
thing! The root zone knows the [nameservers][] for all top level domains (TLDs)
like com, net, org, etc. Each TLD's nameservers in turn resolve another level
down, so on and so forth.

This is a bit of an oversimplification, but this article is on the dot,
not DNS. Let's skip ahead to the action:

```
$ dig +trace www.google.com
; <<>> DiG 9.8.3-P1 <<>> +trace www.google.com
;; global options: +cmd
.                       29875   IN      NS      d.root-servers.net.
.                       29875   IN      NS      f.root-servers.net.
.                       29875   IN      NS      g.root-servers.net.
.                       29875   IN      NS      l.root-servers.net.
.                       29875   IN      NS      j.root-servers.net.
.                       29875   IN      NS      a.root-servers.net.
.                       29875   IN      NS      h.root-servers.net.
.                       29875   IN      NS      m.root-servers.net.
.                       29875   IN      NS      c.root-servers.net.
.                       29875   IN      NS      e.root-servers.net.
.                       29875   IN      NS      i.root-servers.net.
.                       29875   IN      NS      b.root-servers.net.
.                       29875   IN      NS      k.root-servers.net.
;; Received 228 bytes from 192.168.1.1#53(192.168.1.1) in 26 ms

com.                    172800  IN      NS      a.gtld-servers.net.
com.                    172800  IN      NS      b.gtld-servers.net.
com.                    172800  IN      NS      c.gtld-servers.net.
com.                    172800  IN      NS      d.gtld-servers.net.
com.                    172800  IN      NS      e.gtld-servers.net.
com.                    172800  IN      NS      f.gtld-servers.net.
com.                    172800  IN      NS      g.gtld-servers.net.
com.                    172800  IN      NS      h.gtld-servers.net.
com.                    172800  IN      NS      i.gtld-servers.net.
com.                    172800  IN      NS      j.gtld-servers.net.
com.                    172800  IN      NS      k.gtld-servers.net.
com.                    172800  IN      NS      l.gtld-servers.net.
com.                    172800  IN      NS      m.gtld-servers.net.
;; Received 492 bytes from 199.7.83.42#53(199.7.83.42) in 49 ms

google.com.             172800  IN      NS      ns2.google.com.
google.com.             172800  IN      NS      ns1.google.com.
google.com.             172800  IN      NS      ns3.google.com.
google.com.             172800  IN      NS      ns4.google.com.
;; Received 280 bytes from 192.41.162.30#53(192.41.162.30) in 188 ms

www.google.com.         300     IN      A       74.125.141.104
www.google.com.         300     IN      A       74.125.141.106
www.google.com.         300     IN      A       74.125.141.99
www.google.com.         300     IN      A       74.125.141.103
www.google.com.         300     IN      A       74.125.141.105
www.google.com.         300     IN      A       74.125.141.147
;; Received 128 bytes from 216.239.36.10#53(216.239.36.10) in 52 ms
```

Notice how the path gradually builds up from `.` to `com.` to `google.com.` and
finally to `www.google.com.`. We now have an IP address (actually, 6 IP
addresses) for [www.google.com](https://www.google.com), which contains more
dots.

These dots are, you guessed it, completely different from any we've seen thus
far.

### IPv4
Unlike DNS, the Internet Protocol (IP) doesn't work recursively. Each IPv4
address is just a 32 bit number composed of four 8-bit octets. However, you
usually see an IP represented as a set of four base-10 numbers ranging from 0 to
255, separated by dots. We'll be using __74.125.141.147__, one of Google's IPs
from above as our example.

The actual IP address that gets sent over the wire is the binary representation
of these octets. In fact, we can represent an IP address any number of ways.
Let's hop into a Python interpreter to do some quick conversions:

```
$ python3
>>> ip_as_octets = '74.125.141.147'.split('.')
['74', '125', '141', '147']
>>> as_binary_list = ['{:08b}'.format(int(octet)) for octet in ip_as_octets]
['01001010', '01111101', '10001101', '10010011']
>>> as_binary = ''.join(as_binary_list)
'01001010011111011000110110010011'
>>> len(as_binary)
32
>>> as_decimal = int(as_binary, 2)
1249742227
>>> as_hex = hex(as_decimal)
'0x4a7d8d93'
```

So, 74.125.141.147 is the same as 0b01001010011111011000110110010011 in binary,
1249742227 in decimal, and 0x4a7d8d93 in hexadecimal.

Aside: I noticed when writing this post that decimal and hexadecimal IPs are
actually valid for anything that uses [`inet_addr`][inet_addr] &mdash; try
passing one into `ping`, `curl`, or even your web browser. Nifty!

## Recap
While we've covered a lot of ground, we also definitely have not hit everything;
take [regular expressions][regexp] as another example.

Why the dot has been adopted and used in so many different ways? Maybe because
it doesn't draw attention to itself? Because of its use in language? Its
position on the keyboard? I have no idea.

[filesystem-dot]: http://pubs.opengroup.org/onlinepubs/009604499/basedefs/xbd_chap04.html#tag_04_11
[inode]: https://en.wikipedia.org/wiki/Inode
[builtin]: https://www.tldp.org/LDP/abs/html/internal.html#BUILTINREF
[bash]: https://ftp.gnu.org/gnu/bash/
[shebang]: https://en.wikipedia.org/wiki/Shebang_(Unix)
[shell-dot]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_18
[fork-exec]: https://en.wikipedia.org/wiki/Fork%E2%80%93exec
[special-param]: https://www.gnu.org/software/bash/manual/html_node/Special-Parameters.html#Special-Parameters
[pid]: https://en.wikipedia.org/wiki/Process_identifier
[root]: https://en.wikipedia.org/wiki/DNS_root_zone
[fork]: https://linux.die.net/man/2/fork
[execve]: https://linux.die.net/man/2/execve
[nameservers]: https://en.wikipedia.org/wiki/Name_server#Domain_Name_System
[regexp]: https://regexone.com/lesson/wildcards_dot
[syscall]: https://en.wikipedia.org/wiki/System_call
[chdir]: https://linux.die.net/man/2/chdir
[getcwd]: https://linux.die.net/man/2/getcwd
[inet_addr]: https://linux.die.net/man/3/inet_addr
