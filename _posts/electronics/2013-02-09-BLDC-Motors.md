---
layout: post
category: electronics
title: Driving BLDC Motors
tags: [BLDC, PIC, AVR]
---

Driving sensor-less BLDC motors
===============================

Finding a cheap and versatile solution is harder than it seems. The problem stems from using a motor without sensor, as back-EMF signal is very unpredictable and difficult or even impossible to measure with standard ADC, without comparators.

I had several tries at driving a motor from an old Maxtor HDD that I managed to get out in one piece. First attempt was using a Minimus AVR 32k, simply bitbanging the ports at a set frequency with the coils connected directly. Unfortunately due to the way Minimus is wired up it cannot drain current, only source it (or vice versa, don't remember) and generally having very low driving current the motor didn't move at all, just vibrated slightly, not enough to nudge.

I decided to build a simple 3-phase half-bridge out of mosfets and this time use a PIC18F1320 that I had laying around. This time results were a lot better. The bridge had enough current to not only turn the motor, but after implementing a basic pattern on the ports I managed to get it to spin, quite reliably at low speed. Since I was driving the motor without feedback I knew there would be problems with attaining high speed or under load. I have noticed a strange phenomenon where the rotor would "vibrate" rotationally before settling to the preprogrammed frequency or simply missing a step and getting stuck in wiggle mode.

The solution is obviously to sense when the driving voltage is switched on too soon or too late and adjust the timings accordingly. Since my motor doesn't have any sensors set out to measure Back-EMF to see if i could get the pic to trigger an interrupt on it.

First I used PIC's built in ADC, but as that can't sense negative voltage I didn't get very far. I connected the motor's coils to a soundcard to visualise waveforms to help me understand whats going on and devise a method to check for the zero crossing using the components I had available. Looking at the waveforms I realised that I misunderstood some of the material on BLDCs. the back EMF should be measured against ground, but since my motor is in Y configuration there is no ground connection, and if I find it it will have a signal mixed with the driving signals form mosfets. I've seen a waveform of 3 square peaks, and no gentle zero crossing. I couldn't use this signal to detect when to commutate the motor. Measuring the signal between one of the windings gave the desired output, but that means that PICs ADC could not be used as they all have one Vref. I've been stuck at this point ever since. The solution is to use dedicated comparators and ZCD and feed the pulses into digital input interrupts.

Another avenue I have been looking into is getting a dedicated BLDC driver IC. There is a surprising lack of integrated drivers, all the ones that I found are either in awkward packages or not designed for sensorless motors. One device I managed to get as a sample was the MTD6505 from Microchip, unfortunately its in a UDFN package. The package measures 3mm across and I need to find out what kind of PCB printing process I can use to mount the chip, possibly on a simple and pluggable breakout board, which would be useful even in the final prototype
