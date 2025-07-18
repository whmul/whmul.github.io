---
name: Marquee Heart Decoration
tools: [PCB Design, SMD Soldering, ATTiny10]
image: https://i.postimg.cc/G2n8Qc88/IMG-20250718-102946.jpg
description: A small, USB powered heart-shaped array of RGB LEDs that cycle their colors in a marquee style, controlled by the ATTiny10.
---

# Heart Decoration

This was a project that I've made as a mother's day gift. It was originally inspired by those wearables or LED badges you sometimes see, but I didn't want to go that route, and instead came up with this stationary version. While the project says it uses the ATTiny10, it initially started with an ATTiny85. More on that later.

---

## Table of Contents
1. [PCB Design & Schematic](#pcb-design--schematic)
2. [PCB Assembly](#pcb-assembly)
3. [Switch and Power](#switch-and-power)
4. [ATTiny10 Retrofit](#attiny10-retrofit)
5. [Conclusion](#conclusion)

---

## PCB Design & Schematic

I had some experience making designing a circuit board and using the attiny at this point, so I just went straight to designing. PCB layout and schematics are shown here, it's not too complicated. 

![Heart light PCB layout](https://i.postimg.cc/L8qJFqjg/heartpcb.png)
*PCB Layout*

![Heart light schematic](https://i.postimg.cc/QxYFzgKr/heartschematic.png)
*Entire Schematic*

The way it works is, the lights cycle their color clockwise, i.e. R-G-B-R-G-B -> G-B-R-G-B-R -> B-R-G-B-R-G. Thus, we have 3 rows of 3 A2SHB mosfets driving each channel, for each position. By choosing where the cathode of each LED connects to these mosfets, it's sufficient to use 3 output pins to drive this whole array.

---

## PCB Assembly

![Blank PCBs](https://i.postimg.cc/fbpjhn3D/IMG-20200213-193349.jpg)
*These PCBs were designed in EasyEDA and manufactured using the service JLCPCB.*

![Soldering Resistors](https://i.postimg.cc/sfpPbL3C/IMG-20200213-200231.jpg)
*Soldering the resistors.*

![Soldering Mosfets](https://i.postimg.cc/L62kSVsY/IMG-20200213-202139.jpg)
*Then the mosfets.*

![Soldering ATTiny85](https://i.postimg.cc/4NP6xhK8/IMG-20200213-202600.jpg)
*And the microcontroller.*

![Everything soldered](https://i.postimg.cc/q7hywb1Y/IMG-20200213-212941.jpg)
*Lastly, the LEDs.*

![Cleaned up 1](https://i.postimg.cc/hGkx5LwG/IMG-20200213-213721.jpg)

![Cleaned up 2](https://i.postimg.cc/tTFPQ1QN/IMG-20200213-213726.jpg)
*After cleaning all the flux.*

---

## Switch and Power

It was here, admittedly, that I decided to improvise. I didn't know exactly how I was going to display this board, so I used a tiny box as a hub, containing the power switch, USB port, and wiring.

![Box](https://i.postimg.cc/PJhDttds/IMG-20200214-000001.jpg)
*Bahar enclosure bmw 50022, 36x36x20mm*

![Everything inside the enclosure](https://i.postimg.cc/QCbTCRMB/IMG-20200214-153829.jpg)

![Side view](https://i.postimg.cc/YqrFZqXQ/IMG-20200214-161015.jpg)
![USB port closeup](https://i.postimg.cc/wTwNp8PZ/IMG-20200214-161021.jpg)
![Running on a battery pack](https://i.postimg.cc/g256xyGK/IMG-20200214-161030.jpg)
*Here we see it powered by a battery pack.*

---

## ATTiny10 Retrofit

Later on, I realized that due to my use of only 3 pins from the original ATTiny85, I could theoretically use an ATTiny10 if I wanted. I decided to take that risk and design and order some of these tiny adapters.

![SOT-to-SOIC adapter layout](https://i.postimg.cc/fLKdRJ57/sottosoic.png)

They adapt the SOIC-8 footprint of the ATTiny85 to the SOT23-6 footprint of the ATTiny10, following the same pin connections, and matching the power and ground pins.


![](https://i.postimg.cc/pTQ9cgz0/IMG-20250718-103100.jpg)
![](https://i.postimg.cc/JhkHZynC/IMG-20250718-102935.jpg)
![](https://i.postimg.cc/G2n8Qc88/IMG-20250718-102946.jpg)

---

## Conclusion

This project was an excellent exercise in surface mount soldering and PCB design. In retrospect, I would like to have planned the mounting solution better. I *am* proud however of the ATTiny10 retrofit. I only did that because I had more of these chips than ATTiny85s, which are actually overkill for this project. 

If you have any questions, feel free to reach out!

---

*Thanks for reading!*
