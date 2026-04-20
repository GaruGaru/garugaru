+++
author = "GaruGaru"
title = "How I run my homelab"
date = "2026-04-20"
description = "Software and hardware fun building my homelab"
tags = [
    "self-hosting",
    "infrastructure",
]
+++

I've been using my homelab since 1 year now, and I'm starting to be pretty happy about it and by happy I mean that I can stop thinkering and firefighting and actually starting to use it. 

As sole happy user and maintainer I'm going to write this blog post to describe the ideas and the steps behind the creation of my small homelab. 

# Hardware 

Let's start from the physical world, the main drivers behind my hardware choices were the following: 

* **Easiness of replacement**: No fancy obscure harware, everything has to be standard and easly replaciable in case of failures 

* **Power consumption**: Since it will be running in my home I wanted hardware with low power consumption, which translates in lower temperatures and low noise.

* **Form Factor**: I wanted to fit my homelab in a small form factor which I can freely move around and doesn't take an entire room of space, small is beautiful ! 


Given those principles the hardware I'm currently using is:

* 2x Raspberry PI 5 - 8 GB memory
* 1x Raspberry PI 3 - 1 Gb memory 
* 1x Mini itx - intel N100 - 8GB memory - x3 512Gb ssd 

## Rack 

Online we can find many pre-built racks coming in all sizes and shapes, as thinkerers we are never 100% happy about something which comes with all the bells and whistels,so of course here the solution was to build the rack my self.


I've started drawing my own 3d rack trying to fit everything in the smallest space possible, it took a while and more wasted filament that I'd like to admit but in the end it turned out pretty nice.


I'm a happy owner of a BambooLab A1 Mini, which is a great printer but it comes with a small build plate being able to print 18x18x18cm, this posed another challenge since I had to draw everything to be as modular as possible since I wasn't phisically able to print entire parts.


I've realized the simplest yet modular design that I was able to ( I'm not a 3d designer please forgive me ), where the basic idea was to have different **stackable** layers, and each layer with supports to screw in masks so that it could either expose IO ports, fans or other mounts easly.


![Title of image](rack-0.jpg)

