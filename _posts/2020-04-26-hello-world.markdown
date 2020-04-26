---
layout: post
title:  "Hello World and assembly"
date:   2020-04-26 08:59:37 +0100
---

Trying some new languages I was a bit surprised by the executable sizes
of "hello world" programs. I'm not that young anymore and I still remember
entire OSes of the size of hundred kilobytes, utilities included while
now even an hello world can be bigger than this!

I've never been an assembly developer but I was quite curious about and
I knew how to write assembly programs for x86 (well, in the time being
for hacking or study or curiosity I manage to get to touch x64, r3000,
mips, pa-risc and others but that's other stories).

Who needs assembly today? Do you ever:
- play video or audio from internet?
- access secure websites?
- manage photos from your camera/phone?
- do video editing?

I could go on and on with similar examples but believe it or not if
you are doing something of the above you are probably using some piece
of code written in assembly.

But back to the hello world! One day I was asking "how small can be
an hello world in a modern OS?". And obviously the language has to be
assembly!

So, the environment we are going to use here is Linux x64 using `nasm`
as assembler (I could use `gcc` but `nasm` uses Intel syntax and I liked
the idea to get to the origin).

The classic hello world in C, just to get some basic is something like

{% highlight c %}
#include <stdio.h>

int main(void) {
  printf("Hello, World!\n");
  return 0;
}
{% endhighlight %}

If we do an `strace` of the resulting executable we will see amongst the many
calls a call to `write` syscall to write to the output and a call to `exit_group`
to exit the process. These are enough for the hello world!

To understand how to do a syscall in Linux x64 you can read [asm_syscall].

The basic start was this:
- `Makefile`
{% highlight make %}
all: hello
        ./hello

hello: hello.asm Makefile
        nasm -f elf64 -o hello.o $<
        ld -s -o $@ hello.o

clean:
        rm -f hello.o hello
{% endhighlight %}
- `hello.asm`
{% highlight nasm %}
section .rodata
msg     db      "Hello, World!", 10

section .text
        global  _start
_start:
        mov     rax, 1 ; read
        mov     rdi, 1 ; fd
        mov     rsi, msg ; ptr
        mov     rdx, 14 ; len
        syscall

        mov     rax, 60 ; exit
        mov     rdi, 0
        syscall
{% endhighlight %}

Good, it runs correctly as expected. But wait... 8kb ?? There's something wrong!

To check the executable there are many useful commands:
- `ls -l hello` shows the size of the executable;
- `hexdump -C hello` shows the dump of the executable;
- `objdump -x hello` shows the sections of the executable;
- `objdump -d hello` shows the disassembly of the executable;
- `objdump -s hello` shows the data of the executable.

We can see (hexdump) that there are some ELF section, let's try to strip with `sstrip` changing the `Makefile`:

{% highlight make %}
all: hello
        ./hello

hello: hello.asm Makefile
        nasm -f elf64 -o hello.o $<
        ld -s -o $@ hello.o
        sstrip $@

clean:
        rm -f hello.o hello
{% endhighlight %}

A bit smaller but still 8kb.

Another thing we can note is that there are 2 sections, `.text` and `.rodata` which will be aligned to 4kb causing
the executable to be pretty big, but we can merge them changing the source `hello.asm` to
{% highlight nasm %}
section .text
msg     db      "Hello, World!", 10

        global  _start
_start:
        mov     rax, 1 ; read
        mov     rdi, 1 ; fd
        mov     rsi, msg ; ptr
        mov     rdx, 14 ; len
        syscall

        mov     rax, 60 ; exit
        mov     rdi, 0
        syscall
{% endhighlight %}

Now we got 4kb, better. But with `hexdump` we can see a big gap between program header and code. If you have
an old `ld` (linker) this is not happening but with an updated linker this is the case. To avoid the gap you
can use the `noseparate-code` linker option changing the `Makefile` to
{% highlight make %}
all: hello
        ./hello

hello: hello.asm Makefile
        nasm -f elf64 -o hello.o $<
        ld -s -z noseparate-code -o $@ hello.o
        sstrip $@

clean:
        rm -f hello.o hello
{% endhighlight %}

Now, you should get down to just 181 bytes. Nice, this is a more expected size for an assembly program.
Can we get further? Another thing from `hexdump` we can note is that there's still a bit gap between header
and code. This is due to section alignment, 8 bytes are enough, let's set in `hello.asm`

{% highlight nasm %}
section .text progbits alloc exec nowrite align=8
msg     db      "Hello, World!", 10

        global  _start
_start:
        mov     rax, 1 ; read
        mov     rdi, 1 ; fd
        mov     rsi, msg ; ptr
        mov     rdx, 14 ; len
        syscall

        mov     rax, 60 ; exit
        mov     rdi, 0
        syscall
{% endhighlight %}

Good, we are down to 173 bytes. I think trying to go further with linker and minor options is not that safe
(we could try doing some overlapping) so let's now focus on code. The `sstrip` change made `objdump` a bit useless,
but we can use assembly listing option to output the bytes of each generated instruction changing the `Makefile` to
{% highlight make %}
all: hello
        ./hello

hello: hello.asm Makefile
        nasm -f elf64 -o hello.o -l hello.l $<
        ld -s -z noseparate-code -o $@ hello.o
        sstrip $@

clean:
        rm -f hello.o hello.l hello
{% endhighlight %}

Which generates a

{% highlight plaintext %}
$ cat hello.l 
     1                                  section .text progbits alloc exec nowrite align=8
     2 00000000 48656C6C6F2C20576F-     msg     db      "Hello, World!", 10
     2 00000009 726C64210A         
     3                                  
     4                                          global  _start
     5                                  _start:
     6 0000000E B801000000                      mov     rax, 1 ; read
     7 00000013 BF01000000                      mov     rdi, 1 ; fd
     8 00000018 48BE-                           mov     rsi, msg ; ptr
     8 0000001A [0000000000000000] 
     9 00000022 BA0E000000                      mov     rdx, 14 ; len
    10 00000027 0F05                            syscall
    11                                  
    12 00000029 B83C000000                      mov     rax, 60 ; exit
    13 0000002E BF00000000                      mov     rdi, 0
    14 00000033 0F05                            syscall
{% endhighlight %}

Well, I knew already that simple `mov` to load constant can be not that good, usually to load zero you use `xor <reg> <reg>`
which takes only 2/3 bytes. Note that on x64 the 32bit registers get automatically extended to 64 bit writing to them
so, for instance, a `xor rdi, rdi` and a `xor edi, edi` produce the same result but the second takes 1 byte less.
Also you can note that loading `msg` address is pretty long, this as the constant can be huge, better for this to
use a relative addressing (I didn't have this issue in DOS/Windows in the past). Let's see what happens:

{% highlight nasm %}
section .text progbits alloc exec nowrite align=8
msg     db      "Hello, World!", 10

        global  _start
_start:
        mov     rax, 1 ; read
        mov     rdi, 1 ; fd
        lea     rsi, [rel msg] ; ptr
        mov     rdx, 14 ; len
        syscall

        mov     rax, 60 ; exit
        xor     edi, edi
        syscall
{% endhighlight %}

Good, down to 167 bytes. That's a good size and program is in a readable state.

Can we go a bit further? Well, `mov` for constants are still a bit big, we could do something about trying
to use values from other registers or using other tricks. I will show a possible result but I would
personally stop at the previous state, using other registers is going to decrease potentially speed,
you can do just to get the last bytes from it! A possible outcome can be something like this

{% highlight nasm %}
section .text progbits alloc exec nowrite align=8

        global  _start
_start:
        push    1 ; read
        pop     rax
        mov     edi, eax ; fd
        call    end_msg
msg     db      "Hello, World!", 10
end_msg:
        pop     rsi ; ptr
        lea     edx, [rdi+13] ; len
        syscall

        xor     edi, edi
        lea     eax, [rdi+60] ; exit
        syscall
{% endhighlight %}

Yes, this produces an executable of only 157 bytes (code and data fits in 37 bytes) but is not
much readable and these methods not really apply to real world code.

To sum up and return to the original problem. Why a simple hello world program got so big?
Surely some language have less system library code to reuse so its runtime is mostly statically
linked in the executable, surely even the base runtime has to handle some errors and report.
But still I'm convinced that along the way we lost some optimizations that was taking the
executable sizes down quite a lot.

[asm_syscall]: http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/
