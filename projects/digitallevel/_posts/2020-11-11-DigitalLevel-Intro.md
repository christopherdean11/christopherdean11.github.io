---
layout: post
title:  "Digital Level Project Introduction"
date:   2020-11-11 09:30:00 -0500
categories: 
tags: embedded
---

[Project on GitHub](https://github.com/christopherdean11/DigitalLevel)

# Project Introduction
## What is it?
A custom PCB with an accelerometer, a row of LEDs, and a couple push-button switches. 

## Why build it?
I want to become more familiar with embedded system development for both hardware and software.  I want to start with a simple design that has just enough hardware to be interesting so I can then find different ways to drive it with software, and presumably come up with incremental improvement ideas along the way I haven't even considered yet.  
I also like the idea of making the PCB myself as opposed to building the circuit on a breadboard with something like a STM32 "blue pill" because it will force me to understand the entire system at a deeper level.  
Plus, its just cool to design something from scratch and hold it in your hand and say "I made this". 

## Goals
Create a PCB from scratch that can display multiple LED patterns and reflect acceleration data.  
In order to do that, we need to:
1. Select a microcontroller and accelerometer
1. Design the PCB
1. Fabricate and populate the PCB
1. Code it up
1. Iterate. Done? What is "done"?


[Next Post][next_post]


[next_post]: {% post_url /projects/digitallevel/2020-11-12-select-a-micro %}