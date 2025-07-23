---
name: Geiger Counter
tools: [PCB Design, ATMega368p, SMD Soldering]
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
7. [Code](#code)
8. [Conclusion](#conclusion)

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

## Code

```c
/*
 *  Geiger Counter + OLED Display
 *  Counts pulses on INT-pin 2
 *  Calculates CPM and converts to µSv/h
 *  Shows result on 128×32 SSD1306 display
 *  Hardware: Arduino (AVR), SSD1306 I2C OLED, Geiger tube
*/
#if defined(ESP8266) || defined(ESP32)
    #define ISR_PREFIX IRAM_ATTR
#else
    #define ISR_PREFIX
#endif

#include <Arduino.h>
#include <U8g2lib.h>

/* User configuration */
constexpr uint8_t  GEIGER_PIN         = 2;          // must be interrupt pin
constexpr uint32_t UI_REFRESH_MS      = 1000;       // screen update period
constexpr size_t   SAMPLE_COUNT       = 10;         // moving-average window
constexpr float    USV_PER_CPM        = 0.00569476; // tube conversion factor

/* OLED driver instance */
static U8G2_SSD1306_128X32_UNIVISION_F_HW_I2C display(U8G2_R0);

const uint8_t PROGMEM radiationIcon32x32[] = {
    0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,
    0x00,0x00,0x00,0x00,0x00,0x03,0xC0,0x00,0x80,0x03,0xC0,0x01,0xC0,0x07,0xE0,
    0x03,0xE0,0x0F,0xF0,0x07,0xF0,0x0F,0xF0,0x0F,0xF0,0x1F,0xF8,0x0F,0xF0,
    0x1F,0xF8,0x0F,0xF8,0x3F,0xFC,0x1F,0xF8,0x1F,0xF8,0x1F,0xF8,0xCF,0xF3,
    0x1F,0xF8,0xCF,0xF3,0x1F,0x00,0xC0,0x03,0x00,0x00,0xC0,0x03,0x00,0x00,
    0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0xC0,0x03,0x00,0x00,
    0xE0,0x07,0x00,0x00,0xF0,0x0F,0x00,0x00,0xF0,0x0F,0x00,0x00,0xF8,0x1F,
    0x00,0x00,0xF8,0x1F,0x00,0x00,0xFC,0x3F,0x00,0x00,0xFC,0x3F,0x00,0x00,
    0xF0,0x0F,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00
};

/* Ring buffer for the latest pulse timestamps */

struct PulseBuffer {
    volatile uint32_t stamp[SAMPLE_COUNT]{};  // millis() of each pulse
    volatile uint8_t  head = 0;               // index of newest entry
} pulses;

/* Interrupt service routine */
void ISR_PREFIX onGeigerPulse()
{
    pulses.head = (pulses.head + 1) % SAMPLE_COUNT;
    pulses.stamp[pulses.head] = millis();
}

/* Average time between successive pulses (ms). Returns NAN if not enough data. */
float averagePulseIntervalMs()
{
    noInterrupts();                        // atomic copy of shared data
    uint32_t buf[SAMPLE_COUNT];
    uint8_t  newest = pulses.head;
    memcpy(buf, (const void*)pulses.stamp, sizeof(buf));
    interrupts();

    if (buf[newest] == 0) return NAN;      // not initialised yet

    uint32_t sum = 0;
    size_t   n   = 0;

    for (size_t i = 0; i < SAMPLE_COUNT - 1; ++i) {
        uint8_t curr = (newest + SAMPLE_COUNT - i) % SAMPLE_COUNT;
        uint8_t prev = (curr   + SAMPLE_COUNT - 1) % SAMPLE_COUNT;
        if (buf[prev] == 0) break; // reached empty slot
        sum += (buf[curr] - buf[prev]);
        ++n;
    }
    return n ? (float)sum / n : NAN;
}



void setup()
{
    display.begin();
    pinMode(GEIGER_PIN, INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(GEIGER_PIN), onGeigerPulse, FALLING);

    /* Splash screen */
    display.clearBuffer();
    display.setFont(u8g2_font_9x15_tf);
    display.drawStr(0, 16, "Geiger Counter");
    display.sendBuffer();
}

void loop()
{
    static uint32_t lastUI = 0;
    if (millis() - lastUI < UI_REFRESH_MS) return; // wait for next tick
    lastUI = millis();

    /* 1. Calculate CPM and dose rate */
    float   interval = averagePulseIntervalMs(); // ms
    uint16_t cpm     = (isnan(interval) || interval == 0)
                     ? 0
                     : uint16_t(60000.0f / interval + 0.5f); // round
    float   usvh     = cpm * USV_PER_CPM;

    /* 2. Draw UI */
    display.clearBuffer();

    /* Icon */
    display.drawXBMP(0, 0, 32, 32, radiationIcon32x32);

    /* µSv/h */
    char svLine[16];
    char svVal[8];
    dtostrf(usvh, 4, 1, svVal); // “_._”
    snprintf(svLine, sizeof(svLine), "%cSv/h:%s", 0xB5, svVal);
    display.setFont(u8g2_font_9x15_tf);
    display.drawStr(32, 16, svLine);

    /* CPM */
    char cpmLine[16];
    snprintf(cpmLine, sizeof(cpmLine), "CPM:%u", cpm);
    display.setFont(u8g2_font_8x13_tf);
    display.drawStr(36, 31, cpmLine);

    display.sendBuffer();
}
```

## Conclusion

This project took me through a crash course in high voltage design, battery management, and the practical headaches that come with sensitive analog electronics. With a few burns and a lot of problem-solving, I ended up with a robust, pocketable device that not only works, but looks pretty cool too.

If you have any questions about the details (hardware or software), feel free to reach out!

---

*Thanks for reading!*
