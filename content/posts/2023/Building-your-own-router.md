---
title: "Thunderbolt and lightning, very, very frightening"
date: 2023-10-02
draft: true
---

To whoever got that reference congrats your probably old.
So yes in a series of unfortunate incidents, I lost my router to the "Act of God" kind of warranty that is never under warranty. However, this sent me into Panic mode, because I had things planned that involved having internet!

However since it was a long weekend and many people lost their routers too, the local shops were all out of stock and Amazon was going to deliver on Monday.

So I did the next smartest thing I started looking on how to build a router because I've been quite interested in Hardware projects and I had recently purchased a switch so I had that handy.

Bill of Materials:
- OpenWRT
- Raspberry Pi 2b, you could use a zero with a USB ethernet cable too.
- A USB adapter for an extra ethernet port
- A network switch to extend to more connections(Optional)
- A wifi adapter(Optional)

To begin with, head over to the [RaspberryPi page for OpenWRT](https://openwrt.org/toh/raspberry_pi_foundation/raspberry_pi)https://openwrt.org/toh/raspberry_pi_foundation/raspberry_pi

I flashed the SDcard, there are various ways to connect with the raspberry pi, I don't remember exactly how but what I did was give the router a DHCP IP and connect it to my laptop, with this I could further configure the router.

From there it was pretty easy although it took me two days(because I first tried it with Linux), the very first thing I had to do was connect to the ISP via PPPOE, to do this I had to reset the MAC address.

Post that it was quite easy as I had to just setup a LAN section, 
