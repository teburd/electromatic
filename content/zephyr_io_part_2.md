+++
title = "Zephyr IO Part 2"
description = "A new IO driver layer for Zephyr"
date = 2019-05-22T11:57:00Z
+++

# Zephyr Sensor IO - Part 2

## Breakdown of Specifics

My specific issues with the current Zephyr API mostly stemmed around three things.
The way triggers were dealt with, the format of the data, and the lack of hardware
FIFO support.

Triggers in the sensor api were dealt with directly as callbacks, either in
queued work or their own zephyr thread. Either way the data wasn't actually
moved from the device to the application in some way until this callback first
occured and the application explicitly requested data be moved. Which didn't
occur until any other interrupts or higher priority tasks were dealt with.
This is where the variable latency problem stemmed from.

The format of the data was not very useful for me to store any significant
amounts of on a microcontroller, as each value was two 32bit integers
representing the whole and fractional parts of the value. The sensor puts
out 16bit values, this micro has SIMD instructions for 16 bit values,
it makes sense to keep them as 16bit values as long as possible. It also
makes it impossible to implement direct DMA transfers from the device to the
application without an intermediary buffer and some sort of translation. Which
wasn't that helpful for my use case of essentially gathering data, doing a few
processing tasks on it, then passing it along over bluetooth.

Lastly the API had no way of directly supporting hardware FIFOs which can
greatly reduce the interrupt and IO burden on the micro leaving it alone
longer to do more useful things, like some math on the samples.

## The Inspiration

To solve most of my problems I need the ability to abstract away a hardware
fifo or simulate one in software when one didn't exist. There began my
journey of creating a buffer api I could be driver impelmented. It turns
out I wasn't the first one to come across this sort of problem, nor
was the Zephyr project, and I'm sure its been around even longer than Linux.

Linux however has a pretty nice solution if you have the whole sysfs 
abstraction to work with in its IIO module. As a microcontroller RTOS sysfs
is sort of beyond the scale of what I think could happen in a few kilobytes of
ram.

