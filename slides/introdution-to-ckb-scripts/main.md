---
title: Introduction to CKB Scripts
marp: true
author: xjd
size: 16:9
theme: gaia
backgroundColor: default
header: Introduction to CKB Scripts
backgroundImage: ./bg.jpeg
paginate: true
theme: gaia
---
![bg opacity:90%](./bg.jpeg)
# Introduction to CKB Scripts

---
# What is CKB Scripts
![bg opacity:90%](./bg.jpeg)

Same as [Ethereum Smart Contract](https://ethereum.org/en/developers/docs/smart-contracts)

An executable in ELF format containing [RISC-V](https://riscv.org/) instructions, running on CKB blockchain.

---
# What Programming Languages?
![bg opacity:90%](./bg.jpeg)

Any programming languages which can be compiled into RISC-V target.
- C
- C++
- Rust
- ...

---
# A Simple Example
![bg opacity:90%](./bg.jpeg)

```C
int main(int argc, const char* argv[]) {
  return 0;
}
```
[Full Example](https://github.com/XuJiandong/risc-v-playground/blob/master/c/hello.c)

---
# No Dependency
![bg opacity:90%](./bg.jpeg)


Like an embedded program executing on hardware without OS. It means:
- No Dll/Shared library
- No file system
- No time/clock
- ...

Only 4M memory and a few syscalls available.

---
# How to Compile in C
![bg opacity:90%](./bg.jpeg)

[Makefile](https://github.com/XuJiandong/risc-v-playground/blob/7cb13fc7774f6e749a391f5993b061e2646c016e/Makefile#L12-L16)
```Makefile
CFLAGS := -fPIC -O3 -fno-builtin-printf -fno-builtin-memcmp \
-nostdinc -nostdlib -nostartfiles -fvisibility=hidden \
-fdata-sections -ffunction-sections\
-I deps/ckb-c-stdlib -I deps/ckb-c-stdlib/libc -g
LDFLAGS := -Wl,-static -Wl,--gc-sections
```
Notable compiler options:
- nostdinc/nostdlib
- nostartfiles

---
# GNU Toolchain in Docker
![bg opacity:90%](./bg.jpeg)


[docker image](https://github.com/nervosnetwork/ckb-production-scripts/blob/ddaeae8f065a5b600805b46d75c53de9bf290e68/Makefile#L28)
```
nervos/ckb-riscv-gnu-toolchain@sha256:aae8a3f79705f67d505d1f1d5ddc694a4fd537ed1c7e9622420a470d59ba2ec3
```
compiled from [ckb-riscv-gnu-toolchain](https://github.com/nervosnetwork/ckb-riscv-gnu-toolchain). Very large!


---
# ckb-c-stdlib
![bg opacity:90%](./bg.jpeg)


[ckb-c-stdlib](https://github.com/nervosnetwork/ckb-c-stdlib)
Yet another small C runtime library. 
- include functions: printf/memcpy/qsort/...
- not include: malloc/free

Used as C runtime library
Recall: nostdinc/nostdlib options

---
# One Big Header File
![bg opacity:90%](./bg.jpeg)

[One Big Header File](https://wiki.c2.com/?OneBigHeaderFile) is used for ckb-c-stdlib.

pros: easy to integrate for small project
cons: difficult to use when work with large project



---
# Syscalls, System Calls
![bg opacity:90%](./bg.jpeg)

Like linux syscalls, a [system call](https://github.com/nervosnetwork/ckb-c-stdlib/blob/master/ckb_syscalls.h) is a procedure that provides the interface between a script and the blockchain.
- ckb_load_cell_data
- ckb_load_script_hash
- ...


---
# Best Template to Start With
![bg opacity:90%](./bg.jpeg)


[always_success](https://github.com/nervosnetwork/ckb-production-scripts/blob/ddaeae8f065a5b600805b46d75c53de9bf290e68/Makefile#L46)
```C
#include "ckb_syscalls.h"
int main() { return 0; }
```
Compile all scripts by:
```Bash
make all-via-docker
```
A [playground](https://github.com/XuJiandong/risc-v-playground) for learning.

---
# Ckb-debugger
![bg opacity:90%](./bg.jpeg)

Can use [ckb-debugger](https://github.com/nervosnetwork/ckb-standalone-debugger/releases) to run simple scritps.

```bash
❯ ckb-debugger --bin build/always_success
Run result: 0
Total cycles consumed: 571
Transfer cycles: 56, running cycles: 515
```


---
# More Resources
![bg opacity:90%](./bg.jpeg)



https://docs-xi-two.vercel.app/ (tutorial, easy to read and follow)
https://github.com/nervosnetwork/rfcs (RFC, formal but complex)

---
# Thanks
![bg opacity:90%](./bg.jpeg)


Q&A
