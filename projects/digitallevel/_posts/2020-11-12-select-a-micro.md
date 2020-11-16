---
layout: post
title:  "Select the Hardware"
date:   2020-11-12 09:30:00 -0500
categories: 
tags: embedded hardware stm32 microcontroller
---

[Project on GitHub](https://github.com/christopherdean11/DigitalLevel)  


|[Previous Post][prev_post] | [Next Post][next_post] |

 <!--- START OF CONTENT --->

# Where to start with selecting a microcontroller
I have heard a lot about the STM32 line of microcontrollers and have wanted to look through what they have for a while.  Since my project is quite small, I'm sure any MCU they offer will be more than capable, so I am looking at the STM32x0x0 "Value" lines, specifically the [STM32L0][stmL0], [STM32F0][stmF0], [STM32G0][stmG0]

Things to consider:
* Price
* Package (hand-solder friendly)
* Pin count (enough I/O for the application)
* Peripherals (what is available on each pin)

*The "Four P's" are not to be confused with the "Four C's"*

#### Price
I initially thought about picking an `STM32G0` series for low-cost since they are marketed as the "cost-effective" line, but after looking into it more, the `STM32F0` series is not much more when buying low quantity, seems to be more popular, has extensive documentation, and the `STM32F` and `STM32L` series parts are already library components in Kicad.  

#### Package
I plan to use OSHPark for the fabrication, which means I'll have to solder the components to the board myself. This limited the search as I was looking for some kind of leaded package like a TSSOP, not a QFN or SON.  STM32G0 is only available in leadless packages, so it is out of contention.

#### Pin Count
I want to drive 9 LEDs, have a couple buttons, and I2C.  A 20-pin package is sufficient for my needs on this project and will help keep the cost down.

#### Peripherals
With limited experience in this world, my knowledge of peripherals is currently limited to things like I2C/SPI, ADC, DAC, and GPIOs.  For this project I can get away with I2C and use GPIOs for controlling the LEDs. 


With all of these in mind, I made an initial assumption of the [STM32F030F4][stm_pick], it is availble in 20-pin TSSOP and already exists in KiCad. 

To confirm this choice I created a new project in [STMCubeIDE][cube] to set up the pin assignments.
There are 15 pins available for user configuration of the various peripherals multiplexed to each pin.  I'll set 9 of those (PA0-PA7, PB1) aside as GPIO_Output for LEDs, two (PF0, PF1) as GPIO_Input for the push-button switches, and two (PA9, PA10) for the I2C clock and data lines. Pins  PA13 and PA14 are still available at this point.

<div align="center">
<img src="{{site.url}}/assets/images/CubeMX_InitialPinAssignment_NoSWD.png" height="300">
</div>


####  Programming
Another big item is how to program the MCU.  It looks like the [STM32F030F4][stm_pick] has SWD support (I believe all ARM MCUs have this), so I should be able to connect with ST-LINK using SWD pins.  The [SWD support document][stm_swd] from ST states that SWD should be available if the pins are shown in gray for "reset state" or explicity marked as SWD pins and shown in green, but still recommends explicitly marking them as SWD pins.

<div align="center">
<img src="{{site.url}}/assets/images/CubeMX_InitialPinAssignments.png" height="300">
</div>


This exercise in the STM32CubeIDE confirms that I can continue my design with this device.
In the next post I'll talk about how I designed the PCB for this project.  


# Accelerometer
Since I had to hand-solder this board I needed to buy an acceleromter on a breakout board. I could not find any accelerometer in leaded packages, possibly because the leadless packages are required to get the most out of the sensor.  I ended up selecting the [ADXL343 breakout from Adafruit][accel] for a reasonable price and a nice datasheet from Analog Devices. I also liked that this device has so many additional features including
 - single and double tap detection
 - 2 interrupt outputs to trigger a microcontroller
 - programmable sensitivity

These features offered several interesting opportunities for additional software features for this project.


|[Previous Post][prev_post] | [Next Post][next_post] |
 
 <!--- END OF CONTENT --->

[prev_post]: {% post_url /projects/digitallevel/2020-11-11-DigitalLevel-Intro %}
[next_post]: {% post_url /projects/digitallevel/2020-11-13-pcb-design %}
[stmF0]: https://www.st.com/en/microcontrollers-microprocessors/stm32f0-series.html
[stmG0]: https://www.st.com/en/microcontrollers-microprocessors/stm32g0-series.html
[stmL0]: https://www.st.com/en/microcontrollers-microprocessors/stm32l0-series.html
[stm_pick]: https://www.st.com/en/microcontrollers-microprocessors/stm32f030f4.html
[oshpark]: https://oshpark.com
[stm_cad]: https://www.st.com/en/microcontrollers-microprocessors/stm32f030f4.html#cad-resources
[cube]: https://www.st.com/en/development-tools/stm32cubeide.html
[stm_swd]: https://www.st.com/resource/en/application_note/dm00354244-stm32-microcontroller-debug-toolbox-stmicroelectronics.pdf
[accel]:https://www.adafruit.com/product/4097
