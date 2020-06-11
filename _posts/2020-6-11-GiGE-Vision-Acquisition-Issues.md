---
layout: post
title: GigE Vision Acquisition Failed Troubleshoot Guide

---

# Introduction

I had been thinking about starting a developer blog for awhile, but didn't know how I would start it off. I work for a Machine Vision company and write Machine Vision applications with Cognex VisionPro and C#/WPF. Cognex VisionPro is a niche piece of software which many developers will have never heard of unless they work in the field of Machine Vision and therefore there is actually very little on the internet. Since I've been working with this software for more than 4 years. I thought I would start writing this blog to help other engineers that come across the same problems.

# The acquisition failed abnormally. (Buffer retrieve failed (TOO_MANY_CONSECUTIVE_RESENDS)

I spotted a question on reddit asking about an issue with acquiring images from a GiGE camera in [Cognex VisionPro](https://www.cognex.com/en-gb/products/machine-vision/vision-software/visionpro-software) 

> The acquisition failed abnormally. (Buffer retrieve failed (TOO_MANY_CONSECUTIVE_RESENDS)

This is a really annoying issue where you try to acquire an image and VisionPro just doesn't retrieve the image. There is some information on the topic scattered on the internet see the links in [References](#references). Cognex don't include anything in their CHM documentation and best help I could find from Cognex was this [guide](https://support.cognex.com/docs/vpro_950sr2/EN/GigEGuide.pdf).

These are my notes I've made over the last year or so on how to resolve the issue, some of the information has been copied and pasted and slightly rewritten from other sources.

# Theory

## Why GigE vision fails easily

GigE vision sends data over the network via UDP packets. UDP packets are sent from the sender to the receiver with no confirmation that the packet has been received by the receiver. As opposed to TCP where the receiver can request packets dropped by the network. With GigE vision a packet resend request can be issued, but the protocol only allows for a few packets to be resent. Due to the high data rate at which cameras will send data, switches and network interface cards are not always able to keep up with the rate at which data is sent and so they drop the packets which results in the acquisition failing.

Most commonly you will have issues with laptop and USB Ethernet adapters. This can be because the laptop doesn’t support Jumbo frames or only supports upto 4KB jumbo frames and not 9KB. Or your using a USB adapter that supports Jumbo frames, but cannot process the quantity of data arriving and drops frames.

## Why does GigE Vision use UDP and not TCP

GigE vision uses UDP to increase the bandwidth available for data transfer because UDP doesn’t require receive confirmation packets to be sent back to the sender. This allows for shorter transfers times and higher frame rates. The downside is that the protocol is more susceptible to lost of corrupted packets as they cannot be resent continuously until they are received as they can with TCP

# Solution

## Hardware setup

An ideal setup would consist of the following

1. Built in or PCIE Intel Ethernet card (**Intel** cards are specified as the best, avoid **Killer Networks** and **Realtek** cards if you can)

2. Set 9KB Jumbo Frames

3. Hard wired connection, no Wi-Fi as packets will be easily dropped

4. Only use the NIC for camera acquisition, don’t use it for general purpose LAN access for internet or RDP. This data can cause issues with the time sensitive transmission of GigE vision data. (If using multiple cameras with a switch you'll need to setup interpacket delay and frame start delay).

   > Lots of laptops and USB ethernet adapters use Realtek cards and many don't have 9KB jumbo frames and only support 4KB frames. Even if they do support 9KB frames they can't always deal with the large quantity of data and should be avoided if possible.

## Software setup

1. Set **Jumbo Frames** to 9KB

2. Set **Receive Buffers** to highest possible (not always an option)

3. Set **Interrupt moderation** to Extreme (not always an option)

4. Disable all the drivers/services on the network interface other than **IPv4** and the **eBus Universal Pro Driver**

5. Disable any firewalls on that interface

6. Ensure every device supports 9KB jumbo frames. Frame grabbers, switches, POE injectors must all support jumbo frames. Use the command ***ping -l 9000 192.168.x.y*** to test if you can ping your device with a 9KB jumbo frame, this ensures all the devices will work with 9KB jumbo frames

7. If using a managed switch disable any storm control prevention or DDoS attack prevention. These can get confused at shut off the connections as they think its a DDoS or storm.

## My Ethernet adapter doesn’t support 9KB Jumbo

The good news is you can get your camera to acquire images. However you wont be able to acquire at the maximum frame rate of the camera because more packets will be sent from the camera to the PC causing more interrupts and higher CPU usage. Your CPU probably won't be able to keep up and will drop the packets. You can prevent packet dropping by delaying the rate at which the camera sends packets by adjusting the **Interframe packet delay** or **Steam Channel Packet Delay**.

1. Follow all the settings in [Hardware setup](#hardware-setup) and [Software Setup](#software-setup)

2. Go to the **Custom Properties** in the camera settings

3. Add the property **GevSCPD** located at TransportLayerControl -> GigE Vision -> GevSCPD

4. Increase the value until acquisition becomes stable it might take a very high value compared to the default. This value is in ticks and according to [basler](https://docs.baslerweb.com/network-related-parameters-%28gige-cameras%29.html) a tick is worth 8ns. So you might have to increase the value quite high. Just test and see what works.

> **Note** This increases the time delay between each packet and gives switches and NICs more time to process the data. This is more necessary when using small packets as the smaller size packets create 6 times more CPU interrupts (9000/1500) and the CPU cannot keep up with that many packets arriving and will drop the packets. The downside is that this slows down the frame rate of acquisition so might not be suitable for high speed application.

# References
https://support.swingcatalyst.com/hc/en-us/articles/205460708-Gigabit-Ethernet-GigE-Vision-camera-network-recommendations

https://www.flir.com/support-center/iis/machine-vision/application-note/troubleshooting-image-consistency-errors

https://www.flir.co.uk/support-center/iis/machine-vision/application-note/understanding-buffer-handling/

https://www.flir.co.uk/support-center/iis/machine-vision/application-note/troubleshooting-image-consistency-errors/

https://support.cognex.com/docs/vpro_950sr2/EN/GigEGuide.pdf
