---
layout: post
title:  "Porting QMK to a new cheap keyboard"
tags: hardware keyboards hardware-re 
---

# Introduction

As if I didn't already have too many half finished projects, I wanted to to try out porting [QMK](https://docs.qmk.fm/)  (keyboard firmware) to learn more about QMK and also practice some hardware reverse engineering. This also seemed like a good chance to try out non rubber dome keyboard. 

The first thing I did was search for cheapest mechanical keyboards on Amazon. After looking around and making sure it's not already supported by QMK I chose [PERIBOARD-428](https://perixx.com/products/mechanical-usb-gaming-65-keyboard-customizable-rgb-backlit-n-key-rollover-anti-ghosting) . It cost roughly €30. 

# Initial investigation

After shortly trying it out and observing that typing speed tests were neither better nor worse than with my old keyboard it was time to take it appart. 

On one hand I was happy to find that there is no screw hidden under the glued rubber feet. On the other hadn it meant that I had removed it for nothing. At least it seemed to stick back well enough keyboard came with a pair of spare. Next candidate for hidden screws were the bigger top feet. Those turned out to be magnetic allowing to be turned 90° for some angle adjustment. What an amzing place to hide screws while keeping the accessible, but no luck again.  Last place remaining on the bottom was under the label. Poking it with a screwdriver indicated that there is a round recess in the middle of it, surely a place for screw although it would be slightly weird to have only one screw in the middle of board. But again nothing it was just remains from injection molding process. In the end all the screws were on the top hidden under the keycaps.

![PCB back](/assets/posts/qmk-porting/board_back.jpg)

Taking into account functionality there wasn't too much going on the board. Single diode and RGB led for each switch. Few transistors for driving the LEDs and a single microcontroller. No voltage regulators, no crystal oscillator, no LED driver for all those LEDs.

Searching for the label at the top of PCB "TK568U-RGB" didn't give any interesting results.

Label on the microcontroller was hard to read and looked like "VSI1K09A" and logo with V in a circle with text that might be "Vision". First one didn't give any results. And searching for Vision microcontrollers was mostly filled results about running computer vision on microcontrollers. I also tried search on LCSC with no success. But that didn't discourage me too much, since it dindn't feel too confident in digging up information about more obscure eastern electronics manufacturers. 

Time to dig in the provided software. Maybe that will give some clues. Extracting files from installer using 7zip and looking for interesting strings in the executables lead me to the website of http://www.eevision.com/ . The logo matched with what was etched on the microcontroller, that's a good sign. With the help of Google translate I was able to figure out that they are a company specializing in making gaming peripherals for relabeling by other companies. While this confirmed I am on the right track the fact that Vision is a computer peripheral manufacturer and not a microchip manufacturer meant that the microcontroller is probably relabeled by them and getting actual technical information about will be problematic. 

After looking around on QMK discord I found that many cheaper keyboards are using SONNIX microcontrollers and there is a QMK fork supporting them https://sonixqmk.github.io//SonixDocs/ . Supported [MCU](https://en.wikipedia.org/wiki/Microcontroller) list mentioned eVision VS11K09A-1 which is supposedly rebrand of SONIX SN32F248B MCU. Bingo! I had misread VS11 as VSI1 and after starting at the chip carefully it's probably VS11K09A-1 not VS11K09A. This meant that large portion of the hard work figuring out toolchain and RTOS support for the specific microcontroller is already done. This partially defeats the initial intent, but I wasn't too keen on dealing with that and befor buying keyboard hoped that it will have an MCU from better known manufacturer. I would still have to figure out how the board is wired up, this time that's enough reverse engineering for me.


Poking around with multimeter indicated the switches are connected more or less in a tidy matrix similar to physical layout similar how it's done in DIY keyboards. No crazy optimizations like you would see rubber dome keyboards. 

LEDs are a slightly different story since there are more than one common way for dealing with them. 


# Dumping the original firmware

Not strictly necessary and there probably won't be need to reverse engineering the firmware since the general support for MCU is already figured out, there are no additional ICs on the board and switch and LED connections can be figured out by inspecting the board. But just in case I would still like to dump the firmware and this seem like a good practice with some of the tools.


Like many ARM MCUs it supports SWD interface for reading/writing the program and debugging. https://github.com/SonixQMK/sonix_dumper has more information. First step is identifying the programming pins with the help of datasheets from https://github.com/SonixQMK/Mechanical-Keyboard-Database/tree/main/docs/MCU%20chip and multimeter.

![test pads](/assets/posts/qmk-porting/test_pads.png)

But wait the sonix dumper repository doesn't actually contain any scripts for dumping it promised to contain. After messing with OpenOCD and less complete instructions I found these instructions https://discord.com/channels/805578807477534750/805578807477534754/849069502272110592 . 

There is a script which reuses parts of builtin bootloader ROM for reading the firmware. Not quite sure if reading the flash memory directly failed due to read protection, or it was simply lack of proper configuration. Now that the original firmware is backed up it can be explored further.

One more slightly weird aspect is that for some Sonix MCUs by default trying to read flash memory results in reading of builtin bootloader ROM instead of the firmware data even though from memory map they should be at completely different adresses. To be able to read from the user firmware region it is necesarry to switch the register for choosing interrupt vector table. The datasheet doesn't mention anything that changing it also changes memory map layout. I wonder is it:
* common ARM behavior
* quirk of how boot source/interrupt vector table location change is implemented
* one of the interrupt vector table entries having a special meaning
* interrupts from the bootloader ROM getting executing and doing weird stuff

This can probably be test by configuring the MCU to use the interrupt vector table from RAM. What hap happens if the interrupt table within ram is made to look identical to the one in bootloader ROM? Does whole RAM region gets mapped to address 0x00000000 or does 0x00000000 still point to user firmware in such case?


# Analyzing wiring matrix


Some interesting observations: delete key was on a completely separate column with no other switch in the same column even though there are empty spots in previous columns of the same row.

There is an empty spot for led, maybe an illuminated manufacturers logo in different keyboards reusing the same PCB.

I don't see any obvious current resistors for LEDs. Do those RGB LEDs modules have them builtin, or do they rely on column and row transistors?


# Configuring and building firmware

After all the time analyzing wiring matrix it was almost full success on the first try. All the keys worked as intended except space.