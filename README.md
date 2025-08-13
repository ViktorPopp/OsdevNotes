# OSDev Guide

Every operating system starts as a small program that does almost nothing. Over time, it gains the ability to manage memory, run programs, and store files. In this guide, we will start from an empty computer and build up those abilities one step at a time.

## Structure Of The Guide

This guide is structured into chapters. Each chapter will adds a new layer to our kernel, expanding its capabilities. While it isn't necessary to follow the chapters in order later chapters might reference earlier ones.

## What is an Operating System?

At its simplest an operating system is a software that manages hardware resources.

It's responsible for:

* **Process management:** Deciding which programs run and when
* **Memory management:** Keeping that of whats stored where and ensures that programs don't step on eachothers data.
* **Device I/O:** Talking the hardware like disks, keyboards and screens.
* **Filesystems:** Organsizing persistent data so it can be stored and retrieved.
* **User interfaces:** Enabling humans (and sometimes other machines) to interact with the system.

## Layers of an OS

A typical OS has distinct layers each with their own responsibilities:

1. **Bootloader:** The minimal code which runs first on power-on (after the BIOS), responsible for loading a kernel from disk into memory
2. **Kernel:** The heart of the OS managing processes, memory and devices.
3. **Drivers:** Software which talks directly to hardware.
4. **Libraries:** Common code used by applications.
5. **Userspace:** Applications and interfaces running on top of the kernel.

## Assumed Knowledge

When writing an OS you will benefit from experience in:

* A systems programming language like Rust, C or C++.
* Low-level memory concepts such as memory addressing, pointers and bitwise operations.
* Simple computer architecture such as how the CPU executes instructions and how memory is structured.
* How a toolchain work with linkers and compilers.

## The Development Journey

Here is the broad roadmap we will follow:

1. Building a freestanding kernel
2. Booting using Limine
3. Terminal output
4. Loading a GDT
5. Loading an IDT
6. Building a PIC8259 driver
7. Physical memory
8. Paging
9. Virtual memory
10. Heap allocations
