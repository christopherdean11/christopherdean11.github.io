---
title:  "PCB Schematic Design"
date:   2020-11-13 09:30:00 -0500
categories: 
tags: embedded hardware pcb kicad
project_step: 2
header:
  overlay_image: /assets/images/Schematic_MCU_0.png
  overlay_filter: 0.5
---

 <!--- START OF CONTENT --->

# Board Requirements
Before getting started with the design I came up with a list of a few basic requirements for this board:
- As small as reasonably possible to save cost
- A series of LEDs in a row at one end of the board
- SWD programming/debugging pins broken out for easy programming/debugging
- At least 1 push button to cycle through options
- Acceleromter to do I2C comms and display on the LEDs

# Documentation
Here are some refernce documents used during this process:   

Microcontroller:  
- [STM32F030x4 Datasheet][hw_datasheet]
- [RM0360: STM32F030x4 Reference Manual][hw_ref_man]
- [AN4325: Getting started with hardware development for STM32F030xx][hw_dev_doc]
- [AN2606: STM32 microcontroller system memory boot mode][hw_boot_opts]
- [AN4989: STM32 microcontroller debug toolbox][hw_debug_ref]  

Acceleromter:  
- [ADXL343 Datasheet][accel_ds]
- [ADXL343 Breakout Overview][accel_breakout_overview]

#  PCB Design Process
The design process for this board went as follows:
1.  Open CubeIDE to start a project with the STM32F030F4 and set up the pin configurations.
1.  Add the MCU to a new design in Kicad.
1.  Refer to the CubeIDE for the desired pin configuration.  Break out the SWD pins to headers and strap the BOOT pin.
1.  Add some LEDs and a push button and wire it up.
1.  Research and find an acceptable accelerometer.  Low cost I2C capable and hand-solerable.
1.  Research and select a 3.3V regulator to power from 5V.

*A note about power:*  
I wanted to focus on embedded development so I kept the power devliery simple.  I added an 3.3V LDO to allow 5V in from a dedicated header pin with a pair for [+5V, GND], and tied this 3V3 line to the 3V3 rail on the programming header.  This way I  could power it from the programmer or a 5V wall-wart to wire-pair. Decoupling capacitors are based on the recommendations in the [Getting started with hardware development for STM32F030xx][hw_dev_doc] guide.

# Design the schematic
The first step to adding the microcontroller was confirming the pin configuration for the microcontroller in the CubeIDE.  You can actually rename the pins from "GPIO_Output" to more helpful names like "LED1", as shown below.
<div align="center">
<img src="{{site.url}}/assets/images/CubeMX_NamedPinAssignments.png" width="400">
</div>

You may notice the LED numbers do not increment in order.  I did this on purpose.  Since there aren't enough pins available on `PORTA` to fit all 9 LEDs (other pins are reserved for other peripherals), I decided to put the center LED, LED5, on the PORTB pin, and have LED1-4 assigned to `PORTA[0:3]`, and LED6-9 assigned to `PORTA[4:7]`.  

PORTA:  

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
| LED9| LED8| LED7 | LED6 | LED4 | LED3 | LED2 | LED1 |

PORTB:

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
| X | X | X | X | X | X | LED5 | X |

The idea here was that if the LEDs are going to display acceleration where LED5 is 0g, only one side of LED5 is going to be ON at a time, so I might be able to more easily write half a byte to the upper or lower half of `PORTA[7:0]` than if they are split across PORTA and PORTB.  In practice it didn't really matter since I abstracted writing to an LED into a function call anyway.  More on that in the software design post later. 


I addedd the `STM32F030F4Px` to a new KiCad project schematic and labeled the nets for the LED pins so I could draw the LEDs off to the side without crossing wires all over the place.  Then I added nets for I2C clock and data, as well as SWD clock and data.  Next, I added two switches for the GPIO input pins, which will be configured in software to read a logic high when pressed, so the buttons are connected to the 3V3 rail. 
 
## Accelerometer
Since I had to hand-solder this board I needed to buy an acceleromter on a breakout board. I could not find any accelerometer in leaded packages, possibly because the leadless packages are required to get the most out of the sensor.  After following a demo, I had my new symbol and footprint for my [acceleromter of choice][accel], and added this piece to the schematic.

## NRST and BOOT0
I tied `NRST` to the programming header in case the programmer needed to reset the device but realized after board fabrication that I did not include a pullup here to keep it from floating and possibly resetting the microcontroller. The device does however have a 40k<span>&#8486;</span> <!--Ohm--> internal pullup on `NRST` to prevent resets when `NRST` floats.  `BOOT0` was tied to ground through a resistor to always load from flash memory.  
Since the SWD can overtake the state of the microcontroller at any time as long as the SWD pins are not changed to a different peripheral (see section 4.1 of [AN4989][hw_debug_ref]), I don't have to force it to boot into a special state for programming.  

A few finishing touches and a couple 1xN header pins for programming and 5V power later, I had my [schematic][schematic_pdf_github].  
Note that the programming header `J1` was meant to mate with the programming header on the STM32F3Discovery board, which is what I had originally planned to use for programming, but after some issues I ended up just using a USB ST-LINKV2. 

The schematic for this design (also [on GitHub][schematic_pdf_github]):  
<div align="center">
<img src="{{site.url}}/assets/images/led_level_pcb.svg" height="400">
</div>


 <!--- END OF CONTENT --->

[hw_dev_doc]: https://www.st.com/content/ccc/resource/technical/document/application_note/91/66/2d/8c/f9/b5/47/55/DM00089834.pdf/files/DM00089834.pdf/jcr:content/translations/en.DM00089834.pdf
[hw_boot_opts]:https://www.st.com/content/ccc/resource/technical/document/application_note/b9/9b/16/3a/12/1e/40/0c/CD00167594.pdf/files/CD00167594.pdf/jcr:content/translations/en.CD00167594.pdf
[hw_ref_man]:https://www.st.com/resource/en/reference_manual/dm00091010-stm32f030x4x6x8xc-and-stm32f070x6xb-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
[hw_datasheet]:https://www.st.com/resource/en/datasheet/stm32f030f4.pdf
[accel]:https://www.adafruit.com/product/4097
[hw_debug_ref]:https://www.st.com/resource/en/application_note/dm00354244-stm32-microcontroller-debug-toolbox-stmicroelectronics.pdf
[accel_ds]:https://www.analog.com/media/en/technical-documentation/data-sheets/ADXL343.pdf
[accel_breakout_overview]:https://learn.adafruit.com/adxl343-breakout-learning-guide?view=all
[schematic_pdf_github]:https://github.com/christopherdean11/DigitalLevel/blob/master/DigitalLevelSchematic.pdf