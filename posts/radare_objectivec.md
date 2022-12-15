---
title : "Radare2 and iOS Apps"
description : "Analyzing the iOS applications using radare2"
date : 2021-04-14T20:46:23+02:00
---

## Introduction

I have always wanted to use [radare2](https://github.com/radareorg/radare2) for reverse engineering. For those unfamiliar with radare2 it is basically like IDA or Ghidra but more cool. And for the debugging and analyzing applications, I have used all kinds of different tools but with r2 it feels like I have it all there needed.

Per Wikipedia page:
> Radare2 (also known as r2) is a complete framework for reverse-engineering and analyzing binaries; composed of a set of small utilities that can be used together or independently from the command line.

I will be doing the exact some analysis like in [Jailbreak Bypass](https://lateralusd.github.io/jailbreak_bypass/) post but will do everything with r2 except for signing part.

## Loading the binary

Extract your binary from the ipa file and pass it to the ```r2``` binary.
```bash
$ r2 original_binary
 -- radare2 is power, France is pancake.
[0x100005490]>
```

## Interface and method analysis
We will approach this in two ways:
* Getting straight to the class
* Getting the class name from the method name

### Getting straight to the class
Since we are now in the r2, the `i` command is the main information command, and if we type `i?` we will see further commands.

So let's type `i?` and see if there is anything classes related using the `~ class` which is equivalent to ` | grep class` inside the shell.

```
[0x100005490]> i? ~ class
| ic                 List classes, methods and fields
| icc                List classes, methods and fields in Header Format
| icg                List classes as agn/age commands to create class hirearchy graphs
| icq                List classes, in quiet mode (just the classname)
| icqq               List classes, in quieter mode (only show non-system classnames)
```

So we can see that we have have the `ic` command which lists classes, methods and fields. Let's use it and grep for our `RootCheckResult`.

```
[0x100005490]> ic ~ RootCheckResult
0x100469788 [0x100005bc8 - 0x100005c48]    128 class 0 RootCheckResult :: NSObject
```

Now, if we pass just the name of the class to the ```ic``` we will get it's methods.

```
[0x100005490]> ic RootCheckResult
class RootCheckResult
0x100005bc8 method RootCheckResult      init
0x100005c24 method RootCheckResult      isRooted
0x100005c2c method RootCheckResult      setIsRooted:
0x100005c34 method RootCheckResult      rootCheckFailed
0x100005c3c method RootCheckResult      setRootCheckFailed:
0x100005c48 method RootCheckResult      .cxx_destruct
```

We can see that our ```isRooted``` method is indeed inside of RootCheckResult class.

### Getting the class name from the method name

In this approach we will do the same as we did in the original post, by trying to find the method responsible for root check. To shorten our time, let's say we now that the method ```isRooted``` is the one we are after. So let's see how to that exactly.

First let's find the address of the ```isRooted``` method.
```
[0x100005490]> ic ~ isRooted
0x100005c24 method 1      isRooted
```

We have found our ```isRooted``` method at the address of _0x100005c24_ so let's seek to that address using the ```s``` command and dump 10 bytes from that address.

```
[0x100005490]> s 0x100005c24
[0x100005c24]> pd 10$$
            ;-- func.100005c24:
            ;-- method.RootCheckResult.isRooted:
            0x100005c24      00204039       ldrb w0, [x0, 8]           ; [0x8:4]=-1 ; 8
            0x100005c28      c0035fd6       ret
            ;-- func.100005c2c:
            ;-- method.RootCheckResult.setIsRooted::
            0x100005c2c      02200039       strb w2, [x0, 8]
            0x100005c30      c0035fd6       ret
            ;-- func.100005c34:
```

We can see by the comments that the method ```isRooted``` is inside the ```RootCheckResult``` interface.

This method of searching is called cross referencing or XREFs.

## Patching
Copy your binary and load the copied one into the r2 with write mode.
```
$ cp original_binary copy_binary
$ r2 -w copy_binary
```

Let's seek to our ```isRooted``` method and disassemble it.
```
[0x100005490]> ic ~ isRooted
0x100005c24 method 1      isRooted
[0x100005c24]> s 0x100005c24
[0x100005c24]> pd 2$$
            ;-- func.100005c24:
┌ 8: method.RootCheckResult.isRooted (int64_t arg1);
│ bp: 0 (vars 0, args 0)
│ sp: 0 (vars 0, args 0)
│ rg: 1 (vars 0, args 1)
│           0x100005c24      00204039       ldrb w0, [x0, 8]           ; [0x8:4]=-1 ; 8 ; arg1
└           0x100005c28      c0035fd6       ret
```

We can see that we are loading our _w0_ return register from the _x0 + 8_ address.

So let's change that instruction to always return 0 meaning that the device is not rooted.

To do that, we will use ```w``` command. Let's see our options for the command.
```
[0x100005490]> w?
Usage: w[x] [str] [<file] [<<EOF] [@addr]
| w[1248][+-][n]       increment/decrement byte,word..
| w foobar             write string 'foobar'
| w0 [len]             write 'len' bytes with value 0x00
| w6[de] base64/hex    write base64 [d]ecoded or [e]ncoded string
| wa[?] push ebp       write opcode, separated by ';' (use '"' around the command)
| waf f.asm            assemble file and write bytes
| waF f.asm            assemble file and write bytes and show 'wx' op with hexpair bytes of assembled code
| wao[?] op            modify opcode (change conditional of jump. nop, etc)
| wA[?] r 0            alter/modify opcode at current seek (see wA?)
| wb 010203            fill current block with cyclic hexpairs
```

We can notice that we have ```wao``` command which is exactly what we want.
```
[0x100005490]> wao?
| wao [op]  performs a modification on current opcode
| wao nop   nop current opcode
| wao jinf  assemble an infinite loop
| wao jz    make current opcode conditional (zero)
| wao jnz   make current opcode conditional (not zero)
| wao ret1  make the current opcode return 1
| wao ret0  make the current opcode return 0
| wao retn  make the current opcode return -1
| wao nocj  remove conditional operation from branch (make it unconditional)
| wao trap  make the current opcode a trap
| wao recj  reverse (swap) conditional branch instruction
| WIP:      not all archs are supported and not all commands work on all archs
```

We can notice the command ```wao ret0``` which makes the _current_ opcode return 1. Since we are already on that instruction we don't need to seek any further.
```
[0x100005c24]> wao ret0
Written 8 byte(s) (mov x0, 0,,ret) = wx 000080d2c0035fd6
[0x100005c24]> pd 2$$
            ;-- func.100005c24:
┌ 8: method.RootCheckResult.isRooted (int64_t arg1);
│ bp: 0 (vars 0, args 0)
│ sp: 0 (vars 0, args 0)
│ rg: 1 (vars 0, args 1)
│           0x100005c24      000080d2       movz x0, 0                 ; arg1
└           0x100005c28      c0035fd6       ret
```

After writing and disassembling we can see that we no longer see ldrb instruction but instead loading 0 into the x0 register.

Quit radare, copy the binary into the Payload directory, zip it, sign it and install on your device.

This was really short introduction to radare and jailbreak bypass and will for sure publish some posts regarding radare. I hope you like it :)
