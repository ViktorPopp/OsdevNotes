# Building a freestanding kernel in Rust

In this part we will build a freestanding Rust binrary with a custom Limine-compatible linker script.

## The Linker Script

After you create a new crate the next step is to write a linker script. This script ensures that each section of the program is placed in the correct location in memory. Since we are building a static executable the kernel must be loaded at the correct address. If it is loaded at the wrong address the system will fail to run.

The first thing we want to specify in the script is the output format to be a x86_64 ELF64 file:

```plaintext
OUTPUT_FORMAT(elf64-x86-64)
```

Next we want to specify our entry poin to be `kmain`:

```rust
ENTRY(kmain)
```

Then we tell what program headers to generate:

```rust
PHDRS
{
    text    PT_LOAD;
    rodata  PT_LOAD;
    data    PT_LOAD;
}
```

After that we will begin defining or sections and segments.

A linker section are logical groupings of code and data before linking. We define the following:

* `.text`: Contains the executable machine code
* `.data`: Contains initialized global and static variables.
* `.rodata`: Contains initialized global and static variables.
* `.bss`: Contains uninitialized global/static variables (or initialized to zero). The section exists in the binary’s metadata, but it doesn’t store the actual zeros — the loader allocates and clears the memory at runtime.

Linker segments then describe how the sections will be loaded into memory. We define the following:

* `:text`: Contains `.text`. This segment typically has read and execute permissions.
* `:rodata`: Contains `.rodata`. This segment typically gets moved into the read-only part of `:text`.
* `:data`: Contains `.data` and at the end `.bss` to prevent unneeded zeros.

At the end we discard the `.note` and `.eh_frame` sections as they may cause issues on some hosts. We also want to be placed in the topmost 2 GiB of the address space because thats what the Limine spec mandates. Any address in that region will mostly work but we choose `0xffffffff80000000` as it is the most normal. we also align each section to the next memory page which makes sure everything is properly structured.

With all of that we get this:

```rust
SECTIONS
{
    . = 0xffffffff80000000;

    .text : {
        *(.text .text.*)
    } :text

    . = ALIGN(CONSTANT(MAXPAGESIZE));

    .rodata : {
        *(.rodata .rodata.*)
    } :rodata

    . = ALIGN(CONSTANT(MAXPAGESIZE));

    .data : {
        *(.data .data.*)

        KEEP(*(.requests_start_marker))
        KEEP(*(.requests))
        KEEP(*(.requests_end_marker))
    } :data

    .bss : {
        *(.bss .bss.*)
        *(COMMON)
    } :data

    /DISCARD/ : {
        *(.eh_frame*)
        *(.note .note.*)
    }
}
```

Right now our linker script doesn't really do anything. To use it we create a `build.rs` that passes the script to rustc and rebuild completely if it changes.

```rust
fn main() {
    println!("cargo:rustc-link-arg=-Tlinker.ld");
    println!("cargo:rerun-if-changed=linker.ld");
}
```

You will also need to add `-C relocation-model=static` to the `RUSTFLAGS` environment variable in your build system. I recommend `gmake`.

## The Entrypoint

Next we will create out simple entrypoint in `src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::arch::asm;

#[unsafe(no_mangle)]
unsafe extern "C" fn kmain() -> ! {
    halt();
}

#[panic_handler]
fn kpanic(_info: &core::panic::PanicInfo) -> ! {
    halt();
}

fn halt() -> ! {
    loop {
        unsafe {
            #[cfg(target_arch = "x86_64")]
            asm!("hlt");
            #[cfg(any(target_arch = "aarch64", target_arch = "riscv64"))]
            asm!("wfi");
            #[cfg(target_arch = "loongarch64")]
            asm!("idle 0");
        }
    }
}
```

## The Makefile

To easier build our kernel with diffrent configs and profiles we use a makefile.

We want 2 things: an `all` rule for building the kernel and a `clean` rule for removing intermediates. We also nuke all build-in variables and create a custom macro for setting user variables.

```py
# Nuke built-in rules and variables.
MAKEFLAGS += -rR
.SUFFIXES:

# Utility macro for setting user variables
override USER_VARIABLE = $(if $(filter $(origin $(1)),default undefined),$(eval override $(1) := $(2)))

# This is the name that our final executable will have.
override OUTPUT := kernel

# We will use 86_64
$(call USER_VARIABLE,KARCH,x86_64)

# Set the target
ifeq ($(RUST_TARGET),)
    override RUST_TARGET := $(KARCH)-unknown-none
endif

# Set the profile to development
ifeq ($(RUST_PROFILE),)
    override RUST_PROFILE := dev
endif

# Set the profiles subdirectory
override RUST_PROFILE_SUBDIR := $(RUST_PROFILE)
ifeq ($(RUST_PROFILE),dev)
    override RUST_PROFILE_SUBDIR := debug
endif

.PHONY: all clean

all:
    RUSTFLAGS="-C relocation-model=static" cargo build --target $(RUST_TARGET) --profile $(RUST_PROFILE)
    cp target/$(RUST_TARGET)/$(RUST_PROFILE_SUBDIR)/$$(cd target/$(RUST_TARGET)/$(RUST_PROFILE_SUBDIR) && find -maxdepth 1 -perm -111 -type f) kernel

clean:
    cargo clean
    rm -rf kernel
```

We also create a Cargo config file in `.cargo/config.toml`...

```js
[unstable]
build-std = ["core", "compiler_builtins", "alloc"]
```

...and a `rust-toolchain.toml` to pin our nightly toolchain to improve stability:

```js
[toolchain]
channel = "nightly-2025-04-03"
targets = ["x86_64-unknown-none"]
```
