---
title: "Prusto Watch #1: First Steps into Arm and the Embedded-Rust Ecosystem"
date: 2018-02-12T21:34:57-05:00
tags: []
draft: false
---

In this post I will describe my initial dive into Arm development using Rust. There are potentially many mistakes in my understanding, so this post should not be used as a reference for how to do embedded development in Rust. However, I think it is useful to describe the learning process being new to Rust and Arm development so that those more experienced can see the pain-points of getting started.

## Firmware Tools

To get started with writing firmware with Rust, I had to gather a few tools and resources. My build system follows japaric's system, which he writes about [here](http://blog.japaric.io/quickstart/). It was fairly straightforward to get set-up, so I won't repeat what he talks about here. 

One nice aspect about the tools required for writing firmware in Rust is that they are all open source and all CLI tools. This means that it is easy write a script to pipe them together and modify the work-flow for how you want to work.

The general work-flow I took was running `openocd` in the background, compiling with `xargo` and flashing + debugging with `arm-none-eabi-gdb`. Technically you can combine the last two steps by running `xargo run`, which will call `gdb` automatically.

## Programming for Arm

One of the major things I learned this week was developing on Arm MCUs. I come from a background of AVR, so the world of Arm was slightly mysterious. However, after diving into a few datasheets and reading various blog posts online I have mostly demystified them.

The biggest difference I found between AVR and Arm is that for Arm you have to manually turn-on various peripherals, such as a GPIO-pin group. This is done in the set of registers under the *Reset and Clock Control* (RCC) group. At first, this was extremely confusing, since the name of RCC is not at all descriptive of this functionality.

The second main difference I found was that GPIO pins had to be set for a specific alternative function. It was not enough to know that a GPIO pin *could* operate SPI1, you have to manually set that GPIO pin to the correct alternate function first. Thankfully, a lot of the work that is being done in the Rust ecosystem is ensuring that you correctly set up GPIO pins for various peripherals at *compile-time*.

An extremely frustrating aspect of setting up GPIO pins to their alternate functionality is that the alternate function number for each GPIO pin is *not* in the 1141 page reference manual. They are only in the short, 148 page datasheet. This is absolutely ridiculous, and took far too long to figure out.

## Embedded Rust Ecosystem

The embedded ecosystem for Rust is quite young, and there really isn't much out there. However, what is there seems to be very well thought-out and (mostly) functional.

### svd2rust

Perhaps the nicest tool available is the `svd2rust` program. This is a tool that takes in an `svd` file, which is an xml file and the standardized way for describing the peripherals of an Arm device and their registers, and converts it into a Rust library. It isn't perfect, and there seems to be some "fixing" of the `svd` files required before the Rust library is full-featured. However, once the library is created, you can manipulate registers in an extremely readable manner.

For example, the following code sets up pin PE9 as an output pin:

```rust 
let dp = stm32f30x::Peripherals::take().unwrap();
dp.GPIOA.moder.modify(|_, w| w.moder9().output());
```

Although the modify routine on a register takes a closure, the compiled code, according to japaric's tests, is just as fast as modifying the registers by bit-shifting. This is awesome, since the code is significantly more readable than setting registers using bit masks.

The other nice aspect of the generated Rust library is that it allows us to use Rust to its *full* power. Each register is a normal Rust variable and the ownership model applies. This means Rust will catch, at *compile-time*, unsafe memory patterns and race-conditions. Essentially you have all the guarantees that Rust provides when doing direct register manipulation, which is pretty awesome. 

### Real Time For the Masses (RTFM)

