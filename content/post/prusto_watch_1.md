---
title: "Prusto Watch #1: First Steps into Arm and the Embedded-Rust Ecosystem"
date: 2018-02-12T21:34:57-05:00
tags: []
draft: false
---

In this post I will describe my initial dive into Arm development using Rust. There are potentially many mistakes in my understanding, so this post should not be used as a reference for how to do embedded development in Rust. However, I think it is useful to describe the learning process so that those more experienced can see the pain-points of beginners.

## Firmware Tools

The build system I am using follows japaric's system, which he writes about [here](http://blog.japaric.io/quickstart/). Japaric's post goes into a bit more detail about the tools, but the main ones are `openocd`, the arm build-tools, and `xargo`.

One nice aspect about the tools required for writing firmware in Rust is that they are all open source and all CLI tools. This means that anyone can get access to them and they can easily interface with each other.  The general work-flow I currently have is running `openocd` in the background, compiling with `xargo` and then flashing + debugging with `arm-none-eabi-gdb`. Technically you can combine the last two steps by running `xargo run`, which will call `arm-none-eabi-gdb` automatically.

## Programming for Arm

One of the major things I've been learning about is how to develop on Arm MCUs. I come from a background of AVR MCUs, so the world of Arm was slightly mysterious. However, after diving into a few datasheets and reading various blog posts online I have mostly demystified them.

The biggest difference I found between AVR and Arm is that for Arm you have to manually turn-on various peripherals, such as a GPIO-pin group. This is done in the set of registers under the *Reset and Clock Control* (RCC) group. At first, this was extremely confusing, since the name of RCC is not at all descriptive of this functionality.

The second main difference I found was that GPIO pins had to be set for a specific alternative function. It was not enough to know that a GPIO pin *could* operate SPI, you have to set that GPIO pin to the correct alternate function before it would act as a SPI pin. Thankfully, a lot of the work that is being done in the Rust ecosystem is ensuring that you correctly set up GPIO pins for various peripherals at *compile-time*, which I will talk a little bit about later.

An extremely frustrating aspect of setting up GPIO pins to their alternate functionality is that the alternate function number for each GPIO pin is *not* in the 1141 page reference manual. They are only in the short, 148 page datasheet. This, as far as I can tell, is absolutely ridiculous, and took far too long to figure out.

Overall, working with Arm chips is about the same as working with AVR chips: the datasheet has (basically) everything you need to know.

## Embedded Rust Ecosystem

The embedded ecosystem for Rust is quite young, and there really isn't much out there. However, the libraries and tools available seem to be very well thought-out and highly functional given their current state.

### svd2rust

One of the main tools available is the `svd2rust` program. This is a program that takes in an `svd` file, which is an standardized xml file for describing the peripherals of an Arm device and their registers, and converts it into a Rust library. Since the `svd` files aren't perfect there seems to be some "fixing" of the `svd` files required before the generated Rust library is fully-featured. However, this process only needs to happen once before a crate is available online that provides the library for everyone to use.

Once the library is created, you can manipulate registers in an extremely readable manner. For example, the following code sets up pin PE9 as an output pin:

```
let dp = stm32f30x::Peripherals::take().unwrap();
dp.GPIOA.moder.modify(|_, w| w.moder9().output());
```

Although the modify routine on a register takes a closure, the compiled assembly, according to japaric's tests, is just as fast as modifying the registers by bit-shifting. This is a large improvement over the way things are done in C, since the code is significantly more readable and less error-prone than setting registers using bit masks.

The other nice aspect of the generated Rust library is that it allows us to use Rust to its *full* power. Each register is a common struct, and the ownership model applies. This means Rust will catch, at *compile-time*, unsafe memory patterns and race-conditions. Essentially you have all the guarantees that Rust provides when doing direct register manipulation, which is pretty awesome.

### Real Time For the Masses (RTFM)

