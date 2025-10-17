---
date:
  created: 2025-10-17
  updated: 2025-10-17
title: "Using SD card on Jetson nano"
categories:
  - SBC
tags:
  - Jetson nano

draft: false
---
<!-- more -->



# SD card

I own both a Jetson Nano (the B01 revision) and a Jetson Nano Orin.  They are quite tedious single board computer to setup.

The Jetson Nano does not have a NVME slot.  It boots by default from SD card.  
The Jetson Nano Orin comes with a NVME slot, but still it does not boot from NVME by default.

The kicker here is both require a firmware update to boot from a non-SD card source.  
There are instructions on the net on how to do this; hence my purpose of this post is not about setting these up.

What I want to warn is that although these devices are quick to setup using an SD card image and get started, 
the SD card will *wear out* very quickly.  This is because SD cards are not meant to withstand constant writes by 
the operating system, especially both the Nano and Orin are not really high in memory space and will swap often.

As such, I have faced a couple of times SD card corruption.  It took me a while to figure out what is going on. Although
I know that SD card will wear out, I didn't expect it to wear out so soon - I would say less than 2-3 months of infrequent 
use!

So if you are getting started on the Jetson series, do *quickly* move the setup to an NVME, especially with the newer
Jetsons that support nvme.



