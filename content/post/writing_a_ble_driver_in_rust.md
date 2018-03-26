---
title: "Writing A Driver in Rust Using Embedded-Hal Traits for the RN4870 BLE Module"
date: 2018-03-04T17:08:02-05:00
draft: false
---
*The repository for the driver is located here: https://github.com/byronwasti/rn4870*

This post will document my process and thoughts on writing a driver for a bluetooth module using Rust and the `embedded-hal` crate. Note that this is not a driver release, as the driver is not complete and will most likely be going through a rewrite due to things I learned while writing the driver.

The specific bluetooth device I will be using is the RN4870 BLE castellated module. It features a simple UART interface and handles most of the complexities of BLE itself, making it very easy to get a simple BLE connection up and running. It also comes in a variety of sizes and packages, as shown below.

![image of BLE module](/blog_images/ble_driver/rn4870_image.jpg)


## Serial Interface

The `embedded-hal` crate defines two traits for working with serial, `Read` and `Write`. Both traits have an associated error type as well as functions which implement serial transmission and reception. Of course the benefit of the HAL trait system is that we don't have to care about the MCU-specific details and we can just use the `read()` and `write()` method calls on structs that implement one or both of these traits.

For writing a driver, I planned on simply having a struct which contains a generic object which implements both the `Read` and `Write` traits.

```rust
use hal::serial::{Read, Write};

pub struct Rn4870<UART> {
    uart: UART,
}

impl<UART> Rn4870<UART>
where
    UART: Write<u8> + Read<u8>,
{
    // Implementation of the driver
}
```

However, this immediately brings up an issue. The `stm32f30x-hal` crate, which implements the HAL traits for the stm32f30x series of microcontrollers (which is the MCU I have available for testing), has the serial interface split into two objects, a TX object and an RX object. In order to do this, the HAL implementation has to use various `unsafe` routines, such as:

```rust
return Ok(unsafe {
    ptr::read_volatile(&(*$USARTX::ptr()).rdr as *const _ as *const _)
});
```

The reason this has to be done is because the TX and RX functionality of UART have the same registers being used, which doesn't really mesh well with the Rust ownership system. However, this unsafe workaround not only looks gross, but it also seems to do away with all of the benefits of the `svd2rust` generated crate which gives us safe access to registers.

So, instead of modifying my crate, I decided to add an implementation of the HAL traits for the `Serial` struct of the `stm32f30x-hal` crate. This implementation is much cleaner looking and avoids unsafe Rust.

```rust
return Ok((self.usart.rdr.read().bits() & 0xFF) as u8)
```

The issue that still remains is that my driver crate requires for the Serial object to be *one* object with both `Read` and `Write` traits implemented. For microcontroller crates with the split TX/RX implementation users will have to add an additional serial HAL implementation which abides by my driver's requirements. What is unclear is whether or not this is the correct way to move forward; should drivers dictate how the HAL traits are implemented, or should there be a standard style of HAL trait implementation?


## Serial Errors

Before the RN4870 starts responding to serial transmission it needs to be reset by pulling the `nRST` pin low for a few milliseconds and then high. The RN4870 will then send a "%REBOOT%" ASCII message over its TX pin. To account for this, I can easily extend my `Rn4870` struct to take in an output pin, and then add a method which takes in a `Delay` object and implements the reset routine.

```rust
use hal::serial::{Read, Write};
use hal::digital::OutputPin;
use hal::blocking::delay::{DelayMs};

pub struct Rn4870<UART, NRST> {
    uart: UART,
    nrst: NRST,
}

impl<UART, NRST> Rn4870<UART, NRST>
where
    UART: Write<u8> + Read<u8>,
    NRST: OutputPin
{
    pub fn reset<DELAY: DelayMs<u16>>(&mut self, delay: &mut DELAY) {
        self.nrst.set_low();
        delay.delay_ms(200u16);
        self.nrst.set_high();
    }
}
```

This worked as expected, and using a digital analyzer I was able to verify that the RN4870 sent a "%REBOOT%" message. However, our driver ought to verify that a reboot occurred; we already have access to the serial interface! 

I modified the `reset()` method to actually verify that the "%REBOOT%" message occurred, and for now we will just `panic!` if it doesn't or any error occurs.

```rust
pub fn reset<DELAY: DelayMs<u16>>(&mut self, delay: &mut DELAY) {
    self.nrst.set_low();
    delay.delay_ms(200u16);
    self.nrst.set_high();

    let expected = [b'%',b'R',b'E',b'B',b'O',b'O',b'T',b'%'];
    for value in expected {
        rec = block!(self.uart.read()).unwrap();
        if rec != value {
            panic!("Invalid value received");
        }
    }
}
```

This implementation has one obvious issue, which is if the RN4870 device never sends any data, we will be blocking forever. I am not entirely sure how to resolve this issue, since ideally we want to just wait a certain amount of time for a response and then emit an error. However, I do not see an easy way to do this, so currently my driver crate only has blocking reads and writes.

