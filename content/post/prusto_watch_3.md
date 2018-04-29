---
title: "Prusto Watch #3: Learning How to Properly Debug Arm"
date: 2018-04-29T13:36:43-04:00
draft: false
---

This post will document my experience debugging a strange error I ran into while developing firmware for the Prusto Watch. Through the process I learned how the startup process for Arm actually works and how to use debugging tools in a much more effective way. Those that are familiar with Arm assembly and debugging will probably not learn anything from this, but I think it could be useful for those that are new to Arm.

## Overview

After soldering the third version of the Prusto Watch hardware, I flashed `blinky` onto it to ensure that everything was functioning properly. The program loaded just fine, and the LED blinked.  Next I tried to flash the firmware for showing graphics on the memory LCD. Once again it loaded just fine on the MCU, but nothing displayed when running the program through GDB. Ctrl-C'ing made GDB output the following:

```
^C
Program received signal SIGINT, Interrupt.
0x1fffeec2 in ?? ()
(gdb) 
```

Obviously something went wrong, which is weird since the firmware ran just fine on previous iterations. At the time, I had no idea what this error meant, so I attempted to flash the `release` version (which has compile-time optimizations) of the same firmware.  In retrospect, this doesn't make any sense to do (but I'm glad I did it). What I found is that the `release` version of the firmware ran just fine, and shapes displayed on the screen.

This seemed like a bug in the compiler. Since embedded Rust is using a nightly version of the compiler, something could have broken in a recent change. But after searching through issue logs, and talking on IRC, this seemed like an issue only I was having. IRC members also found it *really* weird that it worked when in `release` mode and not in `debug` mode. So began my journey debugging Arm and learning how things work.

## Getting to A Minimum, Complete, Verifiable Example

The graphics code for the memory LCD is rather large if you include dependencies, so there are many, many places for things to go wrong. In order to get to the root of the problem, I slowly took code out of the library until I was left with just what was broken. I found the issue was with the initialization of the main display object,

```rust
impl<SPI, CS, DISP, E> Ls010b7dh01<SPI, CS, DISP>
where
    SPI: Write<u8, Error = E>,
    CS: OutputPin,
    DISP: OutputPin,
{
    /// Create a new Ls010b7dh01 object
    ///
    /// `disp` is the pin connected to the display_enable pin of
    /// the memory LCD.
    pub fn new(spi: SPI, mut cs: CS, mut disp: DISP) -> Self {
        disp.set_low();
        cs.set_low();

        let buffer = [[0; 16]; 128];

        Self {
            spi,
            cs,
            disp,
            buffer,
        }
    }
}
```

The code would hang (as in run indefinitely) when the control sequence reached this function, and it *seemed* to occur right at the `disp.set_low();` line. When I took out that line the program still didn't work. However, taking out the `let buffer = [[0; 16]; 128];` line made the program run. So it seemed like the issue is with initializing a "lot" of memory on the stack. The minimum, complete, verifiable example I had was then:

``` rust
fn main() {
    // Insert a breakpoint
    asm::bkpt();

    // The error line, initializing memory on the stack
    let buffer = [[0; 16]; 128];

    // Another breakpoint which we never hit in `debug` mode
    asm::bkpt();

    loop {
        // Wait on interrupt, apparently needed due to a bug in LLVM
        asm::wfi();
    }
}
```

## Understanding the Error

In order to debug the issue I was having, I had to first understand the error GDB was outputting (with help from people on IRC). Loading the `debug` version of the firmware again, I got GDB to the same spot. This time, I tried to find the backtrace.

```
^C
Program received signal SIGINT, Interrupt.
0x1fffeec2 in ?? ()
(gdb) bt
#0  0x1fffeec2 in ?? ()
#1  0x1fffee9e in ?? ()
Backtrace stopped: previous frame identical to this frame (corrupt stack?)
(gdb)
```

What this means is that something went wrong with the stack memory, and generally it is due to stack overflow where the stack runs out of memory. However, I am using a microcontroller with *more* SRAM (40KB) than my previous microcontroller (12KB), so it didn't make sense that I was running out of memory since it worked fine on the previous microcontroller.

However, the fact that the issue was with the stack makes sense given the code that was breaking, so I knew I was on the right track.

## Arm Vector Tables

Since I knew I had *enough* SRAM, the issue was definitely not a stack overflow. I learned on IRC that it was most likely a linking error, and that my vector table might be wrong. Coming from the world of AVR, I had no idea what a vector table even was.

The Arm vector table is actually quite straightforward. It is a list of places in memory for use by the Arm Core. Most of it is just a list of function-pointers for dealing with interrupts. For instance, if there is a serial receive interrupt, then the Arm Core will set the program counter to the value in the vector table corresponding to the serial receive interrupt. AVR does have interrupt vectors, which I have used, but the vector table is handled under-the-hood, and thus I never had to explicitly create it. In Arm, however, it has to be explicitly created, probably to keep things flexible.

The reason this has relevance to my problem is that the first value of the Arm vector table is where the stack memory is located. Members on IRC figured that this value was incorrect, and thus why the stack was corrupt.

## Arm Tools

Thankfully there are some really awesome tools for inspecting these types of issues. One of which is the `objdump` tool. Using it, I was able to quickly verify that my stack was in fact in the right place:

```bash
$ arm-none-eabi-objdump -D target/thumbv7em-none-eabihf/debug/cortex-m-quickstart | head -n10

target/thumbv7em-none-eabihf/debug/cortex-m-quickstart:     file format elf32-littlearm


Disassembly of section .vector_table:

08000000 <_svector_table>:
 8000000:       2000a000        andcs   sl, r0, r0
```

