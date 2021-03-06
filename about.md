---
title: About Sim502
feature_image: "/assets/boards.png"
---
Last summer, I used some [Ben Eater](https://eater.net){:target="_blank"} videos, as well as my own knowledge about the 6502 from learning about the NES and SNES, to build a W65C02-based breadboard computer. From there, I've tinkered around with a few additional things, such as attaching a [touchscreen](https://www.adafruit.com/product/1770){:target="_blank"} (which also happens to have an SD card slot on the back) and making a serial protocol to send and receive messages to and from my Raspberry Pi (typing messages with the PS/2 keyboard). I’ve also gotten C code to run on it through [cc65](https://cc65.github.io/){:target="_blank"}. But mostly, I didn’t have too much time for this project during the school year.

Now, though, I have an idea for what I want this to be: I want to make this computer into a smart thermostat. Maybe you think that’s not that interesting of an idea, but a smart thermostat would require basically everything I want to achieve with this machine. It would need to be able to take input and output through a touchscreen, communicate over the Internet, implement general-purpose I/O for the temperature sensor and thermostat outputs, and it should really read its application code from an SD card so the software can be updated remotely.

In this blog, I'm going to be documenting my journey towards implementing this project idea on my 6502. I hope you come along on my journey to make these technical descriptions a reality.

(By the way, if you were wondering what “Sim502” meant, Sim just comes from my name, Simon. Project name definitely subject to change.)
