---
title: "Prusto Watch #2: Basic Driver Support"
date: 2018-03-27T13:24:40-04:00
draft: false
---

The goal of the past few weeks has been to develop basic driver support for the various peripherals on the Prusto Watch. This includes the BLE module, the LCD display, the IMU and touch-sensing. So far I have made major progress in developing drivers for the BLE module and for the LCD display. I am having difficulty getting the IMU soldered on correctly, so I have been unable to make progress in actually communicating with it. I have been looking into alternative solutions that will hopefully be easier to solder for future iterations. I have also decided to not pursue touch-sensing, and to instead just use discrete buttons on the outer shell of the Prusto Watch.

## BLE Module ([Github Repo](https://github.com/byronwasti/rn4870))

The first driver I have in a semi-usable state is the RN4870 BLE driver. In terms of functionality the driver can only do the basic necessities, but it is not difficult to add functionality as I need it. One of the hurdles I am running into is figuring out how to deal with incoming UART data on the microcontroller side. Currently the driver exposes a blocking read of the RX pin. One issue with this is that it will block everything until a message has been received. Another issue is that an overrun error is common in case a message comes in when I am not trying to read it.

Below is a screenshot of a phone app which is communicating to the RN4870 device, with firmware running on the microcontroller which provides an "echo" channel.

![BLE echo](/blog_images/prusto_watch_2/ble_echo.png)

So far the "echo" only works if one character is sent at a time from the phone, since the UART peripheral will immediately overrun if more than one character is sent at a time. This is due to the fact that my only way of reading the RX channel is by doing a blocking read. 

There are two potential options that I see to fix this issue. One is to have the UART RX pin cause an interrupt and to fill a circular buffer from the interrupt. The other is to actually get DMA working and to have the UART peripheral pipe in received data into a memory location. Since I know very little about how to get DMA working, I will most likely attempt at getting the interrupt method working. Hopefully this is enough to avoid having constant overruns and to allow non-blocking reading of UART.

The other issue is that the RN4870 seems to be rather finicky to connect to. Sometimes the echo channel is working fine but other times it doesn't work at all. I plan to investigate this issue further once I get the IMU situation sorted out. 

Finally, at some point I will have to look into what is required for an Android phone to send push-notifications over bluetooth. Hopefully there is some simple, default service that Android provides so I don't have to delve too much into Android development.

## LCD Memory Display ([Github Repo](https://github.com/byronwasti/ls010b7dh01))

The second driver I have is for the memory LCD. It took far longer than it should have to get the display to work, mostly because the [datasheet](https://media.digikey.com/pdf/Data%20Sheets/Sharp%20PDFs/LS010B7DH01.pdf) for the display is not entirely clear about how to wire the display. As far as I can tell, the datasheet is actually *incorrect* and says to wire it up backwards.

After many hours of trying to get any sort of communication working with Arduino libraries and a bus-pirate, I found a picture of a breakout board for a memory LCD and upon zooming in, found that it had the connector footprint mirrored from what I had. I desoldered the connector and resoldered it on backwards, and things just worked. 

A good lesson I learned here is if the datasheet is in any way unclear about the footprint, find a reference design or someone else's breakout board.

Below is a demo of the Prusto Watch drawing concentric circles on the display.

![LCD demo](/blog_images/prusto_watch_2/display_demo.gif)

Currently the driver exposes the ability to clear the display, draw individual pixels, draw boxes and draw circles. I plan to clean up the code a little bit at some point, as well as add functionality for drawing text. Otherwise I am very happy with the speed of the library as well as the refresh-rate of the display.

There are a few tricks I used to make the driver memory efficient and quick. One trick was using a lookup-table for converting bytes from MSB to LSB order. The display requires the pixel addresses in LSB order, but currently the `embedded-hal` in Rust has no trait for setting the endianess of SPI communication, so I was stuck with only MSB.

The other trick was having the shadow buffer (in-memory version of what is displayed) store all of the data in a 128x16 (~2K) byte array and using lookup-tables for updating the bits of the shadow buffer based on X and Y values. In this way pixels of the shadow buffer can be changed almost as fast as array indexing. This also made flushing the buffer to the display easy since I could format the data in the way the LCD wanted.

One interesting thing I found while working is that when compiling the code in debug mode, the display lags while updating (you can see the individual lines update one at a time). However, when running in release mode with optimizations, the display behaves as shown above and is extremely quick to update. 

## IMU

The IMU is the last major peripheral I need to write a driver for and I don't expect it to be too much of a problem once I can actually talk to it. However, I have had a lot of difficulty soldering the IMU on correctly.

After being extremely careful with solder-paste, I was able to reflow a board that seemed like the IMU was finally soldered correctly. But when trying to power it on, I found that the board was drawing 10mA when it should only be drawing less than 1mA, since this board only had the IMU and two bypass capacitors on it. It is unclear what exactly was going wrong since I was unable to find any shorts.

Due to the difficulty in soldering my current IMU, I am planning on using a [different chip](https://www.digikey.com/product-detail/en/stmicroelectronics/LIS3DETR/497-16262-1-ND/5799914) for future iterations of the Prusto Watch. It will only be an accelerometer, but it draws less power and should hopefully be easier to solder on. I am also hoping that it is possible to hand-solder, since it would be awesome if this project could be assembled carefully with just a soldering iron.

## Final Thoughts

I am a bit behind schedule in terms of driver development, but I have worked through a number of road-blocks which should mean the rest of the driver development should be quick. I will hopefully not fry any more microcontrollers/boards in the future since the tricky part of the 5V, LCD development is done (playing with 5V on a 3.3V board was a recipe for disaster). 

I plan to have a third revision of the Prusto Watch development board out by late this week or early next week, and ideally it will be in a form factor which can actually be worn (although it will be bulky). In this way I can start working on packaging and getting the various devices working together.

