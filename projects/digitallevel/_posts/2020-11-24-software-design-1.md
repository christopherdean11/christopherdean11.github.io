---
title:  "Software Design - Part 1: Introduction"
date:   2020-11-24 22:05:00 -0500
categories: 
tags: embedded software stmcubeide
project_step: 3
header:
  overlay_image: /assets/images/projects/digitallevel/codeimg-cover-photo-1.jpeg
  overlay_filter: 0.7
---

 <!--- START OF CONTENT --->

# Getting Started Programming STM32
## Initial Setup
I started with the STMCubeIDE device configuration (.ioc file) from the [PCB Design post]({{ site.baseurl }}{% link projects/digitallevel/_posts/2020-11-13-pcb-design-schematic.md %})

## Writing to GPIO
A typical hardware "Hello World" is to blink an LED, proving communication with the device. With a few years of desktop programming under my belt my first attempt was to skip a step and go right into writing patterns to a few LEDs. I quickly realized I was not writing desktop software anymore when changing the LED to light in a loop. The value to write has to be bitshifted, not a plain-old increment when looping. So a for loop to turn a few LEDs on is:
```c
uint32_t delay = 200;
for (uint8_t i=0; i<7;i++){
	// HAL_GPIO_WritePin(GPIOA, i, 1); // this is incorrect
	HAL_GPIO_WritePin(GPIOA, 1<<i, 1);
	HAL_Delay(delay);
}
```
You can see why by looking at the GPIO_PIN defintion by right-clicking on a GPIO pin keyword such as `GPIO_PIN_1` and selcting "Open Definition", where you will find this inside of `stm32f0xx_hal_gpio.h`
```cpp
#define GPIO_PIN_0                 ((uint16_t)0x0001U)  /* Pin 0 selected    */
#define GPIO_PIN_1                 ((uint16_t)0x0002U)  /* Pin 1 selected    */
#define GPIO_PIN_2                 ((uint16_t)0x0004U)  /* Pin 2 selected    */
#define GPIO_PIN_3                 ((uint16_t)0x0008U)  /* Pin 3 selected    */
#define GPIO_PIN_4                 ((uint16_t)0x0010U)  /* Pin 4 selected    */
#define GPIO_PIN_5                 ((uint16_t)0x0020U)  /* Pin 5 selected    */
#define GPIO_PIN_6                 ((uint16_t)0x0040U)  /* Pin 6 selected    */
#define GPIO_PIN_7                 ((uint16_t)0x0080U)  /* Pin 7 selected    */
#define GPIO_PIN_8                 ((uint16_t)0x0100U)  /* Pin 8 selected    */
#define GPIO_PIN_9                 ((uint16_t)0x0200U)  /* Pin 9 selected    */
#define GPIO_PIN_10                ((uint16_t)0x0400U)  /* Pin 10 selected   */
#define GPIO_PIN_11                ((uint16_t)0x0800U)  /* Pin 11 selected   */
#define GPIO_PIN_12                ((uint16_t)0x1000U)  /* Pin 12 selected   */
#define GPIO_PIN_13                ((uint16_t)0x2000U)  /* Pin 13 selected   */
#define GPIO_PIN_14                ((uint16_t)0x4000U)  /* Pin 14 selected   */
#define GPIO_PIN_15                ((uint16_t)0x8000U)  /* Pin 15 selected   */
#define GPIO_PIN_All               ((uint16_t)0xFFFFU)  /* All pins selected */
```
So each GPIO Pin on a given port is its own bit.  This lets you address a single GPIO or multiple GPIOs at once, e.g. to set `GPIO0` and `GPIO1` you can write to `0b11`.  

| GPIO Pin | Hex |  Binary | 
| --- | --- | --- |
| 0 | 0x01 |  0b00000001|
| 1 | 0x02 |  0b00000010|
| 2 | 0x04 |  0b00000100|
| 0 & 1 | 0x03 |  0b00000011 |
{: .reg-tbl}

## Adding Abstraction
Writing to an LED with a targeted GPIO write is one thing, but we really want to be able to think more abstractly about the problem. Instead of having to know which LED corresponds to which GPIO pin every time a state change is needed, its much nicer to write code that lets you think about the problem "one layer up". For now this first layer is a function that simply walks through the LEDs one at a time and turns them on or off with a delay between each LED.  
Remember the LEDs and their pin assignments are:
<div align="center">
<img src="{{site.url}}/assets/images/projects/digitallevel/CubeMX_NamedPinAssignments.png" width="400">
</div>

Here is a function to scan through the line of LEDs in order and set them all on or off with a globally defined `delay` between each LED. The idea here is to use te push-button to change the `delay`. 

```c
void scanLedsOneState(uint32_t delay, uint8_t state){
	// first 4 LEDs
	for (uint8_t i=0; i<4;i++){
		HAL_GPIO_WritePin(GPIOA, 1<<i, state);
		HAL_Delay(delay);
	}
	// LED5, center/green LED is on a different GPIO port
	HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, state);
	HAL_Delay(delay);
	// LEDs 6-9
	for (uint8_t i=4; i<8;i++){
		HAL_GPIO_WritePin(GPIOA, 1<<i, state);
		HAL_Delay(delay);
	}
}
// call scanLeds inside the while(1)
void scanLeds(uint32_t delay){
	scanLedsOneState(delay, 1);
	scanLedsOneState(delay, 0);
}
```

## A Note about Pull-up/down Resistors
While trying to use the onboard switches SW1 or SW2 for GPIO input, I realized I did not include hardware pulldown resistors, the pins are left to float or connected to 3V3. Fortunately there are internal pulldowns that can be turned on inside the MCU.  Maybe I thought of this while designing the board and I just forgot to write it down, or maybe I got lucky with everything that's baked into these MCUs. While reviewing the schematic again after this, I also did not add pullups for I2C, but again there are internal pullups that can be enabled inside the MCU.

<div align="center">
<img src="{{site.url}}/assets/images/projects/digitallevel/CubeMX_I2C_Pullups.png" height="300">
</div>

## Read GPIO - The Naive Implementation
The naive implementation of using a button to change the delay between LEDs might look something like this:
```c
int main(void){
    //...
    while (1){
        if (HAL_GPIO_ReadPin((GPIOF, GPIO_PIN_0))){ 
            scanLeds(100);
        } else {
            scanLeds(500);
        }
    }
}
```
While this technically allows for different LED scan rates by pressing a button, you will quickly notice if you download the code to the microcontroller that it only works if the button is actively being pushed at the top of the while loop, which means you have to time the button push perfectly or hold it down until the right time comes around.  Not a great user experience; we can do better.

The next post introduces solving this problem with interrupts.