RTFM provides a framework for structuring firmware, and making it easy to work with interrupts. Japaric talks extensively about RTFM and what it provides on his blog, and I recommend reading the posts, [v0.1](http://blog.japaric.io/fearless-concurrency/), [v0.2](http://blog.japaric.io/rtfm-v2/), and [v0.3](http://blog.japaric.io/rtfm-v3/). For example, it guarantees that if two interrupts use the same peripherals they cannot preempt one another and cause race-conditions.

Working with RTFM is a pleasure, and for the most part I have not run into many issues. One of the slight annoyances is debugging macro errors, since the entire RTFM "app" is in the form of a macro call. The error messages seem to be getting better with newer nightly releases, but they are still not nearly as friendly as native Rust errors.

By combining RTFM and the library from `svd2rust`, it becomes extremely simple to write firmware in an extremely safe and robust manner while maintaining readability. However, the embedded Rust community has decided that wasn't enough, and have been working on another effort which will increase the ease of use by an order of magnitude.

### embedded-hal

One current issue with firmware development is that there isn't a lot of code sharing. This is partially because there are so many different MCUs, each with their own way of handling peripheral access. To solve this issue, the Rust community is working on a set of traits and trait implementations such that code can be reused much more effectively. The `embedded-hal` crate defines a number of *traits* that are consistent across various MCUs, such as the operations for a digital output pin. These traits are then implemented for specific MCUs, such as the `stm32f30x-hal` crate. The goal of these traits and crates is to allow code to be written which is MCU independent, and allow that code to be easily shared. For instance, write a driver for the BLE module once and be able to use it on *any* MCU which has a `-hal` crate. This will hopefully make embedded development in Rust significantly less fragmented than it is in C or C++.

Currently, however, the traits defined in the `embedded-hal` are not stable and the implementation details of various MCU `-hal` crates is most likely in flux. Using these crates for the Prusto Watch firmware will potentially lead to a number of headaches down the road as things change, but I think it is also important to contribute to this effort of developing reusable code.

One of the interesting things I ran into when working with the `stm32f30x-hal` is that it is no longer possible to go down to the level of the `svd2rust`-generated crates from the Prusto Watch firmware. Once the `-hal` crate is brought in, it is expected that *all* register access is done via that crate. This is slightly frustrating because the `stm32f30x-hal` crate is not close to being fully featured. However, this was easily fixed by forking the `-hal` crate and adding functionality where I need it. Possibly one of the reasons this was done is because people are more likely to upstream additions to the `-hal` crates if they are forced to fork the crate. But it also seems like it might cause a bit of fragmentation across `-hal` crates with different ways things are implemented.

Currently different MCU's `hal` crates seem to have different ways of implementing the same thing. For instance, the `stm32f103xx-hal` crate has the alternate function state of a pin as a single struct which is generic across the different modes, while the `stm32f30x-hal` crate has a separate struct for each alternate function. Hopefully as best-practices are developed, all of the different crates will converge towards on implementation style.

## Prusto Watch Firmware

Given the current state of embedded Rust, there are a few different possibilities for how to write firmware for the Prusto Watch. For instance, I could rely on RTFM + the library provided by `svd2rust` and have everything I would need. However, since a large goal of this project is to hopefully contribute back to the community, the best way to do that is to dive in head-first with the `embedded-hal` traits, and to contribute where I can. I plan on adding all of the functionality I need to my fork of the `stm32f30x-hal` crate, and ideally get them merged upstream. 

Currently the Prusto Watch firmware does basically nothing. My primary goal so far was to ensure that I *could* operate various peripherals, rather than getting them fully functional. Below is a list of things I have working so far:

- Blinking an LED using a `delay` call.
- Blinking an LED using an interrupt driven by a timer.
- Verified that UART can transmit, although I am getting errors when trying to receive from the BLE module.
- In theory SPI is working, although it I have not been able to get probes connected to verify that data is being transmitted.

This is obviously not much, but I believe I now have a good understanding of the various libraries and tools such that I can meaningfully make progress in the next few weeks.

## Hardware

One of the biggest issues I had was actually verifying that various peripherals were actually functional. This is because I designed the hardware *terribly* for a first-pass. Below is the PCB layout that I created:

![watch v0.1](/images/prusto_watch/prusto_watch_ver0.1.svg)

Although I was able to get the form-factor extremely close to ideal, I did not put in any methods for debugging. Thus, I ended up splicing wires to various pads in order to get probes connected to *something*. This allowed me to verify that UART was functional, but it was not ideal.

The second iteration of the board is much, *much* larger. It will also have many header pins for easy probing of basically every peripheral. If I had done this originally I think I could have made significantly more progress actually getting things BLE or the IMU working. Below is the second PCB layout:

![watch v0.2](/images/prusto_watch/prusto_watch_ver0.2.svg)

However, the hardware as is has allowed me to verify a number of things. For instance, I know the MCU is functional, the BLE module is functional, and that the power-multiplexing and LI-charging are working. I also learned that the IMU is extremely difficult to get soldered on correctly.

## Next Steps

The next steps are to send out a revised board design with a focus on debugability, and to use that new board for actually getting various peripherals functional. I hope to have drivers for the IMU, BLE module, screen, and (hopefully) USB in around three weeks. From there I can focus on writing OS-level functionality and actually making a useable smart-watch!

