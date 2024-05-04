---
extends: post.html
title: Programming HID devices in macOS with IOKit
date: 2017-09-02
category: programming
tags: ['swift']
external_links:
    - title: WWDC 2011 presentation on userland device access
      link: 'https://developer.apple.com/videos/play/wwdc2011/207'
      description: "Guides on pip's many capabilities in installing, uninstalling, updating and listing Python modules."
publish: false
---

I've pent a bit of time playing around with machine learning and video games, which involved setting up an
emulator for playing Super Nintendo games, which I would then try get neural networks to 'play'. At
this point, I hit not really a technical hitch, but a pause for thought: did I really want to develop a
neural network that's better at games than me? Clearly to allow this work to proceed, I had to brush up
on my classic gaming.

It's perfectly possible to play games on an emulator using the keys on the keyboard. The problem with that
is that using a keyboard is completely different from using a SNES controller, for example. I quickly
discovered this first-hand and found even easy games difficult to play on the keyboard.

I had a quick look around at controllers for personal computers, and found none in any shops near me.
I then recalled an old USB game controller lying in a drawer at home. It resembled an old PlayStation 1
controller, and was much closer to what I needed. The only problem was that, being about 16 years old, it
had been discontinued and the driver was no longer available. Here, I saw an opportunity for a challenge:
would it be possible to write my own driver?

## HID devices
As a USB device, the controller was already accessible through [a standard API](), which meant that at
least I wouldn't need to dabble in raw assembly code. On top of USB, there are libraries like
[libusb]() for interacting with these devices, as well as bindings for other languages. I first tried a
Python binding for libusb called [pyusb]() and was able to get some metadata about the device but no
actual inputs. This turned out to be a result of an incompatibility between macOS and libusb, the whole
debacle of which is documented [here](http://www.libusb.org/ticket/89).

Plan B was to use macOS's built-in functionality for HID devices. A HID (Human Interface Device) is a
physical device that you can use to interact with a computer - things like keyboards, mice, game
controllers and LCD displays. For this, I would have to delve into native macOS development. I had
heard enough about Objective C to be wary of learning it from scratch, however Apple's successor
language, Swift, looked interesting. The syntax is much cleaner than C variants - much like Python with
a little extra core functionality for things like type checking. It also claimed to be suitable for
high-performance applications like games and other programs that run in real time. It was time to try it out!

## Swift and IOKit
After some preliminary searching, I found a [presentation]() given at an Apple WWDC on interacting with HID
devices using macOS's IOKit library. This gave a good overview of the relevant macOS infrastructure,
and even went through an example of writing a driver for a gaming keyboard. The only problem was that all
of the example code was in Objective C, but even that wasn't too big a deal, as Swift is able to use
existing Objective C libraries. I also found an existing example of using IOKit with Swift, which the author
says was tricky to get working.

## Discovering the device
First, we need some information about the device. I was able to get this in Python with
[pyhidapi](https://github.com/apmorton/pyhidapi) combined with a custom compiled version of the `hidapi` C
library, which worked but was probably more complicated than it needed to be. Regardless of what USB library
you use, you should get some summary information about your HID device:

```
{
    "product_string": "Some kind of controller",
    "vendor_id": 1234,
    "manufacturer_string": "Some company that made this thing",
    "product_id": 56789,
    # ...
}
```

The bits we need are the vendor ID and product ID - we need these so we can tell the system to connect
to, say, a Logitech G13 or an Apple wired mouse. In Swift, we can now use the `IOKit.hid` library to
connect to it:

```swift
let manager = IOHIDManagerCreate(kCFAllocatorDefault, IOOptionBits(kIOHIDOptionsTypeNone))
let device_match = [kIOHIDProductIDKey: 56789, kIOHIDVendorIDKey: 1234]

IOHIDManagerSetDeviceMatching(manager, device_match as CFDictionary?)
IOHIDManagerScheduleWithRunLoop(manager, CFRunLoopGetCurrent(), CFRunLoopMode.defaultMode.rawValue);
IOHIDManagerOpen(manager, IOOptionBits(kIOHIDOptionsTypeNone));
```

First, we create an `IOHIDManager`, which will manage the devices we connect to. `kCFAllocatorDefault` is a
memory management object, and the second argument is an unused option, so we just need to pass in a null value
of type `IOOptionBits`. We then create a Swift Dictionary containing the vendor and product
IDs, which we then cast to a `CFDictionary` and pass along with the IOHIDManager object into a
function called `IOHIDManagerSetDeviceMatching`, which will tell the IOHIDManager to look for the
controller. We also need to schedule the manager with a run loop so it can listen for data from the
device. Finally, we call `IOHIDManagerOpen` to activate it.


## Setting up an IOHIDDevice
Now we need to register some callbacks - functions that get run when a given thing happens.

```swift
func matching_callback(inContext, inResult, inSender, inIOHIDDeviceRef) {
    // do some things
}
func removal_callback(inContext, inResult, inSender, inIOHIDDeviceRef) {
    // do some things
}

IOHIDManagerRegisterDeviceMatchingCallback(
    manager,
    matching_callback,
    unsafeBitCast(self, to: UnsafeMutableRawPointer.self)
)
IOHIDManagerRegisterDeviceRemovalCallback(
    manager,
    removal_callback,
    unsafeBitCast(self, to: UnsafeMutableRawPointer.self)
)
```

These are functions to be run whenever a matching device is connected and removed.

## Dissecting the data

At the moment we have 'match' and 'remove' callbacks, but nothing for when buttons are pressed on the controller.
We can hardly hook up a device to listen for input before it's connected, so we need to do that inside the
'match' callback:

let inputCallback : IOHIDReportCallback = { inContext, inResult, inSender, type, reportId, report, reportLength in
    let this: BusyLight = unsafeBitCast(inContext, to: BusyLight.self)
    this.input(inResult, inSender: inSender!, type: type, reportId: reportId, report: report, reportLength: reportLength)
}

// Hook up inputcallback
IOHIDDeviceRegisterInputReportCallback(device!, report, reportSize, inputCallback, unsafeBitCast(self, to: UnsafeMutableRawPointer.self));


## Firing events
