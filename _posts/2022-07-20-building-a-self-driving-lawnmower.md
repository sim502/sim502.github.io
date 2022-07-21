---
title: "Building an Automated Lawn Mower"
feature_image: "/assets/images/tractor.jpg"
---
So I know that this blog is supposed to be about my 6502 breadboard computer project. While I have been working on a post about that (after first figuring out how to get it to be at all functional again), I’m leaving the country in a couple days. I figured, for now, I’d make a post about another project I’ve been working on that I think is really cool.

My family has this rideable lawn mower, and even though it’s definitely way easier to ride around on this than to push around a normal lawn mower, it’s still kind of annoying to have to sit on it on a hot day in summer. What was interesting about it to me, though, was that it already had a steering mechanism. If I could somehow hijack it, I could potentially make an autonomous, self-driving lawn mower. So that’s exactly what I did (or, at least, have started to do).

**Hardware**

The main steering element is a [$30 linear actuator from Amazon](https://www.amazon.com/dp/B0978XWHWW){:target="_blank"}. Just by attaching the end of the actuator to a point on the arm that connects the steering wheel to the wheel on the ground, I can steer. This is because at the end of the arm, there is a joint that makes the wheel steer right when the arm is extended, and left when it’s retracted.

<p>
  <img src="/assets/images/hardware.jpg" />
  <span class="caption">Actuator connected to the tractor body and the arm</span>
</p>

This method of steering really closely emulates how the actual steering wheel works. The steering wheel is connected to an axle which ends up spinning a gear underneath the lawn mower body. This gear then pushes and pulls the arm in basically the same way as I did with my actuator.

In fact, when my actuator pushes or pulls, it’s strong enough to spin both the gear and the wheel that are attached to the arm, and since it spins the gear, the arm on the other side also gets pushed or pulled. As a result, with just the one actuator, I can steer both front wheels. One side effect of this that I really like is that since the actuator spins the gear that connects to the steering wheel, when I make the actuator extend or contract, the steering wheel also spins – like in a Tesla!

A problem, though, is that this system causes something I call “dead time.” The way the actuator is attached to the tractor body allows it some flexibility to move left and right. Because of that, when the actuator first starts moving, its efforts are put towards moving left and right on its joint rather than pushing the wheel. For instance, when the actuator begins extending, first it will push itself against the joint and get angled more to the right, and then, only when it can’t do that anymore, it will start pushing the wheel. This “dead time,” when the actuator is moving but the wheel isn’t, is super inconvenient for  timing calculations, and I’m planning on fixing it by putting in a ball joint that can’t move left and right but can go up and down. The actuator still has to be able to swing up and down because as the wheel hits peaks and valleys, it and the arm will move up and down, and if the actuator can’t tolerate that, something’s going to break.

**Electronics**

The interface to the actuator consists of a red wire and a black wire (that kind of simplicity is always something I appreciate). Connecting the actuator directly to +12V and ground yields retraction, and reversing the polarity yields extension. There are limit switches to prevent both over-extension and over-retraction, which gave me one less thing to think about.

With that in mind, here’s the schematic I made for the electronics:

<p>
  <img src="/assets/images/schematic.png" />
  <span class="caption">Electronic control system schematic</span>
</p>

Any automation project like this needs a microcontroller, and for this project, I decided to try out the WiFi-equipped NodeMCU ESP8266 board. This was my first experience with the ESP8266, and it was super easy to set everything up with both C and MicroPython. I’m using MicroPython because of its built-in [WebREPL](https://docs.micropython.org/en/latest/esp8266/tutorial/repl.html){:target="_blank"} functionality, which allows sending commands to a Python shell over WiFi, making it super convenient for prototyping and bug-fixing.

The 12V actuator and the 3.3V NodeMCU were going to have to interface through relays. I think this maybe could’ve worked with a single on-off-on DPDT, but since I already had SPDT relays, I just decided to use those. I also couldn’t really find electronics-project-friendly DPDT relays that were definitely on-off-on (though admittedly I didn’t try very hard).

To wire this part up, I plugged each of the actuator’s wires into the common port of a relay. Then I plugged +12V into the NO (normally open) terminal of each relay and 0V into the NC (normally closed) terminals. When neither relay magnet is powered, the actuator gets 0V on both wires; when the relay connected to the actuator’s red wire is powered, the actuator gets directly powered by 12V; and when the relay connected to the black wire is powered, the actuator gets inversely powered.

Finally, we come to the issue of how to actually supply those 12 volts, as well as 5 volts for the NodeMCU. Luckily, the lawn mower’s internal battery happens to be 12V, so I just directly connected it to the relays. I then used a DC-DC buck converter to shift the 12V from the battery down to 5V for the NodeMCU. Upon doing that, though, I realized that at the moment when the actuator first becomes powered through the relays, the load spike causes a drop in voltage throughout the system. This drop resulted in the NodeMCU regularly crashing and resetting. I think a capacitor could help solve this issue, so I ordered a couple of them, but they’re not going to get here until after I leave. For now, I’ve just connected the NodeMCU to an external 9V battery through the buck converter.

<p>
  <img src="/assets/images/battery.jpg" />
  <span class="caption">Wiring attachments to the lawn mower battery</span>
</p>

Here's what all the electronics put together look like so far (the breadboard and the electrical tape, of course, temporary):

<p>
  <img src="/assets/images/electronics.jpg" />
  <span class="caption">On-board electronic control system (white wires come from the lawn mower battery; black wire going into relays comes from actuator)</span>
</p>

**Software**

So the lawn mower isn’t really self-driving yet. It is, however, remotely controlled, which I think is a good start. I wanted it to be controlled through [this cool-looking joystick](https://www.amazon.com/dp/B0002EAA36){:target="_blank"}, so I had to write two pieces of code: the server on the NodeMCU to receive commands and power the relays to cause the appropriate steering, and the client on a laptop to take input from the joystick and relay the appropriate commands to the server.

The server code is pretty simple. It just opens a socket and reads a series of one-byte commands. If the command is “L”, it steers left; it it’s “R”, it steers right; it it’s “N” (for none), it stops steering. It’s also pretty simple for the client code to send commands to the server given a certain joystick input. Here’s the code for the server:

<script src="https://gist.github.com/sjuknelis/666a1f0e78a756968d3533401766efb0.js"></script>

And here's the code for the client:

<script src="https://gist.github.com/sjuknelis/984b57c29d3bad62d1c5aec84a462095.js"></script>

The most complicated part of the software, though, was getting the joystick input in the first place. This was, in large part, because it was surprisingly hard to come by drivers for my joystick. It seems like by far the most popular use cases for these kinds of joysticks are plane simulators or racing games, which tend to have their own proprietary built-in drivers. However, I did manage to find one really reliable general-purpose driver called [Joystick-to-Mouse](https://www.imgpresents.com/joy2mse/j2m.htm){:target="_blank"}, which very generously offers a 100-hour free trial. The only issue for me was that it was for Windows only, and I use a Mac. As a result, I’m using an old Windows laptop for the time being; in the future, I might try setting up a VM so I can control everything directly from my Mac.

As the name implies, Joystick-to-Mouse translates movement on the joystick into mouse movement. In order to capture that movement, I’m using the famous Python library [pyautogui](https://pypi.org/project/PyAutoGUI/){:target="_blank"}. The basic idea is that I use pyautogui to move the mouse to the center of the screen, wait for  1/10th of a second, and then use pyautogui again to get the new position of the mouse. If the mouse moved to the left or right, it must have been a result of joystick input, and so the script sends the appropriate commands. Here's that code:

<script src="https://gist.github.com/sjuknelis/f688c8cce0d4f3d1ae013974065ab063.js"></script>

**What’s next?**

The process of making this lawn mower self-driving is, from here on out, going to be much more software-focused. The main exceptions to that would be fixing some issues like “dead time” and the voltage drop, and adding some features like a compass for more accurate automated steering and possibly speed control using another actuator.

To make it truly self-driving/autonomous, after doing some research, I’ve come up with essentially three methods I could use:
- <u>Record + replay</u> Use the joystick to remotely drive an optimal mowing path and have it measure how long it drives forward for, how long it steers for, etc. Then, to drive itself, it just replays those measurements like a recording. This is by far the simplest method and the most prone to error simply due to slippage. However, I think the compass will help greatly with staying on track. This will probably be the method I’ll start with before trying any others so I can see exactly how big the error is.
- <u>RTK GPS</u> Buy a special real-time kinematics (RTK) GPS and mount it on the lawn mower, and then it can know its position down to the nearest centimeter. In contrast, a normal GPS module has a resolution of around 3 meters, which is pretty much useless when my yard is only 30 meters at its widest. Using an RTK GPS would probably be the most accurate method, but it’s also by far the most expensive (more than likely going into the hundreds of USD). This kind of goes against one of my main principles for this project, which is to limit cost. If my budget were unlimited, I could just buy a self-driving lawn mower and save myself a lot of work; the point in automating this one is to make the most of what I have without spending too much. If the first option doesn’t work, though, I'm going to be seriously considering this one.
- <u>Homemade "GPS"</u> Mount (at least) two distance sensors on fixed points, and have them somehow pivot to track the lawn mower. Then, using the distance measurements, I can pretty accurately find the position of the lawn mower. This uses basically the same principle of the intersection of spheres that GPS uses, except in this case, it's all in 2D and I'm using circles. This could be pretty accurate, but if either sensor ever got even slightly misaligned to where it was no longer pointing at the lawn mower, everything would break. Worse still, if that happened, it’s not like there would be an error message; the lawn mower would simply start receiving garbage data, probably causing very erratic behavior. Because of those factors, this is my least favorite option.

When I get back from my trip, I'm going to continue my work on this project. Once it's in a more finished state, I'll polish up my code, upload it to a Github repo, and then I'll make another post here as the next part to this saga. See you then!

(Also, if you're thinking that the way I split the client code up into two separate files isn't really necessary, you're right – I just did that for demonstration.)