While `svd2rust` provides a Rusty library for working with registers, RTFM provides a *beautiful* framework for working with interrupts. Japaric talks extensively about RTFM and the power it provides on his blog, and I recommend reading the posts in order from version 0.1 to fully understand it. ([v0.1](http://blog.japaric.io/fearless-concurrency/), [v0.2](http://blog.japaric.io/rtfm-v2/), and [v0.3](http://blog.japaric.io/rtfm-v3/))

Working with RTFM is a walk-in-the-park, and the power it provides is amazing. For example, it provides *compile-time* verification that pins are not being used for multiple functionalities at once, registers can be *frozen* and guaranteed that their values cannot be changed elsewhere in firmware, and if two interrupts use the same peripherals it ensures that they cannot preempt one another and cause race-conditions.

By using RTFM it becomes extremely simple to write firmware in an extremely safe and robust manner, while also maintaining readability. It is definitely something to check out!

### embedded-hal

Finally, we come to the `embedded-hal` crate which defines a number of *traits* that are consistent across various MCUs. These traits are then implemented for specific MCUs, such as the `stm32f30x-hal` crate. The goal of these traits and crates is to allow code to be written which is MCU independent, and allow that code to be easily shared. For instance, write a driver for the BLE module once and be able to use it on *any* MCU which has a `-hal` crate. This is a admirable goal, and will hopefully make embedded development in Rust significantly less fragmented than it is in C or C++.

Currently, however, the traits defined in the `embedded-hal` are not stable and the implementation details of various MCU `-hal` crates is most likely in flux. This will potentially lead to a number of headaches down the road as things change, but I think it is important to contribute to this effort.

One of the interesting things I ran into when working with the `stm32f30x-hal` is that it is no longer possible to go down to the level of the `svd2rust`-generated crates from the board-support firmware. Once the `-hal` crate is brought in, it is expected that *all* register access is done via that crate. This is slightly frustrating because the `stm32f30x-hal` crate is not close to being fully featured. However, this is easily fixed by forking the `-hal` crate and adding functionality where I need it. This makes sense, because people are more likely to upstream additions to the `-hal` crates if they are forced to fork the crate, but it is slightly annoying since the structure of the `-hal` crate *might* change dramatically by next week.


## Prusto Watch Firmware

Given the current state of embedded Rust, there are a few different possibilities for how to write firmware for the Prusto Watch. For instance, I could rely on RTFM + the library provided by `svd2rust` and get very far. However, since a large goal of this project is to hopefully contribute back to the community, the best way to do that is to dive in head-first with the `embedded-hal` traits and specifically the `stm32f30x-hal` crate. I plan on adding all of the functionality I need to my fork of the `stm32f30x-hal` crate, and ideally get them merged upstream. 

Currently the Prusto Watch firmware does basically nothing. My primary goal so far was to ensure that I *could* operate various peripherals rather than getting them fully functional. Below is a list of things I have working so far:

- Blinking an LED using an interrupt driven by a timer.
- Verified that UART can transmit, although getting errors when trying to receive from the BLE module.
- In theory SPI is working, although it I have not been able to get probes to verify that data is being transmitted.

This is obviously not much, but I believe I have a good understanding of the various libraries and tools such that I can meaningfully make progress in the next few weeks.

## Hardware

One of the biggest issues I had was actually verifying that various peripherals were actually functional. This is because I designed the hardware *terribly* for a first-pass. Below is the PCB layout that I created:

![watch v0.1](/images/prusto_watch/prusto_watch_ver1.svg)

Although I was able to get the form-factor extremely close to ideal, I did not put in any methods for debugging. Thus, I ended up splicing wires to various pads in order to get probes connected to *something*. This allowed me to verify that UART was functional, but it was not ideal.

The second iteration of the board will be much, *much* larger. It will also have many header pins for easy probing of basically every peripheral. If I had done this originally I think I could have made significantly more progress actually getting things BLE or the IMU working.

However, the hardware as is has allowed me to verify a number of things. For instance, I know the MCU is functional, the BLE module is functional, and that the power-multiplexing and LI-charging are working. I also learned that the IMU is extremely difficult to get soldered on correctly, which may mean switching to a new package.

## Next Steps

The next steps are to send out a revised board design with a focus on debugability, and to use that new board for actually getting various peripherals functional. I hope to have drivers for the IMU, BLE module, screen, and (hopefully) USB in around three weeks. From there I can focus on writing OS-level functionality and actually making useable smart-watch!
