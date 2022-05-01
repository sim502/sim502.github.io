---
title: Hello, world!
feature_image: "/assets/kandr.png"
---
Hello, world! Last summer, I used some [Ben Eater](https://eater.net){:target="_blank"} videos as well as my own knowledge about the 6502 from learning about the NES and SNES to build a W65C02-based breadboard computer. I tinkered around with a few additional things, such as attaching a [touchscreen](https://www.adafruit.com/product/1770){:target="_blank"} (which also happens to have an SD card slot on the back) and a serial protocol to send messages to my Raspberry Pi using the PS/2 keyboard as well as receive them. I’ve also gotten C code to run on it through [cc65](https://cc65.github.io/){:target="_blank"}. But mostly, I didn’t have too much time for this project during the school year.

Now, though, I have an idea for what I want this to be: I want this to be a smart thermostat. Maybe you think that’s not that interesting of an idea, but a smart thermostat would require basically everything I want to achieve with this computer. It would need to be able to take input and output through a touchscreen, communicate over the Internet, implement general-purpose I/O for the temperature sensor and thermostat outputs, and it should really read its application code from an SD card so the software can be updated remotely.

In light of this project idea, this post is a technical summary of what specific things I want to make happen with the base of my 6502 computer.

(By the way, if you were wondering what “Sim502” meant, Sim just comes from my name, Simon. Name definitely subject to change.)

**Hardware**

<u>Memory Protection</u>
The 6502 has no privilege level system and no memory protection, and given that we want to connect this to the Internet, those are probably going to be necessary. Right now, the memory control schematic looks like this:

<img src="/assets/mem_orig.png" />

We can shut out access to the W65C22 I/O device by ANDing either CS line with the output of an RS-NOR latch that can be reset using a flag on the very same I/O device. But then, how can we ever get access to the I/O back? The I/O can be configured to output a timer signal on one of its pins. We can tie this timer to the latch’s set pin as well as the NMI pin on the 6502 (which we would’ve done anyway for multitasking). This means that exactly at the moment when the I/O becomes available for 6502 use, the 6502 is forced to jump to our predefined kernel code. We’ll also tie the set pin to the 6502’s IRQ pin so that when the 6502 receives an interrupt and jumps to kernel code, that code also has I/O access.

We can also tie the high bits of the RAM address lines to a static output from the I/O. Then, when the I/O becomes inaccessible (during “user mode”), the 6502 code will only be able to access the part of memory that we designated through the I/O during kernel mode – voila, memory protection!

In the end, our updated memory control schematic looks like this:
<img src="/assets/mem_result.png" />
(Bonus! – Before, the highest address bit on the RAM, A14, had to be tied to ground because RAM could only be accessed if A14 was 0, but now we can access all 32KB of RAM!)

<u>Illegal Instruction Protection (part 1)</u>
Because the 6502 has no privilege level system, any code running on it can execute the STP instruction, which stops the 6502 until it gets a RESET signal; even an NMI won’t bring it back. We definitely do not want every task to be able to shut off the computer whenever it wants.

To prevent such an attack, we can place a capacitor that charges from the NOT of the output of the timer. It’s critical to note that the timer can be programmed to keep outputting a signal until acknowledged by the 6502. We can make this acknowledgement at the very beginning of our NMI code (triggered by the timer).

Our capacitor can discharge into RESB on the 6502. This means that if, when the timer turns on, it is immediately acknowledged and turned off, the capacitor will continuously charge and so a reset will not be triggered. However, if the timer stays on for an extended period of time as a result of an illegal STP instruction, the capacitor will discharge fully, triggering a 6502 reset. The reset signal causes kernel code to run, which can then terminate the offending process.

**Software**

<u>Key Drivers</u>
The OS for this machine will need:
* An SD card driver (for reading code and config files)
* A screen + touchscreen driver
* An Ethernet driver
* A digital thermometer driver
* A thermostat output driver

All of the hardware for these drivers should be able to be easily attached to the I/O through breakout boards, etc. However, with so many different tasks that have to be managed by the 6502, we will need a multitasking system.

<u>Multitasking</u>
We've already set up a timer on our I/O devices that sends an NMI interrupt to the 6502. The NMI causes our 6502 to jump to our kernel code, which can handle system calls (if any). Then, to implement preemptive multitasking, the kernel can change the accessible region of RAM in order to run a different task. Before passing control back to a task, the kernel will also decide whether or not to re-enable memory protection. If the next task is a driver, memory protection should not be re-enabled so that the driver can access I/O.

This multitasking system will most likely only have one priority policy, but it won't be round-robin because the number of timeslices allocated to each task will be variable. That is, each task will have a certain number of timer ticks before it gets shut off, and the number of timer ticks allocated to each task can vary. Non-driver tasks, in general, won't be allowed more than the minimum number of timer ticks (a.k.a. time slices) unless they get promoted by a driver task.

<u>Illegal Instruction Protection (part 2)</u>
Another bad instruction that user code could execute would be the STI instruction, which inhibits IRQ interrupts. This instruction isn’t quite as bad as STP. It’s impossible for software to inhibit the timed NMI interrupt that passes control back to the kernel, but IRQs could still be important for our peripherals to communicate to the 6502.

Whether IRQs are enabled or disabled is given by the 6502’s I flag, which is pushed to the stack on NMI. Therefore, the kernel should kill any process when it sees that it has changed the I flag to disable IRQs.

The BRK instruction, which simulates an IRQ in software, could also be a problem. For instance, if we expect the 6502 to be notified of keypresses by an IRQ signal, then it could allow a process to create a “phantom” keyboard input. However, a BRK instruction won’t elevate a process’s RAM privileges, so if the kernel’s IRQ code then tries to write to I/O to deal with a supposed IRQ, it will just clobber the process’s own memory space.

<u>Reading Code & the File System</u>
As mentioned, we want to be able to read and execute code off of an SD card. This means we need a file system.

For the sake of simplicity and because we don’t really need anything more, our filesystem is just going to be flat. Each 512-byte sector will be made up of 507 bytes of data, a 2-byte magic constant, and a 3-byte footer indicating the CHS address of the next sector of the file/directory. If the file ends with that sector, the third byte will be 0x00 to indicate that (sector numbering starts with 1), and the first two bytes will indicate how much of the last sector is included in the length of the file/directory.

The directory will be located at the beginning of the disk and be made up of 16-byte entries. The first two bytes will be a magic constant followed by the file name and extension following the 8.3 format – an 8-byte file name and a 3-byte extension. The last three bytes will represent the CHS address of the file.

It may be more space and time efficient for us if we flash some driver code onto our ROM instead of loading it from the SD card. However, we still want to be able to replace this code easily through the SD card (for instance, because of software updates). The easiest workaround for this is just to include driver executable files that simply jump to the actual driver code in ROM. These files can then be replaced by anyone who has a newer version of the driver and doesn’t want to or can’t reflash the ROM.

**Conclusion**

If you somehow made it all the way through this, thank you! I hope you come along on my journey to make these technical descriptions a reality.
