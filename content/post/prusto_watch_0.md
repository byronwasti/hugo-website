---
title: "Prusto Watch #0: Design"
date: 2017-11-23T09:59:59-05:00
tags: ["prusto-watch"]
draft: false
---

*Code Repo is here: https://github.com/byronwasti/prusto-watch*

*Hardware Repo is here: https://github.com/byronwasti/prusto-watch-hardware*

The Prusto Watch is an Open Source smart watch with firmware written in Rust, or at least it will be that one day. Currently, the Prusto Watch is just a concept. I plan on writing a series of blog posts documenting my progress in developing the watch, with a focus on going through my thought process. A large goal of this project is to help document the current state of embedded development in Rust, since currently there is not a lot out there.

Before I dive into the details, this post will be an overview of what this project will entail.

## Functionality

The Prusto Watch is going to have only basic features of a smart watch, and will not be running a full OS such as Android or iOS. The core features that are needed:

- Bluetooth connectivity: Ideally BLE for simple phone connectivity
- Screen: Low-power OLED or E-ink
- IMU: Accelerometer, gyroscope and magnetometer (for compass functionality)
- Vibration: Small vibration motor for notifications
- User input: Capacitive-touch "ring"

Obviously different users will require different functionality, and since this will be an open source project, anyone is free to fork the project and add additional features. There are also a few features which are pretty broad, such as user input. This could be buttons, it could be a touchscreen, or it could even be voice-processing. For this project I am planning on having a small ring around the screen which will be able to detect the users fingers using capacitive touch.

## Overall Goals of the Project

The project has two overarching goals. 

First, the project will attempt to utilize as much open-source software and hardware as possible. This will lower the bar of entry for people who want to hack on the project. 

Second, the project will attempt to use bleeding-edge Rust embedded best-practices. This will bring a lot of headaches with it, considering the fact that a lot of the embedded work in Rust is rapidly evolving. However, by keeping up with the bleeding-edge design this project will be a good test-case for all of the work done.

