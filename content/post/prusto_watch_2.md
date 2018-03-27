---
title: "Prusto Watch #2: Basic Driver Support"
date: 2018-03-27T13:24:40-04:00
draft: false
---

The goal of the past few weeks has been to develop basic driver support for the various peripherals on the Prusto Watch. This includes the BLE module, the LCD display, the IMU and touch-sensing. So far I have made major progress in developing drivers for the BLE module and for the LCD display. I am having difficulty verifying the IMU is soldered correctly, so I have been unable to make progress in actually communicating with it. I plan on soldering up a new board in the next few days with a focus on getting the IMU correct. I have also decided to not pursue touch-sensing, and to instead just used discrete buttons on the outer shell of the Prusto Watch.

## BLE Module ([Github Repo](https://github.com/byronwasti/rn4870))

The first driver I have in a semi-usable state is the RN4870 BLE driver. In terms of functionality it is fairly bare-bones, but it is not difficult to add functionality when I need it. One of the hurdles I am running into is figuring out how to deal with incoming UART data on the microcontroller side. Currently the driver exposes a blocking read of the RX pin. One issue with this is that it will block everything until a message has been received. Another issue is that overrun is an extremely common error in case a message comes in when I am not trying to read it.

Below is a screenshot of a phone app which is communicating to the RN4870 device, with firmware running on the microcontroller which provides an "echo" channel.

![BLE echo](/blog_images/prusto_watch_2/ble_echo.png)

So far the "echo" only works if one character is sent at a time from the phone, since the UART peripheral will immediately overrun if more than one character is sent at a time. This is due to the fact that my only way of reading the RX channel is by doing a blocking read. There are two potential options that I see to fix this issue. One is to have the UART RX pin cause an interrupt and to fill a circular buffer from the interrupt. The other is to actually get DMA working and to have the UART peripheral pipe in received data into a nice memory location. Since I know very little about how to get DMA working, I will most likely attempt at getting the interrupt method working. Hopefully this is quick enough to avoid having constant overruns, and to allow non-blocking reading of UART.

The other issue is that the RN4870 seems to be rather finicky to connect to. Sometimes the echo channel is working fine but other times it doesn't work at all. Hopefully there is some trick to getting the module to work nicely all the time.

Finally, at some point I will have to look into what is required for an Android phone to send push-notifications over bluetooth. Hopefully there is some simple, default service that Android provides so I don't have to delve into Android development.

## LCD Memory Display ([Github Repo](https://github.com/byronwasti/ls010b7dh01))

The second driver I have is for the memory LCD. It took far longer than it should have to get the display to work, mostly because the [datasheet](https://media.digikey.com/pdf/Data%20Sheets/Sharp%20PDFs/LS010B7DH01.pdf) for the display is not clear about how to wire the display. As far as I can tell, the datasheet is actually *incorrect* and says to wire it up backwards, although since it is not clear it is hard to say the datasheet is wrong. After far too many hours, I tried desoldering the connector and resoldering it on backwards, and thankfully things just worked since I was out of ideas at that point.

Below is a demo of the Prusto Watch drawing concentric circles on the display.

![LCD demo](/blog_images/prusto_watch_2/display_demo.gif)

Currently the driver exposes the ability to clear the display, draw individual pixels, draw boxes and draw circles. I plan to clean up the code a little bit at some point, as well as add functionality for drawing text. Otherwise I am very happy with the speed of the library as well as the refresh-rate of the display.

There are a few tricks I used to make the driver be memory efficient and quick. One trick was using a lookup-table for converting bytes from MSB to LSB order (since the display likes the addresses in LSB order). The other was having the shadow buffer (in-memory version of what is displayed) of the display store all of the data in a 128x16 (~2K) byte array, rather than a 128x128 (~16K) byte array, and using lookup-tables for quick updating of the shadow buffer. This also made flushing the buffer to the display easy since the data was all in the correct format anyway. Since I only have 12K of SRAM to work with, this was a necessary optimization.

One interesting thing I found is that when compiling the code in debug mode, the display lags while updating (you can see the individual lines update one at a time). However, when running in release mode with optimizations, the display behaves as shown above and is extremely quick to update. 

## IMU

The IMU is the last major peripheral I need to write a driver for and I don't expect it to be too much of a problem, since most of the actually communication code will be very similar to that of the LCD (they both use SPI).

## Conclusion

Although I am a bit behind schedule in terms of driver development, I have worked through a number of road-blocks which should mean the rest of the driver development should be quick. In theory I should not be frying any more microcontrollers/boards in the future since the 5V LCD development is mostly good-to-go (playing with 5V on a 3.3V board was a recipe for disaster). I am hoping to have a third revision of the Prusto Watch development board out by late this week or early next week, and ideally it will be in a form factor which can actually be worn (although it will be bulky). In this way I can start working on packaging and getting the various devices working together.

