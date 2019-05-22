+++
title = "Zephyr RTIO"
description = " A new DMA friendly IO driver layer for sampled input and output for Zephyr"
date = 2019-05-21T11:57:00Z

[taxonomies]
categories = ["Zephyr", "Embedded", "Sensors"]
tags = ["zehpyr", "sensors", "embedded"]

[extra]
author = "Tom Burdick"
relative_posts=[]

+++

## Zephyr

For the past year or so I've been working on ever more complex electronics
devices and firmware. Devices that have greater needs for performance
sensing, processing, and communicating. Specifically I have a chromatography
control system and a sport training device I've been working on.

Writing infinite loop state machines works well to a point. After a certain
level of complexity I'd argue its easier to reason about several state machines
that communicate. I learned that the hard way when creating a soft realtime
control system for chromatography, a story for another time.

An RTOS provides the means to structure a program with communicating state
machines. It does not prevent a devolvement into spaghetti, data race,
bug ridden nightmare fuel.

Zephyr is a fantastic RTOS, with a large test suite covering its core
functionality, soc drivers, and board support packages. 

What it lacked however for both of the projects I've been working on was
a driver API that made sense for my use case. I need to stream compact,
potentially processed by digital filters, sampled data streams. So
sensors. Fast ones!

## The Status Quo

It's worth noting that the Zephyr API for sensors (current as of this
writing still) is quite simple. That's a good thing!

A sample from Zephyr makes it clear whats involved.