The flash memory of the STM32f303 MCU I am using starts at `0x0800_0000`. The first value, which corresponds to the stack memory location, is `0x2000_a000`. The SRAM of the MCU is located at `0x2000_0000` and `0xa000` is 40KB. Since the stack is top-down, this means that the vector table is correct, and the stack memory is located at the right place.

## Digging Through Assembly

Given that my vector table was correct, I stumped the folks on IRC. The last bit of advice I got was "step through the assembly and try to find something going wrong."

Once again I started up GDB, but instead of letting the program run I was going to step through every line of assembly and figure out where things go wrong. From reading the GDB documentation, the commands I would need are:

- `stepi` for stepping through the code one assembly instruction at a time
- `display/i $sp` and `display/i $pc` for displaying the stack pointer and program counter at each step.
- `disas` for disassembling and printing the assembly.

I started up GDB, and just to get my bearings I ran `disas`.

```bash
(gdb) disas
Dump of assembler code for function cortex_m_rt::reset_handler:
   0x08000400 <+0>:     push    {r7, lr}
   0x08000402 <+2>:     mov     r7, sp
   0x08000404 <+4>:     sub     sp, #16
=> 0x08000406 <+6>:     movw    r0, #0
   0x0800040a <+10>:    movt    r0, #8192       ; 0x2000
   0x0800040e <+14>:    movw    r1, #0
   0x08000412 <+18>:    movt    r1, #8192       ; 0x2000
(gdb)
```

The `.text` section (where the actual program is) is located at 0x0800_0400, since the vector table is 0x400 bytes long. So this shows that the program counter is on the fourth instruction, after completing three instructions. The first three instructions are:

- push R7 and the link register (LR) onto the stack
- Move the stack pointer (SP) into R7
- Subtract the number 16 from SP

Since our vector table defines the stack pointer to start at 0x2000_a000, at this point we would expect the stack pointer to be at 0x2000_9ff0 (which is 0x2000_a000 - 16). However, upon inspecting this register using GDB, I found it to be very different.

```bash
(gdb) display $sp
1: $sp = (void *) 0x20001240
(gdb)
```

The difference between where I expect the stack pointer to be, and where it actually is, is a difference of about 36KB. This means something is very amiss, and it happens immediately.

## An Issue From The Past

When I first soldered the new revision of the PCB, I was able to flash `blinky` on it just fine. However, the program did not run when the board was unplugged from my computer. This was an issue, since a smart watch is not very usable if you have to have it plugged into GDB in order to run firmware. 

After trying to figure out the issue, someone suggested checking my `boot0` pin, and ensuring that it was tied low. It turns out that in the latest revision, I had accidentally tied my `boot0` pin high.

The `boot0` pin defines the *aliasing* of the 0x0000_0000 memory region. This is the region of memory that is used when the microcontroller starts. If `boot0` is low, the memory region is aliased to the flash memory, or 0x0800_0000, which is where my code is.  However, if the `boot0` pin is high, it is aliased to the system memory, or 0x1FFF_D800, where the STM bootloader is located.

What I learned is that when GDB is running, it coerces the program counter into the right position, which for me is 0x0800_0000. But when GDB was not being used, and the board was running unconnected, the microcontroller was running the code in the system memory.

This seemed like a small issue. It just meant this revision would not be able to run without being connected to GDB, which is annoying (since I can't wear it) but also not a deal breaker since I just need to do development with it. Little did I know that this issue is also what is making the stack pointer all crazy.

## The Root Cause

After spending a long time trying to figure out why the stack pointer was so wrong, I remembered the issue I had with the `boot0` pin. What if the vector-table is set before GDB is able to coerce the program counter?

I examined the system memory region, 0x1FFF_D800, to see what the vector table in system memory sets the stack pointer to.

```bash
(gdb) x 0x1fffd800  # inspect the memory at the address
0x1fffd800:     0x20001258
```

The system memory vector table has the stack sized much smaller than it could be. In fact, this value is *almost* the value that GDB shows the stack pointer to be on initial start up. GDB showed the stack pointer to be 0x20001250 (before it had 16 subtracted from it), while the system memory vector table sets it to 0x20001258. This is a difference of 8 bytes, which can be explained to be missing since the stack pointer has to be aligned in memory, so those 8 bytes are truncated. Aha! This seems really promising.

My theory, then, is that the vector table is sourced from *system memory*, and then GDB coerces the program counter to *flash memory* to run the program. Since the system memory stack size is so small, we actually *do* have a stack overflow error.

This also explains why the program is able to run in `release` mode. Due to compile-time optimizations, the stack usage is much lower and does not cause it to overflow. Running the firmware on older revisions with the correct `boot0` pin helps reinforce my theory, since the stack pointer is consistently in the right place. It seems like this is the issue, and one that will be fixed in the next board revision!

## Final Thoughts

From the process of debugging this issue, I learned a *lot* about Arm. As frustrating and as long as it took, I have a significantly better grasp about what is going on under-the-hood. I also know how to use my tools much more effectively even though I'm positive that there are still many tricks that are waiting to be learned.

Although I have made no progress on actually writing firmware for the watch, I think it has been completely worth it from a learning perspective. A few times I thought about just working with an older revision, and hoping that the next revision of the board would work fine because *who knows* what could be going wrong. But deep down I wanted to *know* why it wasn't working. And now, I'm pretty sure I do.

