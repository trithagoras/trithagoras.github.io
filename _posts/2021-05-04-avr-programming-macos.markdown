---
layout: post
title:  "AVR programming on MacOS"
date:   2021-05-04 15:09:00 +1000
categories: jekyll update
permalink: /avr-programming-macos/
---

For my microcontrollers university course, we were told that Windows OS was a requirement, as we needed to use the [Microchip Studio](https://www.microchip.com/en-us/development-tools-tools-and-software/microchip-studio-for-avr-and-sam-devices) (formerly Atmel Studio) IDE. Instead, I'll be showing you how I set up developing for the **atmega324a** specifically for MacOS.

<br>

## Requirements
For this, you are required to have installed:
- `avr-gcc`
- `avrdude`

A great set of commands to install these with [Homebrew](https://brew.sh) can be found [here](https://gist.github.com/jj1bdx/f149305a57c4cb2cef7c). In case this page is not available anymore, they are:

{% highlight bash %}
$ brew tap osx-cross/avr
$ brew install avr-gcc avr-binutils
$ brew install avrdude
{% endhighlight %}

<br>

## Makefile
Now, we can make a Makefile.

I modified a well crafted MakeFile retrieved from [HolaCheck's Github](https://gist.github.com/holachek/3304890) into working with my configuration (using the **atmega324a**). My final MakeFile can be found at [this pastebin](https://pastebin.com/wKndwk0w) or at the bottom of the page.

The Makefile is where the majority of the work is done. The variables may depend on other factors and these are just what worked for me.
  - It should be clear that the variable `OBJECTS` is the list of compiled source files to be included.

  - The `PORT` variable can be found by running `$ ls /dev/cu.*`. You should find two devices in the format of `cu.usbmodem002042642`. Typically, the lower number is the programmer port and is the one that should be assigned to our `PORT` variable.

  - When `$ make install` runs, you may see `Fuses OK (E:FF, H:D1, L:E6)` appear from `avrdude` or some variation. You can match these fuse values in the Makefile. For example, we can deduce that the `-U hfuse:w:0xd1:m` in the Makefile refers to the `H:d1` fuse. If your fuses are different, update them in the Makefile.


<br>


## Test Program
Next, we can make a test program. This program simply outputs a pulse signal through PD1 with delays.

In `main.c`:
{% highlight c %}
#include <avr/io.h>
#include <util/delay.h>

int main() {
    DDRD |= (1 << 1);           // set LED pin PD1 to output
    while (1) {
        PORTD |= (1 << 1);      // drive PD1 high
        _delay_ms(1000);
        PORTD &= ~(1 << 1);     // drive PD1 low
        _delay_ms(200);
    }
}
{% endhighlight %}

<br>

## Testing
Now, we simply run the command in the directory (with our arduino plugged in, of course).

{% highlight bash %}
$ make install
{% endhighlight %}

And our program should have flashed to the arduino and should be immediately working.

<br>

## Serial Communication
As a final note, there will often be serial communication involved in your project. First, you need to know the serial port to communicate with.

Following the same trick we used to find the programming port, we can check with `ls /dev/cu.*` to find the serial port. In my case, it looked like this:

`/dev/cu.usbserial-DA7WKGD`

Then, knowing the communication speed (in my case, 19200), we can run:

{% highlight bash %}
$ screen /dev/cu.usbserial-DA7WKGD 19200
{% endhighlight %}

and it should work perfect.

You should note that sometimes you may get the errors that the port is busy for R/W. This can be solved by closing all `screen` processes in Activity Monitor, or by unplugging and replugging the USB.

<br>

## Potential Errors
Ultimately, I was able to deduce this by reading from many different sources and filling in the blanks with an admittedly limited understanding.

Because of this, there are potentially many errors that could arise, though I haven't encountered any of them yet.

<br>

### Problem 1
As of writing this, `avrdude` does not support the **atmega324a**. To get around this, I made it so that `avrdude` is using the device `atmega328p`, while keeping the `-mmcu` flag in `avr-gcc` as `atmega324a`.

As a consequence of this, every time I run `make install`, I get the response `make: *** [fuse] Error 1` and

{% highlight bash %}
avrdude: Device signature = 0x1e9515
avrdude: Expected signature for ATmega328P is 1E 95 0F
{% endhighlight %}

Though the program should still flash correctly.

<br>

### Problem 2
In my class, we were supplied a source file that would not compile with `avr-gcc` using my configuration. The errors were to do with some defined macros such as:

{% highlight c %}
SPCR0
SPSR0
SPR00
SPR10
SPDR0
SPIF0
SPSR0
{% endhighlight %}

For this, I simply removed the postfixed `0` for each of these macros, and it compiled and flashed fine. As of now, I do not know the full consequences of this.

<br>

**Makefile**
<details>
  <summary>Click to expand</summary>

{% highlight make %}
# Name: Makefile
# Author: <insert your name here>
# Copyright: <insert your copyright message here>
# License: <insert your license reference here>

# DEVICE ....... The AVR device you compile for
# CLOCK ........ Target AVR clock rate in Hertz
# OBJECTS ...... The object files created from your source files. This list is
#                usually the same as the list of source files with suffix ".o".
# PROGRAMMER ... Options to avrdude which define the hardware you use for
#                uploading to the AVR and the interface where this hardware
#                is connected.
# FUSES ........ Parameters for avrdude to flash the fuses appropriately.

DEVICE     = atmega324a
AVRDUDEDEV = atmega328p
CLOCK      = 8000000
PORT	   = /dev/cu.usbmodem002042642
PROGRAMMER = -c stk500v2 -P $(PORT)
OBJECTS    = main.o other.o
FUSES      = -U lfuse:w:0xE6:m -U hfuse:w:0xd1:m -U efuse:w:0xff:m


######################################################################
######################################################################

# Tune the lines below only if you know what you are doing:

AVRDUDE = avrdude $(PROGRAMMER) -p $(AVRDUDEDEV)
COMPILE = avr-gcc -Wall -Os -DF_CPU=$(CLOCK) -mmcu=$(DEVICE)

# symbolic targets:
all:	main.hex

.c.o:
	$(COMPILE) -c $< -o $@

.S.o:
	$(COMPILE) -x assembler-with-cpp -c $< -o $@
# "-x assembler-with-cpp" should not be necessary since this is the default
# file type for the .S (with capital S) extension. However, upper case
# characters are not always preserved on Windows. To ensure WinAVR
# compatibility define the file type manually.

.c.s:
	$(COMPILE) -S $< -o $@

flash:	all
	$(AVRDUDE) -U flash:w:main.hex:i -F

fuse:
	$(AVRDUDE) $(FUSES)

install: flash fuse

# if you use a bootloader, change the command below appropriately:
load: all
	bootloadHID main.hex

clean:
	rm -f main.hex main.elf $(OBJECTS)

# file targets:
main.elf: $(OBJECTS)
	$(COMPILE) -o main.elf $(OBJECTS)

main.hex: main.elf
	rm -f main.hex
	avr-objcopy -j .text -j .data -O ihex main.elf main.hex
# If you have an EEPROM section, you must also create a hex file for the
# EEPROM and add it to the "flash" target.

# Targets for code debugging and analysis:
disasm:	main.elf
	avr-objdump -d main.elf

cpp:
	$(COMPILE) -E main.c

{% endhighlight %}
</details>