---
title: Code signature and new dylibs (Part 1)
description: Adding new loads and removing code signature
date: 2023-02-18T21:32:40.136764+01:00
toc: true
tags:
    - macho
    - golang
    - ios
    - frida
refs:
    - https://opensource.apple.com/source/cctools/cctools-921/include/mach-o/loader.h.auto.html
---

## Introduction

During the engagement, often times I am patching the iOS application with FridaGadget.dylib inside of it.  
The process usually involves adding new *LC_LOAD_DYLIB* to the application and potentially new *LC_RPATH*.

In order to do these two, I was using [insert_dylib](https://github.com/tyilo/insert_dylib) and [install_name_tool](https://www.unix.com/man-page/osx/1/install_name_tool/).

As I was wondering how these two do these things, I wanted to give it a shot and try to implement my own version.  
Golang module to add these new loads and remove signature is available [here](https://github.com/lateralusd/gdylib).

In this first post, we are actually take a look how to remove the signature and in the next part we will add dylibs.

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

The steps to successfully remove code signature are:

* Replacing LC_CODE_SIGNATURE with null bytes
* Truncating file `datasize` bytes starting at `dataoff` (`dataoff = end - datasize`)
* Aligning the `filesize` in `__LINKEDIT` segment (`filesize + fileoff = total_size_of_truncated_binary`)
* Aligning the `strsize` in `__LCSYMTAB` load (`stroff + strsize = total_size_of_truncated_binary`)
* Decreasing `ncmds` inside the header by 1
* Decreasing `sizeofcmds` inside the header by the size of LC_CODE_SIGNATURE `cmdsize`

```go
package main

import (
	"bytes"
	"encoding/binary"
	"io"
	"os"
	"unsafe"
)

type LoadType uint32

const (
	LC_SYMTAB         LoadType = 0x2
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

// Load represents load_command
type Load struct {
	Cmd  LoadType
	Size uint32
}

// lcCode represents structure for LC_CODE_SIGNATURE
type lcCode struct {
	DataOff  uint32
	DataSize uint32
}

type segment64 struct {
	SegName    [16]byte
	VMAddr     uint64
	VMSize     uint64
	FileOffset uint64
	FileSize   uint64
	MaxProt    int32
	InitProt   int32
	NSect      uint32
	Flags      uint32
}

type symtab struct {
	SymOff  uint32
	NSyms   uint32
	StrOff  uint32
	StrSize uint32
}

func main() {
	f, err := os.Open(os.Args[1])
	if err != nil {
		panic(err)
	}
	defer f.Close()

	// read the header from the file
	var hdr Header
	binary.Read(f, binary.LittleEndian, &hdr)

	// slice that will keep all our loads
	var loads [][]byte
	// dataSize will contain size of the signature(inside LC_CODE_SIGNATURE)
	var dataSize uint32
	// total size of LC_CODE_SIGNATURE
	var lcSize uint32

	off, _ := f.Seek(0, io.SeekCurrent)
	for i := 0; i < int(hdr.NCmds); i++ {
		// read the load command
		var ld Load
		binary.Read(f, binary.LittleEndian, &ld)
		switch ld.Cmd {
		/* if we have LC_CODE_SIGNATURE
		* read it
		* save lc.DataSize inside dataSize variable
		* save total size of load inside lcSize
		* write zeroes for the whole load
		* append it to our load slice
		 */
		case LC_CODE_SIGNATURE:
			var lc lcCode
			binary.Read(f, binary.LittleEndian, &lc)
			dataSize = lc.DataSize
			lcSize = ld.Size
			// create ld.Size of null bytes
			zero := zeroSlice(int(ld.Size))
			loads = append(loads, zero)
		default:
			// otherwise just get raw bytes of the load
			f.Seek(off, io.SeekStart)
			buff := make([]byte, ld.Size)
			binary.Read(f, binary.LittleEndian, &buff)
			loads = append(loads, buff)
		}
		off += int64(ld.Size)
		f.Seek(off, io.SeekStart)
	}

	// current tells us from where we need to continue reading the binary
	current := int64(unsafe.Sizeof(hdr)) + int64(hdr.SizeOfCmds)

	// create buffer that will have our modified binary
	buff := new(bytes.Buffer)

	// reduce size of ncmds by 1 because we are removing one load(LC_CODE_SIGNATURE)
	// reduce size by the total size of LC_CODE_SIGNATURE load command
	hdr.NCmds -= 1
	hdr.SizeOfCmds -= lcSize

	// write the header to it
	binary.Write(buff, binary.LittleEndian, hdr)

	end, _ := f.Seek(0, io.SeekEnd)

	// write all the loads to the binary
	for _, load := range loads {
		raw := bytes.NewBuffer(load)
		var ld Load
		binary.Read(raw, binary.LittleEndian, &ld)
		switch ld.Cmd {
		case LC_SEGMENT_64:
			var seg segment64
			binary.Read(bytes.NewBuffer(raw.Bytes()), binary.LittleEndian, &seg)
			segName := string(stripNull(seg.SegName[:]))
			if segName == "__LINKEDIT" {
				// if we found __LINKEDIT, reduce filesize by the datasize
				// and write it to our buffer
				seg.FileSize -= uint64(dataSize)
				modifiedLinkedit := new(bytes.Buffer)
				binary.Write(modifiedLinkedit, binary.LittleEndian, ld)
				binary.Write(modifiedLinkedit, binary.LittleEndian, seg)
				buff.Write(modifiedLinkedit.Bytes())
			} else {
				// if the segment names does not match __LINKEDIT, write the raw load
				buff.Write(load)
			}
		case LC_SYMTAB:
			// align lc_symtab so that StrOff + StrSize = total size of truncated binary
			var sym symtab
			binary.Read(raw, binary.LittleEndian, &sym)
			sz := end - int64(dataSize)
			diffSize := int64(sym.StrOff+sym.StrSize) - sz
			newSize := sym.StrSize - uint32(diffSize)
			sym.StrSize = newSize
			newS := new(bytes.Buffer)
			binary.Write(newS, binary.LittleEndian, ld)
			binary.Write(newS, binary.LittleEndian, sym)
			buff.Write(newS.Bytes())
		default:
			buff.Write(load)
		}
	}

	/*
		    1. calculate how much we need to read
		    2. read restSize
		===========================================================
		|hdr + loads|       rest                    | code sig    |
		===========================================================
		<- current -><- sizeTillCodeSignature      -><- dataSize ->
	*/
	f.Seek(current, io.SeekStart)
	restSize := end - current - int64(dataSize)
	rest := make([]byte, restSize)
	binary.Read(f, binary.LittleEndian, rest)

	buff.Write(rest)

	newFile, err := os.Create("new_file")
	if err != nil {
		panic(err)
	}

	io.Copy(newFile, buff)
}

func zeroSlice(size int) []byte {
	buff := make([]byte, size)
	for i := 0; i < size; i++ {
		buff[i] = 0
	}
	return buff
}

func stripNull(buff []byte) []byte {
	for i := 0; i < len(buff); i++ {
		if buff[i] == 0 {
			return buff[:i]
		}
	}
	return buff
}
```

The code does the following:  
1. Read the header  
2. Read all the loads; if we got LC_CODE_SIGNATURE save datasize and total load size; write load size bytes into our load slice  
3. For every other load just get raw bytes and write into load slice  
4. Calculate our current position so that we know from where we need to start reading the rest of the binary  
5. Reduce `ncmds` and `sizeofcmds` in the header  
6. Write header inside the buffer  
7. Loop through all loads once again  
8. If we have LC_SEGMENT_64 and its name is __LINKEDIT, reduce `filesize` by `datasize` so that `filesize -= datasize`  
9. If we have LC_SYMTAB, reduce `strsize` so that `stroff + strsize = total_size_of_truncated_binary`  
10. For all other loads, just write them  
11. Calculate how much we need to read from the binary  
12. Write it to the buffer and then save it to the new file  

To confirm that we have indeed removed code signature, lets run `codesign -dvv` followed with `otool -l` on `test_file` and `new_file`.

```bash
$ codesign -dvv test_file
Executable=/Users/demon/tools/lateralusd.github.io-content/test_file
Identifier=test_file
Format=Mach-O thin (arm64)
CodeDirectory v=20400 size=386 flags=0x20002(adhoc,linker-signed) hashes=9+0 location=embedded
Signature=adhoc
Info.plist=not bound
TeamIdentifier=not set
Sealed Resources=none
Internal requirements=none
$ otool -l test_file
Load command 15
      cmd LC_DATA_IN_CODE
  cmdsize 16
  dataoff 32920
 datasize 0
Load command 16
      cmd LC_CODE_SIGNATURE
  cmdsize 16
  dataoff 33024
 datasize 406

```

```bash
$ codesign -dvv new_file
new_file: code object is not signed at all
$ otool -l new_file
otool -l new_file | tail
Load command 14
      cmd LC_FUNCTION_STARTS
  cmdsize 16
  dataoff 32912
 datasize 8
Load command 15
      cmd LC_DATA_IN_CODE
  cmdsize 16
  dataoff 32920
 datasize 0
```

We can see that we have successfully removed code signature from the binary, as the final check lets run `install_name_tool` to make sure that everything is working correctly.

```bash
$ install_name_tool -add_rpath @executable_path/. new_file
otool -l new_file | tail
 datasize 8
Load command 15
      cmd LC_DATA_IN_CODE
  cmdsize 16
  dataoff 32920
 datasize 0
Load command 16
          cmd LC_RPATH
      cmdsize 32
         path @executable_path/. (offset 12)
```

In the next part of the series, we will take a look at the how to actually add new load command.