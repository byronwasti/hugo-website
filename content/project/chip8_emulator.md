---
title: "Chip-8 Emulator"
date: 2017-10-23T09:21:51-05:00
description: A Chip-8 emulator written in Rust with the goal of being extremely flexible.
---

*tldr; Code is here: [https://github.com/byronwasti/chip-8-emulator](https://github.com/byronwasti/chip-8-emulator)*


The Chip-8 is a fairly small instruction set originally designed for playing video games on old microcontrollers. Because of the small instruction set size it is a very approachable computer to write an emulator for. Since I was learning Rust at the time, it seemed like the perfect opportunity to further learn Rust and to learn how to write an emulator.

One of the major goals of this project was to write the core independent of the peripherals. This means that someone could take the core emulator and write their own display or keyboard input system. By default it comes with a keyboard input using stdin and a display using Sdl2.

The second goal of the project was to utilize Rust's error handling system to ensure the code is robust. Thankfully, Rust makes this rather easy to do.

# Examples:

### Brix being played:
![brix](/images/chip8/brix.png)

### Space Invaders Intro:
![space_invaders_intro](/images/chip8/space_invaders_intro.png)

### Space Invaders game:
![space_invaders_play](/images/chip8/space_invaders_play1.png)

### Tic Tac:
![tic_tac](/images/chip8/tic_tac.png)

