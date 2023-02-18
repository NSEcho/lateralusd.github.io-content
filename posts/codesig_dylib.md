---
title: LC_LOAD_[WEAK_]DYLIB and LC_CODE_SIGNATURE
description: Adding new loads and removing code signature
date: 2023-02-18T21:32:40.136764+01:00
toc: true
refs:
    - https://www.google.com
    - https://www.facebook.com
---

## Introduction

During the engagement, often times I am patching the iOS application with FridaGadget.dylib inside of it.  
The process usually involves adding new *LC_LOAD_DYLIB* to the application and potentially new *LC_RPATH*.

In order to do these two, I was using [insert_dylib](https://github.com/tyilo/insert_dylib) and [install_name_tool](https://www.unix.com/man-page/osx/1/install_name_tool/).

As I was wondering how these two do these things, I wanted to give it a shot and try to implement my own version.  
Golang module to add these new loads and remove signature is available [here](https://github.com/lateralusd/gdylib).

## Preparation

First let's create the file test.c that we are going to utilize to test these things out.

```c
#include <stdio.h>

int main(void) {
    printf("hello world!");
    return 0;
}
```

To compile it, we can just do `clang -o test_file test.c`.

## Examining the binary

At the start of every MachO file, there is a header which contains specific information. We can view the raw header with _hexdump_.

```bash
$ hexdump -C test_file | head
00000000  cf fa ed fe 0c 00 00 01  00 00 00 00 02 00 00 00  |����............|
00000010  11 00 00 00 20 04 00 00  85 00 20 00 00 00 00 00  |.... ..... .....|
00000020  19 00 00 00 48 00 00 00  5f 5f 50 41 47 45 5a 45  |....H...__PAGEZE|
00000030  52 4f 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |RO..............|
00000040  00 00 00 00 01 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000060  00 00 00 00 00 00 00 00  19 00 00 00 88 01 00 00  |................|
00000070  5f 5f 54 45 58 54 00 00  00 00 00 00 00 00 00 00  |__TEXT..........|
00000080  00 00 00 00 01 00 00 00  00 40 00 00 00 00 00 00  |.........@......|
00000090  00 00 00 00 00 00 00 00  00 40 00 00 00 00 00 00  |.........@......|
```

`cf fa ed fe` is the actual magic file telling us what kind of file are we dealing with:
* file endian (big or little)
* 32 bit or 64 bit

The structure that header maps to looks like this:

__32 bit__
```c
struct mach_header {
    uint32_t    magic;      /* mach magic number identifier */
    cpu_type_t  cputype;    /* cpu specifier */
    cpu_subtype_t   cpusubtype; /* machine specifier */
    uint32_t    filetype;   /* type of file */
    uint32_t    ncmds;      /* number of load commands */
    uint32_t    sizeofcmds; /* the size of all the load commands */
    uint32_t    flags;      /* flags */
};
```

__64 bit__
```c
struct mach_header_64 {
    uint32_t    magic;      /* mach magic number identifier */
    cpu_type_t  cputype;    /* cpu specifier */
    cpu_subtype_t   cpusubtype; /* machine specifier */
    uint32_t    filetype;   /* type of file */
    uint32_t    ncmds;      /* number of load commands */
    uint32_t    sizeofcmds; /* the size of all the load commands */
    uint32_t    flags;      /* flags */
    uint32_t    reserved;   /* reserved */
};
```

Those four bytes we have seen map to `magic` inside the `mach_header` or `mach_header_64` structure.

* 32 bit magic: `0xfeedface`
* 64 bit magic: `0xfeedface`

Since the architecture on my Maco is 64 bit, we can confirm that our magic is indeed `0xfeedfacf`.

```bash
$ otool -h test_file
test_file:
Mach header
      magic  cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777228          0  0x00           2    17       1056 0x00200085
```

Another thing that is really important are `ncmds` and `sizeofcmds`.

`ncmds` tells us how many load commands are inside the binary and `sizeofcmds` tells us in term of bytes how much space those load commands consume.

Load commands follow immediately the header of the binary, so their location is `sizeof(header)`.

Load command is defined as:

```c
struct load_command {
    uint32_t cmd;       /* type of load command */
    uint32_t cmdsize;   /* total size of command in bytes */
};
```

* cmd - type(LC_CODE_SIGNATURE, LC_LOAD_DYLIB etc)
* cmdsize - total size of load command including `cmd` and `cmdsize`

So in order to extract the load commands we need to do the following:

* Read the header
* offset = sizeof(header)
* Loop hdr.ncmds times
   * read the load structure
   * offset += load.cmdsize

In golang, it would look something like this:

```go
package main

import (
	"encoding/binary"
	"fmt"
	"io"
	"os"
)

type LoadType uint32

const (
	LC_SEGMENT_64     LoadType = 0x19
	LC_CODE_SIGNATURE LoadType = 0x1d
)

// Header reflects mach_header_64
type Header struct {
	Magic      uint32
	CpuType    uint32
	CpuSubtype uint32
	Filetype   uint32
	NCmds      uint32
	SizeOfCmds uint32
	Flags      uint32
	Reserved   uint32
}

type Load struct {
	Cmd  LoadType
	Size uint32
}

func main() {
	f, err := os.Open(os.Args[1])
	if err != nil {
		panic(err)
	}
	defer f.Close()

	var hdr Header
	binary.Read(f, binary.LittleEndian, &hdr)

	fmt.Printf("magic: %x\n", hdr.Magic)
	fmt.Printf("ncmds: %d\n", hdr.NCmds)
	fmt.Printf("sizeofcmds: %d\n", hdr.SizeOfCmds)

	off, _ := f.Seek(0, io.SeekCurrent)
	for i := 0; i < int(hdr.NCmds); i++ {
		var ld Load
		binary.Read(f, binary.LittleEndian, &ld)
		switch ld.Cmd {
		case LC_SEGMENT_64:
			fmt.Printf("%d => LC_SEGMENT_64\n", i)
		case LC_CODE_SIGNATURE:
			fmt.Printf("%d => LC_CODE_SIGNATURE\n", i)
		}
		off += int64(ld.Size)
		f.Seek(off, io.SeekStart)
	}
}
```

```bash
$ go run main.go test_file
magic: feedfacf
ncmds: 17
sizeofcmds: 1056
0 => LC_SEGMENT_64
1 => LC_SEGMENT_64
2 => LC_SEGMENT_64
3 => LC_SEGMENT_64
16 => LC_CODE_SIGNATURE
```

We can see that _magic_, _ncmds_ and _sizeofcmds_ correspond to what we have seen using `otool -h`.

Lets quickly confirm that our load commands are in the order that our program returned.

```bash
$ otool -l test_file
test_file:
Load command 0
      cmd LC_SEGMENT_64
  cmdsize 72
  segname __PAGEZERO
   vmaddr 0x0000000000000000
   vmsize 0x0000000100000000
  fileoff 0
 filesize 0
  maxprot 0x00000000
 initprot 0x00000000
   nsects 0
    flags 0x0
Load command 1
      cmd LC_SEGMENT_64
  cmdsize 392
  segname __TEXT
   vmaddr 0x0000000100000000
   vmsize 0x0000000000004000
  fileoff 0
 filesize 16384
  maxprot 0x00000005
 initprot 0x00000005
   nsects 4
    flags 0x0
Section
...
Load command 2
      cmd LC_SEGMENT_64
  cmdsize 152
  segname __DATA_CONST
   vmaddr 0x0000000100004000
   vmsize 0x0000000000004000
  fileoff 16384
 filesize 16384
  maxprot 0x00000003
 initprot 0x00000003
   nsects 1
    flags 0x10
Section
  sectname __got
   segname __DATA_CONST
      addr 0x0000000100004000
      size 0x0000000000000008
    offset 16384
     align 2^3 (8)
    reloff 0
    nreloc 0
     flags 0x00000006
 reserved1 1 (index into indirect symbol table)
 reserved2 0
Load command 3
      cmd LC_SEGMENT_64
  cmdsize 72
  segname __LINKEDIT
   vmaddr 0x0000000100008000
   vmsize 0x0000000000004000
  fileoff 32768
 filesize 662
...
Load command 16
      cmd LC_CODE_SIGNATURE
  cmdsize 16
  dataoff 33024
 datasize 406
```

## Removing code signature