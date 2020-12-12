---
title:  "Software Design - Part 3: Interrupts Continued"
date:   2020-12-11 23:08:00 -0500
categories: 
tags: embedded software stmcubeide
project_step: 3
header:
  overlay_image: /assets/images/projects/digitallevel/codeimg-cover-photo-3.jpeg
  overlay_filter: 0.7
---

 <!--- START OF CONTENT --->

# Continuing with interrupts

## Addding Direction Change
Now the goal is to use the other SW button to change the direction the LEDs are scanning. Since we have already learned that we can't pass in the delay, the same will apply to the direction variable.  Add a new global direction variable and modify the callback function. Since the IRQHandler for EXTI0 and EXTI1 is the same (its called `EXTI0_1_IRQHandler`), we aren't going to be making a *new* function for the second button, but modifying the one we already have.  

### Abstracting Low-Level Implementation
Notice in the snippet below I've also refactored the code a bit so now there is a new function `writeLedState`, which accepts `led_id` and `state` as inputs. This abstraction layer lets us focus on a higher-order problem by providing a tool to easily modify an LED state without thinking about *how* to set which LED to set to what state. We'll want this when updating the `scanLedOneState` function for direction change. 
```c
void writeLedState(uint8_t led_id, GPIO_PinState state){
  // Note: this does not check to make sure the 
  // led_id is within the bounds of 1-9
  if (led_id < 5){
    // LED 1-4
    HAL_GPIO_WritePin(GPIOA, 1<<(led_id-1), state);
  } else if (led_id > 5) {
    // LED 6-9
    HAL_GPIO_WritePin(GPIOA, 1<<(led_id-2), state);
  } else{
    // LED 5
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, state);
  }
}
```
### Updating the EXTI Callback
Go back to the `HAL_GPIO_EXTI_Callback` function and add a new case for when it is invoked by GPIO pin 1.
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin){
  if (GPIO_Pin == GPIO_PIN_0){
    if (delay == 50){
      delay = 250;
    }else{
      delay = 50;
    }
  }

  // new: check if GPIO pin 1 caused the interrupt 
  // and flip the state of direction global 
  if (GPIO_Pin == GPIO_PIN_1) {
    direction = 1 - direction;
  }
}
```

### Bring It All Together
Now `writeLedState` can be used in `scanLedOneState` inside a `while` loop that continues until it hits the upper or lower boundary, in this case LED1 or LED9. Changing from a `for` loop to a `while` loop lets the loop continue indefinitely if the direction button is pushed before the first or last LED is reached. 

```c
// globals
uint8_t delay;
uint8_t direction;

void scanLedsOneState(uint8_t state){
  // direction == 1 -> count from 1 to 9
  // direction == 0 -> count from 9 to 1
  int8_t i;
  i = direction ? 1 : 9;
  // sweep either direction and update direction on the fly
  while (i < 10 && i > 0) {
    writeLedState(i, state);
    HAL_Delay(delay);
    i += direction ? 1 : -1;
  }
}
void scanLeds(){
  scanLedsOneState(1);
  scanLedsOneState(0);
}
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin){
  if (GPIO_Pin == GPIO_PIN_0){
    if (delay == 50){
      delay = 250;
    }else{
      delay = 50;
    }
  }
  if (GPIO_Pin == GPIO_PIN_1) {
    direction = 1 - direction;
  }
}

int main(void){
  // set initial values
  delay = 250;
  direction = 0;
  // loop forever
  while (1)
  {
    scanLeds();
  }
}
```
This code works, but once again it leaves a bit be desired from a usability standpoint. When you press the button to change direcions, there is a noticable delay before it actually changes direction.  Why is that? 

## More Debugging
Back to the debugger!  This time the issue is an artifact of how the LEDs are being enabled/disabled with `scanLedOneState`. If you are in in the middle of scanning through the LEDs to enable them, but stop halfway to change direction, it will indeed count backwards, but the state is still high while they are already on, so it appears like nothing is happening while it walks back to the beginning, setting LEDs to a state they are already in.  Once it gets to one end of the LEDs (1 or 9), it returns from `scanLedOneState` to `scanLeds` and calls `scanLedOneState` with the state set low. It starts at the other side by disabling the already disabled LEDs until it gets back to where it was when the button was pushed and starts disabling LEDs that were enabled. Now the direction will appear reversed.  

Imagine the loop just turned on LED6 

|  | D1 | D2 | D3 | D4 | D5 | D6 | D7 | D8 | D9 |
| -- | -- | -- | | -- | -- | -- | -- | -- | | -- |  -- |
| LED states | 1 | 1 | 1 | 1 | 1 | 1 | 0 | 0 | 0 |
| Loop index (state=1) | | | |  | | ^ | | | | 

Then the direction changed

|  | D1 | D2 | D3 | D4 | D5 | D6 | D7 | D8 | D9 |
| -- | -- | -- | | -- | -- | -- | -- | -- | | -- |  -- |
| LED states | 1 | 1 | 1 | 1 | 1 | 1 | 0 | 0 | 0 |
| Loop index (state=1) | | |  | | ^ | | | | | 

Since `state` is still 1 even though direction changed, the loop walks back to LED1 but each LED is already on. Then it gets to the end of `scanLedOneState`, returns to `scanLeds`, and re-enters `scanLedOneState` with `state=0`. The scan now starts at LED9 since direction changed, but LED9 is already at state 0, so no noticable effect takes place until it gets back to LED6.

The solution here is similar to [my last post][last_time]. A new global `led_state` keeps the current state, high or low, of the next LED and is updated when `direction` is updated and the input argument to `scanLedOneState` is removed in favor of using the new global variable inside the function. 



```c
void scanLedsOneState(){
  // direction == 1 -> count from 1 to 9
  // direction == 0 -> count from 9 to 1
  int8_t cur_dir;
  int8_t i;
  i = direction ? 1 : 9;
  // sweep either direction and reflect direction state on the fly
  while (i < 10 && i > 0) {
    cur_dir = direction; // save current direction before LED write
    writeLedState(i, led_state);
    HAL_Delay(delay);
    if (direction!=cur_dir){
      // direction changed since LED write, go back and turn the 
      // last one off. led_state changes with direction so just
      // call writeLedState again
      writeLedState(i, led_state);
    }
    i += direction ? 1 : -1;
  }
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin){
  if (GPIO_Pin == GPIO_PIN_0){
    if (delay == 50){
      delay = 250;
    }else{
      delay = 50;
    }
  }
  if (GPIO_Pin == GPIO_PIN_1) {
    direction = 1 - direction;
    // new: update led_state when direction changes
    led_state = 1 - led_state; 
  }
}

int main(void){
  delay = 250;
  direction = 0;
  // new: assume this was defined in the 
  // /* USER CODE BEGIN PV */ area above with other globals
  led_state = 1;   
  while (1)
  {
    // changed: don't need scanLeds() anymore since 
    // its only purpose was to alternate scanLedsOneState high and low
    scanLedsOneState();         
    // new: update led_state at the end of the main so
    // the LEDs turn off on the next loop if no button presses occur
    led_state = 1 - led_state;  
  }
}
```

## Progress Video
Check out this video to see the progress so far. Next post I'll talk about getting data out of the acceleromter!
<iframe width="560" height="315" src="https://www.youtube.com/embed/VrO6qh6XZE4" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


[last_time]:{{ site.baseurl }}{% link projects/digitallevel/_posts/2020-12-11-software-design-2.md %}#fixing-the-timing-issue