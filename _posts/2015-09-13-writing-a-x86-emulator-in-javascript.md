---
layout: post
title: "Writing an x86 emulator in JavaScript"
date: 2015-09-13 13:21:35
image: '/assets/img/'
description:
tags:
- x86
- emulator
- javascript
categories:
- fun with binaries
twitter_text: 'Writing an x86 emulator in JavaScript'
---

*Disclaimer: this is not a proper tutorial, I have no prior experience with
emulators, the sample code is very simplistic, it doesn't emulate the actual
CPU and is far from optimal.*

---

As someone that doesn't have a CS background I've always wanted to properly
understand how things work at the lower level, and had decided to put more
effort into it.

As part of my learning path I got the book *Programming from the Ground Up* but
hadn't started yet, until I had an 11 hours flight to Brazil, and it felt like a
good opportunity to start.

I really liked the book, but it turns out the examples were written in Linux
x86 GNU assembly, and I was on a Mac OS X 64 bits... I struggled a little bit
bit between assembler and linker flags and syntax between `i386` and `x86_64`
but couldn't figure it without internet...

*(it turns out that for `i386` the return value for the exit syscall had to go
into the stack instead of into `%ebx`, for `x86_64` the syscall numbers have an
offset of `0x2000000`, different registers and you actually use `syscall` instead
of `int $0x80`... all these things are not very easy when you're first starting
with assembly)*

I tried leaving it aside, but couldn't stop thinking about it, so I started
writing a very dummy x86 parser, that was enough for the exercises I got to do
before running out of battery (nothing worse on a flight than no sockets).

A couple months later we'd have a hackathon at Facebook, on the Chelsea stadium,
and I thought it'd be cool to implement an x86 emulator, to understand how to
actually execute a binary.

Convincing people to work on this project wasn't very easy either, and it took
me some time to prove that I wasn't crazy, that I had a reasonable idea of how
hard it'd be to write a proper emulator, and to explain that I only wanted to
write it for `fun` and support only very basic use cases.

I got one coworker to work with me and our initial goal was to run the simplest
x86 program possible: just exit with code `0`:

{% highlight asm %}
# program.s

 .section __TEXT,__text
  .globl start
start:
  mov $0x1, %eax
  push $0x0
  call _syscall

_syscall:
  int $0x80
{% endhighlight %}

Here's how to build the above program in a Mac OS X:

{% highlight bash %}
$ as -static -arch i386 -o program.o program.s
$ ld -static -arch i386 -o program program.o
{% endhighlight %}

We split the problem in two: Finding the actual assembly instructions in the
binary and executing them.

To find the actual code, it's necessary to know the Mach-O binary architecture.
The binary contains diverse segments with different shapes:

![Mach-O Binary
image](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/MachORuntime/art/mach_o_segments.gif)

*Note: To have a better visualisation of the binary layout I recommend using
[MachOView](http://sourceforge.net/projects/machoview/), it's super helpful when
investigating the binary itself.*

We implemented a very simple loader, that worked only for non-fat binaries,
read only the number of load commands from the header, and actually used only
the segment commands and the unix thread.

The `LC_SEGMENT` load commands have a few fields

- Command (`LC_SEGMENT` in this case)
- Command Size
- Segment Name (e.g. \__TEXT)
- VM Address (Location into virtual memory where segment should be copied)
- VM Size (Size of virtual memory the segment will use)
- File Offset (Location of the segment in the binary)
- File Size (Size of segment in the binary)
- Maximum VM Protection (Maximum protection level allowed for the VM)
- Initial VM Protetction (Initial protection level for the VM)
- Number of Sections (Number of sections contained in the segment)
- Flags (The possible flags can be found in
  [mach-o/loader.h](http://www.opensource.apple.com/source/xnu/xnu-1456.1.26/EXTERNAL_HEADERS/mach-o/loader.h)
  right after the `struct segment_command`)

But we can ignore the VM protection and flags for now, and just copy the
contents of the binary from `File Offset` to `File Offset + File Size` into `VM
Address` (up to `VM Address + VM Size`).

The Unix Thread load command tells us the initial state of the program (the
initial value for the registers), the only thing we really care for the basic
example is the value of `%eip`, the instruction pointer, that is where the
program code actually begins (In the sample code we are actually using a global
variable called `PC` instead of an actual register).

OK, so now we have the program code mapped to virtual memory, and a pointer to
the beginning of the code, we `just` need to execute it. One thing that makes
it more fun is that x86 has variable size instructions, so we have to read the
opcode first in order to know how many bytes does it take.

![x86 Instruction
Format](http://securitydaily.net/wp-content/uploads/2015/01/cpu1.jpg)

In order to Keep It Simple™ I disassembled the generated binary with `objdump`,
a CLI that can be installed with the `binutils` package. It should look like:

{% highlight sh %}
$ brew install binutils # if you don't have it installed yet
$ gbojdump -d program

Disassembly of section .text:

00001ff2 <start>:
    1ff2:       b8 01 00 00 00          mov    $0x1,%eax
    1ff7:       6a 00                   push   $0x0
    1ff9:       e8 00 00 00 00          call   1ffe <_syscall>

00001ffe <\_syscall>:
    1ffe:       cd 80                   int    $0x80
{% endhighlight %}

Since I didn't intend to implement a fully functional emulator, I just looked
for what was the syntax for the necessary opcodes, a good reference for x86 can
be found at
[http://ref.x86asm.net/coder.html](http://ref.x86asm.net/coder.html).

I created a map (the implementation was actually a plain JavaScript object) from
opcodes to functions. The functions would be responsible to read more data if
needed by the opcode, e.g:

{% highlight javascript %}
var Functions = {};
Functions[0x6a] = () => {
  // Push one byte
  push(read(1));
};
{% endhighlight %}

For syscalls, we'd have to actually simulate it, so we'd have another map from
sycall numbers to functions:

{% highlight javascript %}
var Syscalls = {};
Syscalls[0x01] = () => {
  // Fake exit, since there's no OS
  console.log('Program returned %s', Stack[Registers[ESP + 1]]);
  PC = -1; // Mark the program as ended by setting the program counter to -1
};
{% endhighlight %}

In order to load the binary, we used the [File
API](https://developer.mozilla.org/en-US/docs/Web/API/File). A html page is the
entry point, with a single `input` to drop the binary and the output shows up in
the console.

The resulting code of the hackathon can be found in this
[gist](https://gist.github.com/tadeuzagallo/3853299f033bf9b746e4), contains 3
files as described: the loader (`mach-o.js`), the opcodes' logic (`x86.js`) and
the `index.html` that works as the entry point. The code can execute basic
assembly and C programs.

*NOTE:* it cannot execute libc, so C programs have to be compiled with
`-static -nostdlib` and provide a custom assembly boostrap.

As mentioned above the code is super simple, was completely written in a
hackathon, so there wasn't much effort into making it very readable (or very
good for that matter) and we didn't iterate any further on that.

It was enough to run fibonacci in C compiled with `-O3`, but it was unsuccessful
when trying to benchmark `fibonacci(40)`, it'd just take forever... :(
