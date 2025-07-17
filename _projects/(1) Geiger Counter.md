---
name: Geiger Counter
tools: [PCB Design, ATMega368p]
image: https://i.postimg.cc/4dkhBM1n/geigerdefault.jpg
description: A fully functional, portable Geiger-Mueller Counter based on the SBM20 GM Tube and an ATMega368p, with an OLED display.
---

# Geiger Counter

I’ve always been fascinated by Geiger counters, partially thanks to the Fallout franchise, so I set out to build a fully functional, portable Geiger-Mueller counter from scratch. After a fair share of prototyping, frustrations, and a great deal of patience, here’s what I came up with.

---

## Table of Contents
1. [Initial Testing & Ideation](#initial-testing--ideation)
2. [The Prototypes: Iterating on High Voltage](#the-prototypes-iterating-on-high-voltage)
3. [High Voltage Module](#high-voltage-module)
4. [Final Electronics Integration](#final-electronics-integration)
5. [PCB and Schematic](#pcb-and-schematic)
6. [Testing & Results](#testing--results)
7. [Conclusion](#conclusion)

---

## Initial Testing & Ideation

The first step was to get reliable CPM (counts per minute) readings and convert them into microsieverts/hr, displayed on a basic OLED display. This was all breadboarded and paired with a microcontroller for speed and flexibility.

![OLED CPM + µSv/h readout test](https://i.postimg.cc/BnyRXcqJ/IMG-20200827-224843.jpg)
*Testing out the CPM reading/display logic on a breadboard.*

---

## The Prototypes: Iterating on High Voltage

### Mark I: Off-the-shelf High Voltage

My first prototype was housed in a project box using an Arduino Nano and an off-the-shelf high voltage module. Unfortunately, the high voltage module didn’t work correctly, so I had to pivot. Frustrated, I decided to design my own high voltage circuit and also ordered an authentic SBM20 GM tube for better sensitivity.

![Mark I - initial prototype using Arduino Nano and commercial HV module](https://i.postimg.cc/LXF7V9Y7/IMG-20200905-203653.jpg)

---

### Mark II: DIY High Voltage (Camera Flash Transformer Hack)

For the Mark II prototype, I used a camera flash transformer to generate the necessary \~400V for the Geiger-Mueller tube, a bit of mixture between hacking and desperation, but it did the trick after a lot of tinkering.

![High voltage module built with a camera flash transformer](https://i.postimg.cc/KjZHd05D/IMG-20200926-140510-01.jpg)

The schematic shows just how much of a challenge high voltage regulation can be. Maintaining a steady supply without frying components or the tube is tricky business.

![Hand-drawn schematic alongside the MKII module](https://i.postimg.cc/d34xvGHP/IMG-20200926-143035-01.jpg)
*The most difficult part of the build: getting stable high voltage!*

The finished Mark II lived in a new project box, now using the authentic SBM20 tube, shielded with copper foil to minimize noise, and powered by a pair of chunky 18650 lithium cells.

![Inside the Mark II with final tube and copper shielding](https://i.postimg.cc/X71zzcB9/IMG-20250717-143111.jpg)
![Mark II again, with visible internal arrangement](https://i.postimg.cc/gk27sBjY/IMG-20250717-143146.jpg)
![Still Mark II, but yes, it works!](https://i.postimg.cc/0y6BRM7F/IMG-20250717-143313.jpg)

---

## High Voltage Module

After some experimenting, I tried driving a CCFL transformer with a 555 timer and stabilizing it with an LM358, breadboarding each revision progressively closer to reliable performance.

![Pre-final HV board - CCFL transformer driven by a 555 timer](https://i.postimg.cc/MGQNRMTV/IMG-20200912-010022.jpg)
![Rough on-board version for pre-final](https://i.postimg.cc/jqX9zrsg/IMG-20200914-185144.jpg)

---

## PCB and Schematic

Eventually, I consolidated everything onto a custom PCB, my favorite part of the process.

![Final PCB design](https://i.postimg.cc/KcDXJM55/GC-PCB.png)

And, for the nitty-gritty, here’s the schematic:

![Full schematic](https://i.postimg.cc/XvNXYCgN/GC-Schematic.png)

### How It Works

- **High Voltage Generation:** Uses a 555 timer (astable mode) to drive a transistor and pulse a CCFL transformer. This boosts the voltage up to the 400V needed by the SBM20, and an LM358 op-amp is used for voltage feedback/stabilization.
- **GM Tube Signal Conditioning:** The Geiger discharge is detected by a transistor circuit and then passed to a second 555 timer (monostable mode) for pulse shaping.
- **Microcontroller & Display:** An ATMega328p counts pulses, does simple µSv/h math, and updates the OLED display.
- **Battery Management:** Integrated Li-Ion charging, protection, and boosted 5V/10V rails.
- **User Interface:** Simple button input, beeper, and LED for feedback.

### Finished Product
![PCB, assembled, showing tube and main components](https://i.postimg.cc/4dkhBM1n/geigerdefault.jpg)

---

## Testing & Results

### Safe Sources

To test the device safely, I used a radium watch hand and thorium dioxide. Both these sources are weak enough to be safe for brief handling but active enough to trigger the tube and show clear readings.

![Radium watch hand & thorium dioxide check sources](https://i.postimg.cc/R0KphksT/IMG-20250717-143506.jpg)

---

## Conclusion

This project took me through a crash course in high voltage design, battery management, and the practical headaches that come with sensitive analog electronics. With a few burns and a lot of problem-solving, I ended up with a robust, pocketable device that not only works, but looks pretty cool too.

If you have any questions about the details (hardware or software), feel free to reach out!

---

*Thanks for reading!*
