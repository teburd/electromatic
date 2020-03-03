+++
title = "Rust for i.MX RT"
description = "Safer, faster, better, stronger"
date = 2020-03-03T04:47:00Z

[taxonomies]
categories = ["Rust", "Embedded"]
tags = ["imxrt-rs", "rust", "embedded", "nxp"]

[extra]
author = "Tom Burdick"
relative_posts=[]

+++

## Why Rust

There's dozens of reasons to get Rust on this scorching fast microcontroller,
not least of which is development speed. Nothing kills development speed on
a microcontroller faster than hitting some dreaded hard to find memory or
state issues.
Stack overflows and corruption? No thank you, slows me down. Interrupt 
in the middle of memory critical mutations got you down? Yes, yes they do. 
Unpredicatable timing of interrupt handlers cause your grey matter to melt?
Most definitely. Peripheral configured but not working? Perhaps I need to add
a sleep somewhere, it seems to work sometimes, but not others. Always when
the debugger is looking at it. A heisen bug.

The things that got me excited about Rust on a microcontroller are the same
things that drew me to it in the first place. Saving me incalculable debugging
time by pushing error handling to compile time checks.

## Worst possible Bug Timing

A quick detour on when software tends to fail, at least for me.

Software fails when you need it to work most, every time.

Today is a fantastic example of that. The stock market roared 5% higher than
the previous close on February 28th, which followed with February 29th. Which
meant this year was in fact a leap year.

Robinhood, a stock trading app I myself use occasionally, failed to work today
because it failed to account for a leap year. Wonderful timing! This isn't
a memory safety issue to be sure. It is however a great example of software
failing when you need it most. It isn't my first experience or last experience.

It almost seems like a rule in fact. Software fails when you need it most.

Pulled over and looking for your insurance app on your phone to get your
insurance id? That's the time when the app will inevitably cause your phone to
lockup, meaning you might be locked up.

Streaming a live feed of the world cup, there's the bullet to the goal and...
oh no, there's the loading spinner. Of course that's when it shows up.

About to present your prototype to some eager new investors when you get a
never ending spinner? Yep, its that time again, worst time to fail.

You could argue this is a form of murphy's law, I'd argue its murphy's law
applied to software. Not only do things go wrong, they go wrong at the *worst
possible time*.

Life events can seemingly coincide like that at times too.

## Pushing the errors to the compiler

Lets just quickly checklist the ways in which Rust and its embedded ecosystem
attempts to push errors to compile time.

* Type states with transition functions ensuring going from one state to another.
  https://rust-embedded.github.io/book/static-guarantees/typestate-programming.html
* Using memory safety to avoid multiple mutable references avoids data corruption.
  https://rust-embedded.github.io/book/concurrency/index.html
* Worst case execution time analyzable code with pre-emption and no need for 
  stack management, avoiding a whole slew of common embedded problems.
  https://rtfm.rs/0.5/book/en/

None of these are new and the rust embedded group has great docs covering it all.

It made sense reading all of the above, tinkering on a well supported
stm32f446 and wondering why all of this wasn't well supported on this beastly
microcontroller for NXP, it was clear, it needed to happen while the iron is
hot and the TCP, USB, and audio stacks are coming.

## End Goal, Fuzzy, But Fun

My end goal is something like a i.mx rt running rust to be able to
read from the camera, interpret the sequence of images into a midi controller
note/chord and generate a fun waveform with the display showing something fun.

I think this would be close to fully excercising the capabilities of this chip
in a very fun way. But first it needs to run stuff. So it begins with the help
of others who have already started to pave the way such as Ian McIntyre
who initially ported things to the teensy 4.


https://github.com/imxrt-rs

