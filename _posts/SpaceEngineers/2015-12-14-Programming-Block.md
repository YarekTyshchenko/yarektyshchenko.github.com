---
published: true
layout: post
comments: true
---








## Space Engineering - Programming block

Since KeenSWH released the Programming block its now possible to build pretty good automations for your ships
and go as far as building fully automated ships. However, I notice that there is a distinct lack of interface design in the exposed API.

For now I'm going to dump a few bits of code that i'm working on here

### Speed calculator
Shamelessly adapted from a [Gist](https://gist.github.com/awstanley/eb72ef4aa8683b7f0cf3) by [A.W. Stanley](https://gist.github.com/awstanley)
{% gist yarektyshchenko/8d5760966fc1b5d56c20 SpeedCalculator.cs %}

How to use:

{% gist yarektyshchenko/8d5760966fc1b5d56c20 SpeedCalculator_usage.cs %}

I'm still working on the store implementation, but this first iteration should flexible about the persistence method. The ser-de logic is hidden in the struct.

### LCD Panel
{% gist yarektyshchenko/8d5760966fc1b5d56c20 LCD.cs %}

How to use:
{% gist yarektyshchenko/8d5760966fc1b5d56c20 LCD_usage.cs %}

An object should be created for each Panel. The interface is "fluid" so you can chain functions together `lcd.clear().writeLine("foo");` or `lcd.writeLine("foo").writeLine("bar");`, so you don't have to add newlines manually

### Cruise Control
{% gist yarektyshchenko/8d5760966fc1b5d56c20 CruiseControl.cs %}

How to use it:
{% gist yarektyshchenko/8d5760966fc1b5d56c20 CruiseControl_usage.cs %}

The thruster passed in need to be already overridden for this to work.
The naive implementation will just toggle the overridden thruster on and off dependent on the speed passed in.

### Navigator
Navigational Computer is responsible for making maneuvers, changing the orientation of the ship, rather than translating ship's position.
{% gist yarektyshchenko/8d5760966fc1b5d56c20 Navigator.cs %}

Usage:
{% gist yarektyshchenko/8d5760966fc1b5d56c20 Navigator_usage.cs %}

The interface here is somewhat badly designed as I would like to make the navigator work with angles rather than seconds, but as it is it should allow decent automatic control of the ship
