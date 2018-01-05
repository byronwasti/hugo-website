---
title: "Adaptive Biasing Differential Amplifier"
date: 2017-05-05T22:16:49-05:00
description: A differential amplifier designed in LTSpice with a slew rate that is dependent on the voltage differential on the inputs.
---

This was my final project for a circuits class. It is the design for an adaptive-biasing differential amplifier. This means that the slew-rate will increase if the differential inputs are far apart from each other, which overall increases the speed of the differential amplifier.

![schem](/images/adaptive_biasing_diff_amp/adaptive-biasing-schematic.png)

The following graph shows the comparison of the slew-rates between the adaptive-biasing differential amplifier, and a differential-amplifier that does not have the adaptive-biasing circuitry (which was from Lab 9).
![slew_rate](/images/adaptive_biasing_diff_amp/slew_rate.png)


The following plot shows the Voltage Transfer Characteristic (VTC) of the adaptive-biasing differential amplifier. Due to the adaptive-biasing, the VTC curve is not completely symmetric and is not easily described by a function.
![VTC](/images/adaptive_biasing_diff_amp/VTC.png)


[Github Link To Projects](https://github.com/byronwasti/CircuitsLabs)
