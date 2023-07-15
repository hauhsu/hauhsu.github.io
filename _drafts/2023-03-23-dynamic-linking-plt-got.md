---
layout: single
title: PLT & GOT
date: 2023-03-23 12:23 +0800
sitemap: false
---

This article is about how PLT and GOT works in dynamic linking.

There are two parts in this article. The first part introduces the brief
concept of the dynamic linking, where PLT and GOT involve. The second part
compiles a simple hello world C program to RISC-V executable, then use
disassembler and debugger to see how it works practically.

This article assumes you have the basicthe assembly code and ABI concepts.

# Concept: Function Call to a Method in Dynamic Library

## Introduction

This section introduces the basic concepts of dynamic linking, mainly how
PLT and GOT works, but won't involve dynamic linker internals. Also, for
convenience, we will use some easy pseudo assembly codes to indicate the
operations in program and PLT.

## Concept

In static linking, the assembly of calling a function is simple:

```asm
sum:
 # Sums the first (a0) and second (a1) parameters and returns it. 
 add a0, a0, a2
 return  # a0 is the return register

main:
 move a0 3
 move a1 2
 call sum
```

For dynamic linking, it becomes a little bit complex. There are two tables in memory:

* PLT: **P**rocedure **L**inkage **T**able. An entry contains a small piece of code.
* GOT: **G**lobal **O**ffset **T**able. An entry stores a dynamic library function address.

Each function in the dynamic library has a corresponding entries in both PLT and GOT.
When calling a function in dynamic library, the PC jumps to the PLT entry of
the function. The small piece of code in the PLT loads the function address
from GOT, and then jump to the address. As the following figure shows:

```text
┌───────────────┐       ┌──────────────────────────┐     ┌──────────────────────────┐
│main:          │       │           PLT            │     │           GOT            │
│   ...         │       │──────────────────────────│     │──────────────────────────│
│               │       │ sys  │  call dynamic     │     │      │                   │
│   jal sum@PLT─┼──┐    │      │  loader           │     │      │                   │
│               │  │    ├──────┼───────────────────┤     ├──────┼───────────────────┤
│   ...         │  └───▶│ sum  │  load t0, foo@GOT │◀────│ sum  │  0x00007000       │
│               │  ┌────┼──────┼─ j    t0          │     │      │                   │
│foo:  ◀────────┼──┘    ├──────┼───────────────────┤     ├──────┼───────────────────┤
│               │       │ foo  │  load t0, bar@GOT │     │ bar  │  0x00007084       │
│   ...         │       │      │  j    t0          │     │      │                   │
│               │       │      │                   │     │      │                   │
└───────────────┘       │      │                   │     │      │                   │
                        └──────────────────────────┘     └──────────────────────────┘
```

As long as the GOT contains the right function address, the PLT can jump to the
function. But how does the GOT entries are filled?

## GOT Initialization

When the binay is loaded to the memory, the GOT entris are not filled with the correct
function memory addresses. The contents are point to the first entry of the PLT, which calls
to the dynamic loader. The dynamic loader then loads the required dynamic library and fill
the GOT entry.

```text
┌───────────────┐       ┌──────────────────────────┐     ┌──────────────────────────┐
│main:          │       │           PLT            │     │           GOT            │
│   ...         │       │──────────────────────────│     │──────────────────────────│
│               │       │ sys  │  call dynamic     │     │      │                   │
│   jal sum@PLT─┼──┐    │      │  loader           │     │      │                   │
│               │  │    ├──────┼───────────────────┤     ├──────┼───────────────────┤
│   ...         │  └───▶│ sum  │  load t0, foo@GOT │◀────│ sum  │  sys@PLT          │
│               │  ┌────┼──────┼─ j    t0          │     │      │                   │
│foo:  ◀────────┼──┘    ├──────┼───────────────────┤     ├──────┼───────────────────┤
│               │       │ foo  │  load t0, bar@GOT │     │ bar  │  sys@PLT          │
│   ...         │       │      │  j    t0          │     │      │                   │
│               │       │      │                   │     │      │                   │
└───────────────┘       │      │                   │     │      │                   │
                        └──────────────────────────┘     └──────────────────────────┘
```