The other issue, which I ran into when running the code, is that it immediately has an "Overrun" error, which is when there is unread data in the read register of the UART peripheral and additional data comes in. This is normally not a big issue, and can be easily avoided by reading the entire data stream into memory before doing validation checks. However, I also realized that -- as far as I can tell -- there is no clean way to deal with *any* hardware errors.

## Handling Hardware Errors

First, I am going to back up a step. The stm32f30x family of microcontrollers will throw an "Overrun" error if there is unread data in the `RDR` register of a UART peripheral. When this error occurs, new data is thrown away and the `RDR` register will retain the old data. This error is also a persistent error, meaning it won't resolve itself, even if you try to read from the `RDR` register. The way to resolve the error is to set the `ORECF` bit in the `ICR` register.

First, there are no HAL traits for dealing with serial errors, so my driver has no device-agnostic way of dealing with the overrun error.

Second, the `stm32f30x-hal` crate does not automatically reset the overrun error when it occurs. There is a similar issue for the SPI peripheral of this device, as noted here: [https://github.com/japaric/stm32f30x-hal/issues/13 ](https://github.com/japaric/stm32f30x-hal/issues/13). The proposed solution is to automatically handle the error when it occurs. However, is auto-resetting the error the correct way to handle this situation? Currently nobody other than the OP has responded to the issue, and there is a pull-request which implements the auto-resetting of the SPI errors sitting with no responses. This does not instill confidence in me that this issue will be resolved soon.

So, as far as I can tell, there is no method for handling the overrun error in a clean, device-agnostic way. I decided to implement a workaround that allows users of the driver to have a device-specific handling of errors. To do this, I added a method which takes in a closure which will be run on the UART registers.

```rust
pub fn handle_error<T: Fn(&mut UART) -> ()>(&mut self, func: T) {
    func(&mut self.uart);
}
```

Next, I had to add functionality to the `stm32f30x-hal` crate for clearing the overrun error.

```rust
pub fn clear_overrun_error(&mut self) -> u8 {
    self.usart.icr.write(|w| w.orecf().set_bit());
    (self.usart.rdr.read().bits() & 0xFF) as u8
}
```

And now I can use my error handling routine in my driver like so:

```rust
ble.handle_error(|uart| { uart.clear_overflow_error(); } );
```

However, is this the correct way moving forward? Should driver crates expose safety-hatches for device-specific error routines? Is there a better way to handle errors within the microcontrollers HAL crates?

## Implementing a State Machine

The final thing I want to talk about for my driver is attempting to implement a state machine using the type system of Rust. This state machine is based on the work done here: [https://hoverbear.org/2016/10/12/rust-state-machine-pattern/ ](https://hoverbear.org/2016/10/12/rust-state-machine-pattern/).

My reasoning for using a state-machine is due to the two modes of the RN4870 module. It is either in Command Mode, where you can send different configuration commands, or it is in Data Mode, where it sends UART data over BLE. It would be nice if certain functionality is exposed based on the mode the module is in.

I won't go through all of the small details of my implementation, which can be found in the [tagged driver repository](https://github.com/byronwasti/rn4870/tree/v0.1.0), but I do want to discuss some of the larger issues with having a state machine implemented using the type system.

The general idea behind the type system state machine I used (and is described in the blog post above) is that there are various structs which encompass the various states a system can be in. There is then a main struct (the state machine) which contains one of the various structs. Currently for my driver these are empty structs, but one could easily have data connected with them.

```rust
pub struct CommandMode {}

pub struct DataMode {}

pub struct Rn4870<UART, NRST, S> {
    uart: UART,
    nrst: NRST,
    _state: S,
}
```

In order to implement state-specific functionality, you just have an implementation step which requires the state to be a specific struct. For instance:

```rust
impl<UART, NRST> Rn4870<UART, NRST, DataMode>
where
    UART: Write<u8> + Read<u8>,
    NRST: OutputPin,
{
    // Functionality specific to the DataMode state    
}
```

There are various other ways of implementing a state machine in Rust, but this method gets you a clean interface which hides away the implementation details from the user. However, one of the big issues with a state machine like this is how to handle errors. 

For instance, in order to transition from data mode to command mode for the RN4870 you have to send "$$$" over serial and the RN4870 will then respond with "CMD> ". What if there is a hardware serial error when trying to receive the RN4870 message, and we have no idea if the response was "CMD> " or some other message? Which state are we actually in?

Currently if there is an error during a state transition, my driver will return that error and destroy the `Rn4870` object (since it is consumed by the state-transition method call and only the error is returned). This is not great behavior, and I don't see a great way to rectify the situation given my current implementation.

It would be great to read about other methods for having state machines in drivers, and possibly having a collection of best-practices for writing drivers.

## Final Thoughts

The Rust embedded ecosystem is still very young and it seems like there are still a few big issues with the `embedded-hal` trait system, such as error handling. I think there are a lot of places for less experienced embedded developers (such as myself) to contribute, but it would be nice to have a more organized system for driver submission and feedback. I would also love to beef up the `stm32f30x-hal` crate with additional functionality, but the lack of direction in how to do so and lack of any comments on current pull-requests has made me hesitant.

I also think it would be awesome to have a living best-practices document for writing drivers. This way people new to the embedded-Rust ecosystem can get going much more quickly.

