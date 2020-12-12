---
title:  "Software Design - Part 2: Interrupts"
date:   2020-12-11 21:10:00 -0500
categories: 
tags: embedded software stmcubeide
project_step: 3
header:
  overlay_image: /assets/images/projects/digitallevel/codeimg-cover-photo-2.jpeg
  overlay_filter: 0.7
---

 <!--- START OF CONTENT --->

# Getting started with interrupts on STM32
## What is an interrupt?
An interrupt is a trigger to invoke a function that is called independently of what is happening inside the currently executing function (call it `main` for now). In fact, when the interrupt is triggered the current execution state is saved to a stack so the the interrupt code can be loaded and run, followed by restoring the state back to what it was before the interrupt, well, interrupted it. The function that gets invoked is called a `callback` and can have input arguments but rarely if ever has output arguments; I haven't seen a use case for an output argument yet. It needs to be generic and standalone because the callback is running in a different context to the main function (the one that was interrupted) and there is no way to get an output argument back to the main function because the callback was not invoked by the main function and therefore there is no storage allocated for a result. Therefore, any "output" of the callback function needs to be in the form a state change, either global variable state or phyiscal hardware state.  
This post will demonstrate using interrupts to modify a global state variable `delay` to adjust the time between LED state changes. 

## Enable Interrupts on STM32
1. Open the Device Configuration Tool and modify pins PF0 and PF1 (Port F pin 0 and pin 1) from `GPIO_Input` to `GPIO_EXTI0` and `GPIO_EXTI1` respectively.  EXTI here means External Interrupt, and the 0/1 identifies unique external interrupt sources.  
1. Open the GPIO section inside "System Core" and find PF0 and PF1 at the bottom of the GPIO pin list. Select PF0 and modify the pin configuration to "External Interrupt Mode with Rising edge trigger detection".  
      * We want rising edge detection because the PCB is configured to connect the the GPIO pin to 3V3 when the button is pushed
      * Make sure to also enable the pulldown resistor as well, this will coerce the pin to read low unless driven high externally
      *  Repeat for PF1
1. Finally, navigate to the NVIC section and enable `EXTI line 0 and line 1 interrupts`.

<div align="center">
<img src="{{site.url}}/assets/images/projects/digitallevel/CubeMX_EXTI_Enabled.png" height="300">
</div>

## How to Code an Interrupt
Now that the MCU is configured to respond to the button press and trigger an interrupt, its time to write code that will run each time the button is pushed. Take a look inside the interrupt service routine c file `stm32f0xx_it.c` and you'll see the Interrupt Request (IRQ) Handler:
```c
void EXTI0_1_IRQHandler(void)
{
  /* USER CODE BEGIN EXTI0_1_IRQn 0 */

  /* USER CODE END EXTI0_1_IRQn 0 */
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_1);
  /* USER CODE BEGIN EXTI0_1_IRQn 1 */

  /* USER CODE END EXTI0_1_IRQn 1 */
}
```
which calls the following code:
```c
void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin)
{
  /* EXTI line interrupt detected */
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_Pin) != 0x00u)
  { 
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_Pin);
    HAL_GPIO_EXTI_Callback(GPIO_Pin);
  }
}

/**
  * @brief  EXTI line detection callback.
  * @param  GPIO_Pin Specifies the port pin connected to corresponding EXTI line.
  * @retval None
  */
__weak void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  /* Prevent unused argument(s) compilation warning */
  UNUSED(GPIO_Pin);

  /* NOTE: This function should not be modified, when the callback is needed,
            the HAL_GPIO_EXTI_Callback could be implemented in the user file
   */ 
}
```
So call chain looks like this:  
\<External hardware interrupt occurs\>  
`EXTI0_1_IRQHandler`       
`HAL_GPIO_EXTI_IRQHandler(GPIO_Pin)`   
`HAL_GPIO_EXTI_Callback(GPIO_Pin)`  \<- Callback code goes here

The `EXTI0_1_IRQHandler` function will run when the interrupt is triggered, and it calls `HAL_GPIO_EXTI_IRQHandler` passing the GPIO pin that triggered the interrupt, which in turn does some checking and calls `HAL_GPIO_EXTI_Callback`.  Note in the code snippet the callback function has the `__weak` attribute, which means if another function with a matching prototype (function name and input/output argument types) exists without the weak attribue it will override the `__weak` one.  You can "Open Definition" on `__weak` to see that it is a macro for `__attribute__((weak))` to simplify the syntax/readability a bit.  

In order to implement a callback function, copy the callback function definition without the weak attribute and paste it as a new function into `main.c` and implement the code for button presses.  This is some quick code to toggle the delay between a "fast" and "slow" delay each time the button is pressed:
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin){
    // written as an if-else instead of ternery for readability
    if (delay == 50){
        delay = 250;
    }else{
        delay = 50;
    }
}
```
Remember there is no output.  You can't return anything from a callback because it's called "far away" from the code that was running and has to be self-contained.  In order to "keep" the changes that occur inside the callback, write the changes to a global variable.  
In `main.c` declare `uint32_t delay` above the start of `main()`.
```c
uint32_t delay;
int main(void){
    delay = 200;
    while(1){
        scanLeds(delay);
    }
}
```
It looks like `delay` is going to be `200` and never change. It looks like that to the compiler too, so its best to add the `volatile` keyword to the variable declaration. This tells the compiler what we know as the code author, that the value in `delay` could change "unexpectedly", in our case inside the interrupt callback.  
```c
volatile uint32_t delay;
int main(void){
    delay = 200;
    while(1){
        scanLeds(delay);
    }
}
```
See [https://barrgroup.com/embedded-systems/how-to/c-volatile-keyword](https://barrgroup.com/embedded-systems/how-to/c-volatile-keyword) for more detail about the `volatile` keyword. 

Run the code and realize it doesn't *quite* work as well as it could.  The button press is detected but it doesn't take effect until the main while loop starts over.  

## Fixing the timing issue
The button press only takes effect after the while loop starts over. This happens because the `scanLeds` function is called with `delay` as an argument, which is cached into that execution context. So when the interrupt runs and changes the global, it doesn't take effect until `scanLeds` is called again with an updated value of `delay` to pass into its execution context.  A quick solution for this is to actually remove the parameter from the `scanLeds` function and all of its callees, so the global `delay` is always used. This way every time `delay` is used it will be pulled from the global state, which is updated inside the callback and therefore updates the LED scan rate inside the `scanLeds` function "live", even in the middle of the scan. 
You can also prove this by changing the delay argument name to something else, like `pdelay`, and set a breakpoint inside `scanLeds`. You'll see that `delay`, the global variable, is affected by the callback and `pdelay` is not.
