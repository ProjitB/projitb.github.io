# Self Modifying Code

Due to the von-neuman architecture (basically what all our processors use), code and data are indistinguishable. A pretty great result of this is that you can technically intersperse code with data, and vice versa. What does that mean? It means that what would normally be an input to your program, can actually _be_ the program itself.

The concept of injecting code isn't new. But the way we'll do it here is actually quite interesting (not very unique though). I found seeing how this worked at an assembly level, helped me understand how things like JIT compilation and meta-programming would be implemented under the hood. Assuming you can write code, which actually modifies the code it will execute next (or even later), it doesn't seem like a far stretch to add a few conditions to generate new optimized code at runtime (essentially JIT compilation at a very high level).


Contents:
- [Tools](#tools)
- [Beyond Root](#beyond-root)
- [No Sharing!](#no-sharing)
- [Sometimes Sharing is Good?](#sometimes-sharing-is-good)
- [Building that weird tar](#build-that-weird-tar)
- [Buildroot: Make your own custom container](#myol)


## Tools
Quickstart on vagrant setup for this:
```
> vagrant init ubuntu/bionic64
> vagrant up
> vagrant ssh
```

Couple of linux tools that we'll be using (most come with major distros anyway):
 - gcc
 - objcopy
 - hd (hexdump)


## Basic Assembly Code
So for this next part, what we'll do is write a bit of assembly to read and print out a file. Pretty simple right?
``` bash
> sudo vi /file1
> chmod 777 /file1
```

Even if you're not too comfortable with assembly, this should be decently understandable. All we'll be doing is moving some values to appropriate registers and invoke syscalls.
I'm not too great with assembly, but here's a shot at it:
``` assembly
.data
_somefile:
    .string "/file1"
.text
.global _start

_start:
.intel_syntax noprefix
        push 2
        pop rax
        push _somefile
        mov rdi, rsp
        mov rsi, 0
        mov rdx, 0
        call _sysc

sendfile:
        mov rdi, 1
        mov rsi, rax
        mov rdx, 0
        mov r10, 100
        mov rax, 40
        call _sysc
closefd:
        mov rdi, rsi
        mov rax, 3
        call _sysc
exit:
        mov rax, 60
        call _sysc
_sysc:
        syscall
        ret
```
All we did here was open up file `/file1`, use the sendfile syscall to send 100 bytes of content to stdout (fd 1), and then some cleanup. We'll see in a bit why I didn't directly invoke syscall and instead put it as part of a function call. Let's save the above in a file called `normal.s`.
A great resource on syscall usage is [1](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/).

Let's now generate an ELF (executable linkable format) file from this, so that we can actually run it.

``` bash
> gcc -static -nostdlib normal.s -o normal-elf
> ./normal-elf
hello this is file1
```

The 'code' part of an ELF file is in the text section. This can be extracted by:
``` bash
> objcopy --dump-section .text=normal.raw normal-elf
```
Running `cat` on the file will yield some gibberish. However, if we do a hexdump of it, we can start looking at essentially what the computer sees of this file.
```bash
> hd normal.raw
00000000  6a 02 58 ff 34 25 36 01  40 00 48 89 e7 48 c7 c6  |j.X.4%6.@.H..H..|
00000010  00 00 00 00 48 c7 c2 00  00 00 00 e8 3f 00 00 00  |....H.......?...|
00000020  48 c7 c7 01 00 00 00 48  89 c6 48 c7 c2 00 00 00  |H......H..H.....|
00000030  00 49 c7 c2 64 00 00 00  48 c7 c0 28 00 00 00 e8  |.I..d...H..(....|
00000040  1b 00 00 00 48 89 f7 48  c7 c0 03 00 00 00 e8 0c  |....H..H........|
00000050  00 00 00 48 c7 c0 3c 00  00 00 e8 00 00 00 00 0f  |...H..<.........|
00000060  05 c3                                             |..|
00000062
```

The thing to note here is that all the instructions we wrote in assembly, have been converted into bytes. Let's use this to reverse engineer a bit (cause who has time to look through documentation right?)

## Start deleting stuff
What if we want to find out which bytes correspond to the syscall instruction? An easy way to figure it out would be to just remove that instruction, and then diff the hexdumps (this is why we kept syscall as part of a function call :) ).
Simply commenting out syscall, and following the previous steps (`gcc`, `objcopy`, `hd`).. we get:
```
> gcc -static -nostdlib nosyscall.s -o nosyscall-elf
> objcopy --dump-section .text=nosyscall.raw nosyscall-elf
> hd nosyscall.raw
00000000  6a 02 58 ff 34 25 34 01  40 00 48 89 e7 48 c7 c6  |j.X.4%4.@.H..H..|
00000010  00 00 00 00 48 c7 c2 00  00 00 00 e8 3f 00 00 00  |....H.......?...|
00000020  48 c7 c7 01 00 00 00 48  89 c6 48 c7 c2 00 00 00  |H......H..H.....|
00000030  00 49 c7 c2 64 00 00 00  48 c7 c0 28 00 00 00 e8  |.I..d...H..(....|
00000040  1b 00 00 00 48 89 f7 48  c7 c0 03 00 00 00 e8 0c  |....H..H........|
00000050  00 00 00 48 c7 c0 3c 00  00 00 e8 00 00 00 00 c3  |...H..<.........|
00000060
```

A quick diff shows us that the bytes corresponding to the `syscall` instruction are: `0f 05`. But so what? How is this useful to us?

Now let's go back to our code, and replace the syscall instruction with these bytes
The part that you'd change is:
```
// before
// _sysc:
//    syscall
//    ret

_sysc:
    .byte 0x0f, 0x05
    ret
```

Running `gcc` you'll see that this generates an ELF file identical to that which uses the syscall instruction itself (can do `hd` as well to confirm). This is expected though right, after all instructions are just a small semantic abstraction.
We can go on figuring out the instruction-byte mapping by doing this a couple times (or, you know, go read documentation), but more than a single instance isn't really required for our purposes now.

## Modifying code
Once in bytes form, code is indistinguishable from data. Which also means that we should be able to modify our code right? The instruction to be executed is stored somewhere, and if our code can modify that, then we have self-modifying code.

Normally, there are restrictions on making the `text` portion of the ELF file writable. These can be bypassed by compiling using a few additional flags (the flags are actually passed to the linker):
```
> gcc -static -nostdlib -Wl,-N modifying.s -o modifying-elf
```

What's left then? Nothing.
Say that instead of writing `.byte 0x0f, 0x05`, we wrote `.byte 0x00. 0x05`. Try it.
Trying to run it, you'll get a segmentation fault. Clearly this code doesn't work right? Now let's add some code, to modify this line, while the program runs. Somewhere at the top, add: `mov byte ptr [rip+_sysc], 0x0f`.
For clarity our file now looks like this:
``` assembly
.data
_somefile:
    .string "/file1"
.text
.global _start

_start:
.intel_syntax noprefix
        push 2
        pop rax
        // Here's the runtime modification
        mov byte ptr [rip+_sysc], 0x0f  
        push _somefile
        mov rdi, rsp
        mov rsi, 0
        mov rdx, 0
        call _sysc

sendfile:
        mov rdi, 1
        mov rsi, rax
        mov rdx, 0
        mov r10, 100
        mov rax, 40
        call _sysc
closefd:
        mov rdi, rsi
        mov rax, 3
        call _sysc
exit:
        mov rax, 60
        call _sysc
_sysc:
        .byte 0x00, 0x05
        ret
```
Compile (using the aformentioned flags), and run it. No more seg fault :)
In fact, go a step further and again analyze the hexdump of the file. You'll see your bytes: 0x00 and 0x05. This implies they were definitely part of your program...and yet they never ran because they were modified.


## Conclusions
What we've just seen here is the basis for larger concepts like metaprogramming and JIT compilation. At runtime, you can convert what is 'data' into 'code'. Implementing some naiive JIT optimizations doesn't seem too difficult either then. All you would need to do is add a few lines to check for some continuation (ex. check if a loop was hit 100 times), after which you could overwrite the main branch by moving in new assembly.



## References:
- [1] [https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
