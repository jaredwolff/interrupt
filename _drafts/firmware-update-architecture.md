---
title: "Firmware Update Architecture"
description: "WIP"
author: francois
---

<!-- excerpt start -->
<!-- excerpt end -->

## Table of Contents

<!-- prettier-ignore -->
* auto-gen TOC:
{:toc}

## Setup

All the code in this post was written for the STM32F429 MCU by ST Micro. While
the examples run fine on the STM32F429i discovery board they were developed in
Renode, a popular MCU emulation platform.

You can find the complete code example for this blog post in the [Interrupt
Github repository](https://github.com/memfault/interrupt/tree/master/example/fwup-architecture)

### Renode

Since writing about Renode for Interrupt, I’ve been looking for an opportunity
to use it for another project. This blog post was the perfect pretext. If you
are not familiar with Renode, I recommend reading [my previous blog post]() on
the topic.

Because I use `bin` rather than `elf` files as firmware images, I had to make two
change to the renode configuration:
1. I used `sysbus LoadBinary $bin 0x8000000` rather than `LoadELF` to load the
   firmware.
2. I manually set the Vector Table Offset with `sysbus.cpu VectorTableOffset
   0x8000000`. By default, Renode looks for the vector table at `0x0` which
different from the default behavior of the STM32.

Additionally, I had to modify Renode slightly to enable software-controlled
resets. Cortex-M microcontrollers can be reset by writing the AICR register,
which was not fully implemented in the emulator. As of this writing, this change
is still in review and not yet merged into the emulator. You can find the pull
request [on Github](https://github.com/renode/renode-infrastructure/pull/15/files).

Thankfully, building our own version of renode is relatively
straightforward using [their
instructions](https://renode.readthedocs.io/en/latest/advanced/building_from_sources.html).

I updated my start.sh script to run my home-built Renode instance rather than
the installed binary:

```
#!/bin/sh

RENODE_EXE_PATH=~/code/renode/output/bin/Release/Renode.exe

mono64 $RENODE_EXE_PATH renode-config.resc
```

You will have to update this script to point at your own `Renode.exe`.

### Toolchain

I used the following tools to build my firmware:
* GNU Make 4.2.1 as build system
* `arm-none-eabi-gcc` version 9.2.1 20191025 (release) as compiler

Rather than the STM32Cube HAL, I used an open source MCU HAL called `libopencm3`
with excellent support for the STM32. I find it easier to use, and like that it
is open source and on Github. The included `Makefile` will clone `libopencm3`
during your first build.

### Building & running the example

The example can be built with Make. From the `examples/fwup-architecture`
directory:

```terminal
$ make
  LD            build/fwup-example-boot.elf
  OBJCOPY       build/fwup-example-boot.bin
  LD            build/fwup-example-app.elf
  OBJCOPY       build/fwup-example-app.bin
  XXD           build/app_bin.c
  LD            build/fwup-example-loader.elf
  OBJCOPY       build/fwup-example-loader.bin
  CAT           build/fwup-example.bin
```

After which you can call `./start.sh` to start Renode. You will need to type the
`start` command in the Renode window to get the emulation going.

![](/img/fwup-architecture/renode-running.png)

## High Level Architecture

Over the years, I've come up with a set of basic requirements most DFU systems
should fulfill. Let's walk through  these one by one and iteratively design our
system.

We start with the simplest possible description of what we want to achieve with
DFU: an application that updates itself.

<style>
.diag1 {
    max-width: 400px;
    margin-left: auto;
    margin-right: auto;
}
</style>
{% blockdiag size:120x40 %}
blockdiag {
   // Set labels to nodes.
   A [label = "Application"];
   A -> A [label = "Updates", fontsize=8];
}
{% endblockdiag %}{:.diag1}

This is not a practical design. For one, self modifying code is easy
to mess up. Let's see how we might modify this architecture to get to something
we're happy with.

### DFU should be separate from the application

The only time I ever broke DFU on a device, I did it without changing a line of
code relating to DFU. Unbeknownst to me, our DFU processes depended on an
uninitialized variable which up until then had always ended up being `0`.
Inevitably a new version reshuffled the content of the stack, and all of a
sudden our uninitialized variable held a "1". This prevented DFU from taking
place.[^chris-dfu-debug]

The moral to this story: keep your DFU process and your application code
separate. Firmware update code is critical and should not be changed unless
absolutely necessary. Separating application code from firmware update code
allows us to update our application code without risking problems with DFU.

How do we modify our architecture to meet this requirement? We simply split the
firmware into a "Loader" and an "Application". The loader verifies the
application, runs it, and can update it.

<style>
.diag2 {
    max-width: 800px;
    margin-left: auto;
    margin-right: auto;
}
</style>
{% blockdiag %}
blockdiag {
    span_width = 100;
    // Set labels to nodes.
    A [label = "App Loader"];
    B [label = "Application"];
    A -> B [label = "Loads, Updates", fontsize=8];
}
{% endblockdiag %}{:.diag2}

### DFU code should be updatable

While we want to update our DFU code as little as possible, updating it should
still be *possible*. Invitably we will find bug in our firmware update code
which we must fix. We may want to change our memory map to allocate more code
space to our app, or to rotate a security key baked into our Loader.

But where should the code lives that updates our Loader? It cannot be in the
application, or else we would violate the previous principle. It cannot be in
the Loader itself either. That leaves one option: a third module tasks with
updating the loader. We'll call it the "Updater".

The Updater is loaded by the Loader, perhaps when a specific input is received.
All it knows how to do is update the Loader.

<style>
.diag3 {
    max-width: 800px;
    margin-left: auto;
    margin-right: auto;
}
</style>
{% blockdiag %}
blockdiag {
    span_width = 100;
    // Set labels to nodes.
    A [label = "App Loader"];
    B [label = "Application"];
    A -> B [label = "Loads, Updates", fontsize=8];

    E [label = "Updater"];
    A -> E [label = "Loads, Updates", fontsize=8];
    E -> A [label = "Updates", fontsize=8];
}
{% endblockdiag %}{:.diag3}

### DFU should use minimal code space

Every firmware project I've ever worked on has run out of code space. At Pebble,
we spent months porting our 3.0 firmware to the original watch; most of that
time was spent slimming down the code so it could fit within the 512KB of Flash
available on that device.

With that in mind, we should make sure our Loader and Updater do not take more
code space than absolutely necessary. Are there ways we can update our design to
use less code space? Absolutely!

The key insight here is that the Updater needs to run very rarely, and that it
never needs to coexist with the application. We can therefore use the same
"slot" in flash for both the Updater and the Application. The main tradeoff here
is that our Loader update flow becomes more complicated:

Go to Loader -> Update the Application with the Updater -> Load Updater ->
Update Loader -> Reboot into Loader -> Replace the Updater with the Application

We can tolerate this complexity because it should not be used often.

{% blockdiag %}
blockdiag {
    span_width = 100;
    // Set labels to nodes.
    A [label = "App Loader"];
    B [label = "Application"];
    A -> B [label = "Loads, Updates", fontsize=8];

    E [label = "Updater"];
    A -> E [label = "Loads, Updates", fontsize=8];
    E -> A [label = "Updates", fontsize=8];

    group {
        label = "Slot 1";
        color = "LightPink";
        A;
    }
    group {
        label = "Slot 2";
        color = "LemonChiffon";
        B; E;
    }
}
{% endblockdiag %}{:.diag3}

### DFU failures should not brick the device

This one should be obvious. Whether there is a bug in our DFU process, or a power loss event while we are
writing firmware, the device should be able to recover. It may operate in a
degraded mode for a bit, but it should at least be able to update itself back to
a good state.

Our design already does a reasonable job of this: if we lose power in the middle
of an Application update, we can reboot into our Loader and start our update
again. There are however two failure modes we must deal with.

First, in the event we lose power while updating the Loader, we could find
ourselves with no valid image at the address the chip boots from (`0x0` by
default for Cortex-M, but aliased to `0x80000000` on STM32). The solution is to
add a small, immutable bootloader whose sole job is to sit at the start address
and load our Loader.

Second, what happens if we find ourselves without a functional Loader? We would
want to fall back to the Updater. Here again, the small bootloader is the
solution. In the event no valid Loader is found, it should try to load whatever
is found in the "Application" slot.

<style>
.diag4 {
    max-width: 1200px;
    margin-left: auto;
    margin-right: auto;
}
</style>
{% blockdiag size:120x40 %}
blockdiag {
    span_width = 100;
    // Set labels to nodes.
    C [label = "Bootloader"];
    A [label = "App Loader"];
    B [label = "Application"];
    C -> A [label = "Loads", fontsize=8];
    A -> B [label = "Ld, Updt", fontsize=8];

    E [label = "Updater"];
    A -> E [label = "Ld, Updt", fontsize=8];
    E -> A [label = "Updates", fontsize=8];
    C -> E [label = "Loads", style=dashed, fontsize=8];

    group {
        label = "Slot 0";
        color = "PaleGreen";
        C;
    }
    group {
        label = "Slot 1";
        color = "LightPink";
        A;
    }
    group {
        label = "Slot 2";
        color = "LemonChiffon";
        B; E;
    }
}
{% endblockdiag %}{:.diag4}

## Implementation Cookbook



## References

- [^chris-dfu-debug]: This is not the end to this story. My cofounder Chris
    eventually found a set of inputs to provide to the device which would set
the stack just so, and allow us to update out of that state. Phew!


