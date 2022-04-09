---
layout: post
title: Smart home display
subtitle: Putting IoT to use
---

A while back I bought some sensors and placed them in my home. They worked fine, but to actually see the data or data history, a specific application or website was needed. While this is easy to do, it still resulted in the sensor data going largely unused.

To actually start to do something with the data, I decided to create a small smart home display for myself. I wanted to work on a project that involved some hardware aspects and having some useful data available all the time seemed like a good starting point.

I bought a 7.5" e-Paper display with black, white, and red colors. While the 'simple' e-Ink displays with only black perform a lot better, the additional red color allows to do some fun things with the display. My end goal initially was to build a fully independent display running wirelessly on an ESP32 chip. To get started quickly however I decided to use a raspberry pi instead which was already up and running anyway as a [Pi-hole](https://pi-hole.net/) instance.

Architecture and code wise the project is extremely simple. There are some processes periodically updating information to data files. Aside from that a small python script is scheduled on a cron job which is responsible for the actual writing to the display.

Every 'box' I defined on the screen has its own responsibility. It will display the data it gets from the data files. Important data and values which are outside the defined norm are displayed in red, so they're easily noticed.

Below you can see the current result. I have yet to model a case to 3D print but in the meantime the display is already doing some nice work!

![Smart Home Display](/img/smart-home-display.jpg)

<br />
<br />
