---
layout: post
title: "Setting up a development environment for the ATTiny85 using Raspberry Pi 3 and avrdude"
author: "Joakim Lönnegren"
categories: blog
tags: [Electronics, Embedded, ATTiny85, Raspberry Pi]
image:
  feature: dev-env-attiny85/feature.jpg
  teaser: dev-env-attiny85/teaser.jpg
---


I've started looking at programming USB devices recently and wanted to set up a small development environment for some ATTiny85 chips I have. This first project was just to get a LED blinking. I programmed it using a Raspberry Pi 3b and some spare components. All the code for this project is available on my [Github](https;//github.com/joalon/blinker).

## Hardware
* Raspberry Pi 3b running [CentOS 7](https://wiki.centos.org/SpecialInterestGroup/AltArch/Arm32/RaspberryPi3)
* [Raspberry Pi breakout board](https://www.adafruit.com/product/2029)
* Two small breadboards
* An ATTiny85
* Five 1k resistors
* Some colorful copper wires
* A red LED

![Some hardware](/images/dev-env-attiny85/assorted_hardware.jpg)

When wiring this up I set it up with two spaces on a breadboard, one where the chip is programmed and one where it blinks a LED.

![Breadboard setup](/images/dev-env-attiny85/programming_spot.jpg)


![Blinking spot](/images/dev-env-attiny85/blinking_spot.jpg)

## Configure the Raspberry Pi
I'm not very well versed in wire protocols but to program the chip I needed to enabled SPI and install a burner program, AVRDude. But first I installed some development libraries:

```fish
yum install git avr-libc
yum group install "Development Tools"
```

Enable SPI in the kernel:

```fish
cat >> /boot/config.txt <<EOF
dtparam=spi=on
EOF
reboot
```

I found a [guide](http://kevincuzner.com/2013/05/27/raspberry-pi-as-an-avr-programmer/) to install avrdude with SPI support by the guy who added the support:

```fish
git clone https://github.com/kcuzner/avrdude
cd avrdude/avrdude
./bootstrap && ./configure && make install
```

I made sure the linuxspi programmer in the avrdude config-file (`/usr/local/etc/avrdude.conf`) used the right gpio port for reset. Default is 25, I used 22:

```
programmer
  id = "linuxspi";
  desc = "Use Linux SPI device in /dev/spidev*";
  type = "linuxspi";
  reset = 22;
  baudrate=400000;
;
```

Before programming the chip I had to burn some fuses. I followed the [AVR fuses for beginners](http://m0agx.eu/2016/04/02/avr-fuses-for-beginners/) and ran:

```
avrdude -p t85 -P /dev/spidev0.0 -b 10000 -U lfuse:w:0x62:m -U hfuse:w:0xdf:m -U efuse:w:0xff:m -c linuxspi
```

After this I could try it out with `avrdude -p t85 -P /dev/spidev0.0 -c linuxspi `:

![Test run](/images/dev-env-attiny85/avrdude-test-run.png)

I had some trouble when upgrading the kernel during the project and the /sys/class/gpio [directory disappeared](https://www.kernel.org/doc/Documentation/ABI/obsolete/sysfs-gpio). I had to downgrade the kernel until I'll be able to install an upgraded AVRDude with support for the new kernel API.


## Write the code & flash to chip
The following code will cycle the output power on portc once every half second, which makes the LED blink:

```c
#ifndef F_CPU
#define F_CPU 1000000
#endif

#include <avr/io.h>
#include <util/delay.h>

int main(void) {

	PORTB = 0xFF;
	DDRB |= (1 << PB3);
    while(1) {
	 	PORTB = ~PORTB;
         	_delay_ms(250);
    }

	return 0;
}
```

I compiled this with the following Makefile:

```Makefile
CC=avr-gcc

.PHONY: install clean

all: build strip

clean:
	rm blinker.elf blinker.hex

blinker.elf:
	${CC} -g -Wall -Os -mmcu=attiny85 blinker.c -o blinker.elf

blinker.hex: blinker.elf
	avr-objcopy -O ihex -R .eeprom blinker.elf blinker.hex

build: blinker.elf
strip: blinker.hex

install: blinker.hex
	avrdude -p t85 -P /dev/spidev0.0 -c linuxspi -b 10000 -U flash:w:blinker.hex
```

Running `make install` in the source directory should now succeed:

![avrdude success](/images/dev-env-attiny85/successful-burn.jpg)

Physically moving the attiny85 to the place on the breadboard with the LED should make it start happily blinking:

![Blinking](/images/dev-env-attiny85/blink.gif)


## Summary
I'm using this setup to practice some embedded programming. Next up I hope to do a small USB device with the attiny85. Thanks for reading!
