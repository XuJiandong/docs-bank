---
title: Introduction to CKB Scripts
marp: true
author: xjd
size: 16:9
theme: gaia
backgroundColor: default
backgroundImage: bg.jpeg
---
# What is CKB Scripts
![bg opacity:90%](./bg.jpeg)
<!-- header: Introduction to CKB Scripts -->

Same as [Ethereum Smart Contract](https://ethereum.org/en/developers/docs/smart-contracts)

An executable in [RISC-V](https://riscv.org/) instructions, running on CKB blockchain.

---
# What Programming Languages?
![bg opacity:90%](./bg.jpeg)
<!-- header: Introduction to CKB Scripts -->

Any programming languages which can be compiled into RISC-V target.
- C
- C++
- Rust
- ...

---
# A Simple Example
![bg opacity:90%](./bg.jpeg)
<!-- header: Introduction to CKB Scripts -->
```C
int main(int argc, const char* argv[]) {
  return 0;
}
```
[Full Example](https://github.com/XuJiandong/risc-v-playground/blob/master/c/hello.c)

---
# No Dependency
![bg opacity:90%](./bg.jpeg)
<!-- header: Introduction to CKB Scripts -->

Like a embedded program executing on hardware without OS. It means:
- No Dll/Shared library
- No file system
- No time/clock
- ...

Only 4M memory and a few syscalls available.


---
# How to Compile in C
![bg opacity:90%](./bg.jpeg)
<!-- header: Introduction to CKB Scripts -->
[Makefile](https://github.com/XuJiandong/risc-v-playground/blob/7cb13fc7774f6e749a391f5993b061e2646c016e/Makefile#L12-L16)
```Makefile
CFLAGS := -fPIC -O3 -fno-builtin-printf -fno-builtin-memcmp \
-nostdinc -nostdlib -nostartfiles -fvisibility=hidden \
-fdata-sections -ffunction-sections\
-I deps/ckb-c-stdlib -I deps/ckb-c-stdlib/libc \
-Wall -Werror -Wno-nonnull -Wno-nonnull-compare -Wno-unused-function -g
LDFLAGS := -Wl,-static -Wl,--gc-sections
```
Notable compiler options:
- nostdinc/nostdlib/nostartfiles


---
# <TODO>
![bg opacity:90%](./bg.jpeg)
<!-- header: Introduction to CKB Scripts -->

