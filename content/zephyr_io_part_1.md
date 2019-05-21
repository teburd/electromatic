+++
title = "Zephyr IO Part 1"
description = " A new IO driver layer for Zephyr"
date = 2019-05-21T11:57:00Z
+++

# Zephyr Sensor IO - Part 1

## Background

Being relatively new to the Internet-of-Things on the truely compact, battery
powered, and consumer delivered device, I had quite the task ahead of me.

I needed bluetooth, a decent chunk of memory (for a micro), reasonably fast
IO to support a wide berth of sensors, and battery life that lasts for days.

A quick search and discovery led to the most excellent Nordic Semiconductor
NRF52840 SoC, which has all of the above in spades.

What I didn't expect coming from the world of Atmegas with its relatively
straightfoward avr-libc and bare metal programming was just how big the mental
burden would be to develop on such a device. Bluetooth alone is very much like
a network stack with all of the many issues that come with it, add in the need
for a few other things and it was clear, while the NRF SDK is excellent, it
would take quite some time to develop the application I needed. Then to debug
my one off superloop seemed like a poor route to go. Zephyr was a clear winner
after a quick search. The banner image showcasing my hometown had nothing to do
with Zephyr winning me over, I swear. A quick git clone and I was running the
sample apps on a dev board in less than an hour. Record breaking time as these
things tend to go. 

It was a clear winner from day 1.


## Trouble in Paradise

While most of the Zephyr API's are quite good the sensor API left me scratching
my head a bit as I tried to create my own drivers for the TDK ICM20649, a
6 DoF accelerometer/gyro combo with great ranges and sampling rates for my
use case. This very fast Arm was failing to do the simplest of things, take in
a stream of 6 6 bit samples at 1KHz over SPI and save it to memory. It turns
out that the way the driver API is written leads to a wide variance in latency
from the time the GPIO pin is triggered to the time the SPI transfer even
begins. At first the fix was to use the built in FIFO to simply avoid the
latency issue altogether by allowing a wider variance in interrupt servicing
times, all was wonderful again. I worked around the API issue.

While this was fine at first there was yet another problem. Fusing the data
of this sensor with that of another sensor relative to time became problematic.

It turns out none of these problems were new and an issue from 2017 by
Anas Nashif, the chair of the TSC of Zephyr, described almost exactly the
problems I had ran into. https://github.com/zephyrproject-rtos/zephyr/issues/1387

Short term I found ways of working around the general problem, longer term
I really wanted a better way of using sensors in Zephyr. For this project and
future ideas I have for products.