[Sensor Sample Snippet](https://github.com/zephyrproject-rtos/zephyr/blob/zephyr-v1.14.0/samples/sensor/magn_polling/src/main.c)
``` c
int ret;
struct sensor_value value_x, value_y, value_z;

while (1) {
    ret = sensor_sample_fetch(dev);
    if (ret) {
        printk("sensor_sample_fetch failed ret %d\n", ret);
        return;
    }

    ret = sensor_channel_get(dev, SENSOR_CHAN_MAGN_X, &value_x);
    ret = sensor_channel_get(dev, SENSOR_CHAN_MAGN_Y, &value_y);
    ret = sensor_channel_get(dev, SENSOR_CHAN_MAGN_Z, &value_z);
    printf("( x y z ) = ( %f  %f  %f )\n",
            sensor_value_to_double(&value_x),
            sensor_value_to_double(&value_y),
            sensor_value_to_double(&value_z));

    k_sleep(500);
}
```

This API is convienent and straightforward.

This API got me quite far with *most* what I was trying to do. In fact
the product at [Baseball Tech](https://baseballtech.com) uses the
Zephyr sensors API in its interrupt driven form though I did have to
do some tweaking.

What I wanted to be able to do was...

* Use the hardware FIFO on many of these devices to free up time on my cpu,
  the Nordic NRF52840 is quite a SoC but time is finite and much of it was
  being used by other things, like bluetooth.
* Not work with already converted values, I wanted the values that the sensor
  gave directly in the native sensor format (usually 16 bit integers) primarily
  because of their compact size.
* Change the state of the device at various times.

Do the above without necessarily tacking on a bunch of custom
device specific functionality like I was doing.

## Reflecting on the first attempt

My first attempt at creating a new API that checked off the bullet list
above was based on my own experience and the opinions described in a few
issues on github.

[Issue 1387](https://github.com/zephyrproject-rtos/zephyr/issues/1387)
[Issue 13718](https://github.com/zephyrproject-rtos/zephyr/issues/13718)

The first attempt really took a lot of inspiration from Linux's IIO API.
After all, Linux often times does things right.

The driver was centered around a typified ringbuffer inspired by linux's
k_fifo, tailored for microcontrollers for streaming data from drivers to
applications. Along with statically defined device and channel attribute
descriptions similiar to IIO, but read and written to using a tagged type
union to represent each attributes value.

[ZIO Branch](https://github.com/bfrog/zephyr/tree/zio)

The API based on IIO however really didn't solve **my** problems well.

DMA usage with a ringbuffer means I would need to have written a memory
allocator on top of it to get contiguous sections. Some form of tracking
when the end of the buffer has been skipped over. A lot more work. I'm lazy
and I wanted something working **now-ish**.

Changing the state of the device left a lot of confusion about the ringbuffer.
At what point in the ringbuffer for example was the device in state 1
rather than say state 2, where state 1 had 2 8 bit values and state 2 
has 4 16 bit signed values and a timestamp? It also left my wondering
how it would ever be possible to set even a simple attribute like
sample rate when that sample rate is often times dependent on several
other configuration options. Filter options, power mode options, fifo
enabled options. So if you write to one attribute, a trigger occurs,
then write to the next attribute, was the data inbetween valid? Did
you know what format it was in, what rate the data was sampled at?

The attributes themselves were not necessarily simple to provide
or work with in either the driver or the application.

Having the driver API require a large number of attributes to be defined,
using a ringbuffer, and other decisions made on the branch turned out to
be in my opinion a dead end for the above reasoning. I'm sorry to those
that started to build on that experiment of mine in earnest. There were a few

[PR 16119](https://github.com/zephyrproject-rtos/zephyr/pull/16119)
[PR 16456](https://github.com/zephyrproject-rtos/zephyr/pull/16456)
[PR 17921](https://github.com/zephyrproject-rtos/zephyr/pull/17921)

Maybe attributes could be salvaged, but the API wasn't really helping me
directly solve my problems. It was, it turns out, a bad API for what I was
trying to do. Hindsight is always of course clearer. I do feel like I should
have seen the issues sooner, but it took writing a driver myself and
seeing a few others starting to write drivers to realize the significant
downsides.

## Second Attempt, A Work in Progress

In my second attempt I focused more on what is the minimum API **I** need to get
the checklist ticked off. The results of that thought experiment lead me
to the following pseudo C sample snippet.

``` c

RTIO_BLOCK_MEMPOOL_ALLOCATOR(blockalloc, 64, 512);
K_FIFO_DEFINE(myfifo);

int main() {
    struct device *mydev = device_get_binding(SENSOR_DEV_NAME);

   /* Using the ST H3LIS3331DL as a sample here */
   struct mysensor_config = {
       .sample_rate = MYSENSOR_1000HZ,
       .scale = MYSENSOR_200G, /* yes this exists in the H3LIS331DL! */
   };
   
   struct rtio_configuration config =  { 
       .output_config = {
           .fifo = myfifo, 
           .timeout = K_FOREVER,
           .byte_size = 512
       },
       .trigger_config = {
           .trigger_source = RTIO_TRIGGER_GPIO,
           .gpio = {
               .device = MYGPIO_DEV,
               .pin = 4,
           }
       },
      .allocator = blockalloc, 
      .driver_config = &mysensor_config
   };
   
   int res = rtio_configure(mydev, &config);
   _ASSERT(res == 0);

   struct rtio_sensor_reader reader;
   while(true) {
       struct rtio_block *block = k_fifo_get(myfifo, K_FOREVER);
       struct rtio_sensor_channel channels[4] = {
           RTIO_SENSOR_TIMESTAMP_CHANNEL(),
           RTIO_SENSOR_CHANNEL(float, SENSOR_ACC_X, 0, SENSOR_SCALED | SENSOR_SI),
           RTIO_SENSOR_CHANNEL(float, SENSOR_ACC_Y, 0, SENSOR_SCALED | SENSOR_SI),
           RTIO_SENSOR_CHANNEL(float, SENSOR_ACC_Z, 0, SENSOR_SCALED | SENSOR_SI),
       };
       res = rtio_sensor_reader(mydev, myblock, &myreader, channels, sizeof(channels));
       _ASSERT(res == 0);
       while(rtio_sensor_reader_next(&myreader)) {
           printf("cycle: %d, acc (m/s^2) x,y,z: %f, %f, %f\n", channels[0].value.u32, channels[1].value.f32, channels[2].value.f32, channels[3].value.f32);
       }
       rtio_block_free(allocator, block);
    }
}
```

Configuration is now a single call to ```rtio_configure```. This includes
a way of allocating contiguous blocks, a place to put blocks that are done being
written to, and when to do so. It also includes what drives the reads, whether
its a GPIO interrupt, hardware timer, or a function all done in a forever loop
with a sleep. All of these options lead to calling```rtio_trigger```. With some
convienence sprinkled in.

To implement this API drivers need to implement three C functions and optionally
define a configuration struct. The first attempt had close to a dozen API calls.

Optionally a driver can extend the streaming API by providing a reader which
lets the application use the device almost exactly like the previous sensor API.
Some additional features this API provides are all channels can be fetched at
once, and the value may be converted to other numerical formats. The results
may be optionally scaled and converted to SI units. There are of course costs to
doing those conversions and providing that functionality.

The end results of this API are that

* The number of functions a driver must implement is reduced.
* DMA transfers are straightforward.
* Losing samples is less likely, so long as there's enough time to process 
  the data at some point and buffers are large enough.
* Latency of data can be tuned as needed by the application with the timeout and
  byte_size output config options.
* Configuration, ordering, and verification of configurations are possible.
* State of the device for data buffers is clear.
* Timestamps, computed or stored, and virtual sensors could be easily provided
  in the blocks, the reader is defined by the driver and channel types can be
  added easily.
* Devices have 2 functions they must implement, and a third optional function if
  the device is like a sensor where you'd like to provide a way to interpret
  the readings semantically.
* Devices have a lot of room to optimize most if not all branching on reading
  out data. Its very sensible that each channel would have a read function
  associated with it.
* Multiple channels of the same physical measurement are possible, the 0 in
  the RTIO_SENSOR_CHANNEL macros is an added identifier describing which
  channel of that type to get data for.
   
This API is a work in progress now on several branches. Work is ongoing
and I hope to have something to show for my efforts for Zephyr 2.2!

Future improvements once the initial API is wrapped up.

* Provide fixed point formats that are DSP instruction friendly on ARM,
  CMSIS DSP in particular has ```q7_t, q15_t, and q31_t```. The Cortex M4
  in particular has several mutiply and accumulate functions for smaller
  types that CMSIS DSP takes advantage of.
* Provide an input_configuration for writing to devices. Many devices
  you can upload firmware to. This would allow both read and write
  functionality. This might mean ```rtio_trigger``` would need to be
  better thought out for reads and writes.
* In the future the thinking is rtio_confgure will take a configuration name
  defined by devicetree. This would avoid the large number of pointers being
  passed from the application to the kernel preventing user mode usage.

Things this API does not do that have been discussed, some of which
would be potentially useful in the future.

* Dynamic devices, its expected your application knows which device its
  using, or atleast one of several devices. exists and could be checked
  by calling ```rtio_configure``` for each.
* Device attributes of any kind, in the future it *may* be sensible to 
  add back in some of the device attributes from the first attempt in a
  more limited, read only, manner.
