---
layout: post
title:  "AVR programming on MacOS"
date:   2021-05-04 15:09:00 +1000
categories: jekyll update
permalink: /avr-programming-macos/
---

For my microcontrollers university course, we were told that Windows OS was a requirement, as we needed to use the [Microchip Studio](https://www.microchip.com/en-us/development-tools-tools-and-software/microchip-studio-for-avr-and-sam-devices) (formerly Atmel Studio) IDE. Instead, I'll be showing you how I set up developing for the **atmega328p** specifically for MacOS.

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

## MakeFile
Now, we can make a MakeFile.

I modified a well crafted MakeFile retrieved from [HolaCheck's Github](https://gist.github.com/holachek/3304890) into working with my configuration (using the **atmega328p**). My final MakeFile can be found at [this pastebin](https://pastebin.com/ENDP4XES) or at the bottom of the page.

The MakeFile is where the majority of the work is done. The variables may depend on other factors and these are just what worked for me.
  - It should be clear that the variable `OBJECTS` is the list of compiled source files to be included.

  - The `PORT` variable can be found by running `$ ls /dev/cu.*`. You should find two devices in the format of `cu.usbmodem002042642`. Typically, the lower number is the programmer port and is the one that should be assigned to our `PORT` variable.

  - When `$ make install` runs, you may see `Fuses OK (E:FF, H:D1, L:E6)` appear from `avrdude` or some variation. You can match these fuse values in the MakeFile. For example, we can deduce that the `-U hfuse:w:0xd1:m` in the MakeFile refers to the `H:d1` fuse. If your fuses are different, update them in the MakeFile.


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

## Conclusion
Now, we simply run the commands in the directory (with our arduino plugged in, of course).

{% highlight bash %}
$ make clean
$ make
$ make install
{% endhighlight %}

And our program should have flashed to the arduino and should be working.

<br>

**MakeFile**
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

DEVICE     = atmega328p
CLOCK      = 20000000
PORT	   = /dev/cu.usbmodem002042642
PROGRAMMER = -c stk500v2 -P $(PORT)
OBJECTS    = main.o other.o
FUSES      = -U lfuse:w:0xE6:m -U hfuse:w:0xd1:m -U efuse:w:0xff:m


######################################################################
######################################################################

# Tune the lines below only if you know what you are doing:

AVRDUDE = avrdude $(PROGRAMMER) -p $(DEVICE)
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