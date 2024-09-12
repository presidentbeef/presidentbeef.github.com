---
layout: post
title: "Tiny E-Ink Picture Display"
date: 2024-04-14 14:23
comments: true
categories: 
---

After being surprised by the capabilities of a three-color e-ink display (and struggling to get it to work!), I thought I'd put together a little guide.

## Hardware

The hardware I used:

- [Adafruit 2.13 E-Ink Tricolor Display (ThinkInk)](https://www.adafruit.com/product/4947)
- [Adafruit ESP32 Feather (ESP32-WROOM-32E)](https://www.adafruit.com/product/3619)
- A micro SD card

In this case, I messed up a little. I already had an ESP32 feather board from Adafruit, so I should have grabbed an [e-ink "feather wing"](https://www.adafruit.com/product/4778) which would have plugged straight into the ESP32 board.

But since I did not do that... here's how I wired up the display:

* `3V3` (power) to `3V`
* `GND` (ground) to `GND`
* `SCK` (clock) to `SCK`
* `MISO` to `MISO`
* `MOSI` to `MOSI`
* `ECS` to 27
* `D/C` to 33
* `SRCS` to 15
* `SDCS` to 32

The rest I didn't connect.

_Note: Don't be tempted to use pins 12 and 13!! Pin 13 is actually shared with the onboard LED, and documentation for pin 12 says "this pin has a pull-down resistor built into it, we recommend using it as an output only"._

The names on the board don't quite match the names in the code, so here's a cheatsheet:

```c
#define EPD_DC 33     // D/C
#define EPD_CS 27     // ECS
#define SRAM_CS 15    // SRCS
#define EPD_BUSY -1   // can set to -1 to not use a pin
#define EPD_RESET -1  // can set to -1 and share with chip Reset (can't deep sleep)
#define SD_CS 32      // SDCS
```

The libraries use defaults for the rest of the pins automatically.

## Setting Up Pictures

To fit the display exactly, pictures should be 250 pixels x 122 pixels. However, the display will crop as needed.

I used Gimp and ImageMagick to make the pictures, but the main thing is the images need to be 24-bit bitmaps. I couldn't get Gimp to save images directly to a working format.

Here are the steps I took:

* In Gimp, crop and resize to 250x122 pixels (I prefer to crop, resize to 250 pixels wide, crop again to 122 pixels high.)
* Set palette:
  * Go to `Image` → `Mode` → `Indexed...`
  * Select "black and white 1 bit palette"
  * OR create a new black/red/white palette and use that
  * Choose a dithering option that looks good

As far as the color to use for "red", I believe as long as it has the `r` value of `255`, it will work.

![Gimp image mode](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jofyf7mptsr4k2eyp2z9.png)

Then...

* `File` → `Export As...`
* Rename to end in `.bmp` and save

Then...

From the command line, run

* `convert your_image.bmp -type truecolor your_image_24.bmp`

### Upload

Save the pictures to the root directory of a micro SD card, then put the card in the display (for me, it's text "down"). The slot is spring-loaded, so just push on the end to eject.

![Back of e-ink display showing micro SD card inserted](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0796oc6oijha0y15p1qb.jpg)

## Arduino

I used the [Arduino IDE](https://www.arduino.cc/en/software) (2.3.2).

For the board type, use "Adafruit ESP32 Feather". (This may seem obvious, but it took me a while to figure out which to use!)

These are the libraries I used (via the IDE's Library Manager):

- [Adafruit ImageReader library (v2.9.2)](https://github.com/adafruit/Adafruit_ImageReader)
- [Adafruit EPD (v4.5.4)](https://github.com/adafruit/Adafruit_EPD)

![Arduino IDE](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mq752bcbtwgk83dcnlus.png)

## Code

[Full code is available here!](https://github.com/presidentbeef/e-ink-photo-frame)

Other good examples to start from:
* [Tricolor test](https://github.com/adafruit/Adafruit_EPD/blob/6c7aff424af5fde45b14ee6acd2f0ce92f2f459d/examples/ThinkInk_tricolor/ThinkInk_tricolor.ino)
* [Image reader](https://github.com/adafruit/Adafruit_ImageReader/blob/4588e481534d3f9319eb79f251595007e650c116/examples/EInkBreakouts/EInkBreakouts.ino)

You'll want to adjust the pin definitions like I did above if you are following along.

`ThinkInk_213_Tricolor_RW` is the right type to use for the display above.

In my code, I stripped out anything not related to loading and displaying images from the SD card. If you are doing something different, try looking at the other examples.

**Update these lines with the names of your images!**

```c
  int num_images = 4; // Update with number of images

  // List image paths
  char *images[num_images] = {
    "/image1.bmp",
    "/image2.bmp",
    "/image3.bmp",
    "/image4.bmp",
  };

```

The program will cycle through the images and update every 5 minutes (or whatever you change the delay to - recommended minimum is 3 minutes).

## Results

Here are some examples. Images look best from a little distance.

![E-ink display connected to ESP32; showing picture of a computer](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/osf9sondxhxm7oib1sgi.jpg)

![E-ink display showing picture of a house](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/detw3xbclf3wtm6papmd.jpg)

![E-ink display showing picture of a rhinoceros](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nlo1mjodbwzp0oagii54.jpg)

## No Power?!

Yep, the main cool thing about an e-ink display is that they don't need power to maintain the image.

However, I found just removing power from the ESP32 would cause the red pixels to "bloom" and make everything a bit pink.

To prevent this, just disconnect power from the display first. It's possible there is a way to fix this in the code - let me know if you figure it out!

## Have Fun!

![E-ink display showing picture of a handsome fella](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i0d6hoyareg4noqn86fc.jpg)
