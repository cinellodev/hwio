# hwio

## Introduction

hwio is a library for interfacing with hardware I/O. It is loosely modelled on
the Arduino programming style, but deviating where that doesn't make sense in
go. It makes use of a thin hardware abstraction via an interface so a program
written against the library for say a beaglebone could be easily compiled to run
on a raspberry pi, maybe only changing pin references.

To use hwio, you just need to import it into your go project, initialise pins as
required, and then use functions that manipulate the pins.

## Basic Usage

Initialising a pin looks like this:

	myPin, err := hwio.GetPin("GPIO4")
	err = hwio.PinMode(myPin, hwio.OUTPUT)

Or the shorter, more convenient form:

	myPin, err := GetPinWithMode("GPIO4", hwio.OUTPUT)

Unlike Arduino, where the pins are directly numbered and you just use the number, in hwio
you get the pin first, by name. This is necessary as different hardware drivers may provide
different pins.

The mode constants include:

 *  INPUT - set pin to digital input
 *  OUTPUT - set pin to digital output
 *  INPUT_PULLUP - set pin to digital input with internal pull-up resistor (where supported by pin)
 *  INPUT_PULLDOWN - set pin to digital input with internal pull-down resistor (where supported by pin)
 *  INPUT_ANALOG - set pin to read analog input

Writing a value to a pin looks like this:

	hwio.DigitalWrite(myPin, hwio.HIGH)

Reading a value from a digital pin looks like this, returning a HIGH or LOW:

	value, err := hwio.DigitalRead(myPin)

Reading an analog value looks like this:

	value, err := hwio.Analogread(somePin)

Analog values (on BeagleBone Black at least) are integers typically between 0-1800, which is the number of millivolts. 
(Note that you cannot drive analog inputs more than 1.8 volts on the BeagleBone, and you should use the analog voltage
references it provides)

## Utility Functions

To delay a number of milliseconds:

	hwio.Delay(500)  // delay 500ms

Or to delay by microseconds:

	hwio.DelayMicroseconds(1500)  // delay 1500 usec, or 1.5 milliseconds

The Arduino ShiftOut function is supported in a simplified form for 8 bits:

	e := hwio.ShiftOut(dataPin, clockPin, 127, hwio.MSBFIRST)   // write 8 bits, MSB first

or in a bigger variant that supports different sizes:

	e := hwio.ShiftOutSize(dataPin, clockPin, someValue, hwio.LSBFIRST, 12)  // write 12 bits LSB first

Sometimes you might want to write an unsigned int to a set of digital pins (e.g. a parallel port). This can be done as
follows:

	somePins := []hwio.Pin{myPin3, myPin2, myPin1, myPin0}
	e := hwio.WriteUIntToPins(myValue, somePins)

This will write out the n lowest bits of myValue, with the most significant bit of that value written to myPin3 etc. It uses DigitalWrite
so the outputs are not written instantaneously.

## Driver Selection

The intention of the hwio library is to use uname to attempt to detect the platform and select an appropriate driver (see drivers section below), 
so for some platforms this may auto-detect. However, with the variety of boards around and the variety of operation systems, you may find that autodetection
doesn't work. If you need to set the driver automatically, you can do:

	hwio.SetDriver(new(BeagleBoneBlackDriver))

This needs to be done before any other hwio calls.

## More Advanced

By default, hwio performs error checking on all operations (e.g. if you write to a pin, but have not set it to a writeable mode, it will return
an error). However, error checking carries a minimal performance overhead. To turn this error checking off, you can:

	hwio.SetErrorChecking(false)

This only affects operations such as DigitalRead and DigitalWrite, which could be invoked at high frequency.

For more information about using the library, see http://stuffwemade.net/hwio which has:

 *	Pin diagrams for supported boards



## BIG SHINY DISCLAIMER

REALLY IMPORTANT THINGS TO KNOW ABOUT THIS ABOUT THIS LIBRARY:

 *	It is under development. If you're lucky, it might work. It should be considered
	Alpha.
 *	Currently it is limited to GPIO on Raspberry Pi, and GPIO and analog input on BeagleBone Black
 *	Only GPIO is tested. Negative interactions with other system functions
	is not tested.
 *	If you don't want to risk frying your board, you can still run the
 	unit tests ;-)


## Board Support

Currently there are 3 drivers:

  *	BeagleBoneBlackDriver - for BeagleBone boards running linux kernel 3.7 or higher, including
  	BeagleBone Black. This is untested on older BeagleBone boards with updated kernels.
  * RaspberryPiDTDriver - for Raspberry Pi modules running linux kernel 3.7 or higher, which includes newer Raspian
    kernels and some late Occidental kernels.
  * TestDriver - for unit tests.

Old pre-kernel-3.7 drivers for BeagleBone and Raspberry Pi have been deprecated as I have no test beds for these. If you want
to use these, you can check out the 'legacy' branch that contains these, but no new features will be added.


### BeagleBoneBlackDriver

This driver accesses hardware via the device interfaces exposed in the file system on linux kernels 3.8 or higher, where
device tree is mandated. This should be a robust driver as the hardware access is maintained by device driver authors,
but is likely to be not as fast as direct memory I/O to the hardware as there is file system overhead.

Status:

  * In active development
  * New driver, not yet fully tested but good indications it works as it is supposed to.
  * Driver automatically blocks out the GPIO pins that are allocated to LCD and MMC on the default BeagleBone Black boards.
  * GPIOs not assigned at boot to other modules are known to read and write.
  * USR0-USR3 don't work and cannot be accessed as GPIO, as the LED driver reserves them.
  * Analog reads are working
  * GPIO pull-ups is not yet supported.


### RaspberryPiDTDriver

This driver is very similar to the BeagleBone Black driver in that it uses the modules compiled into the kernel and
configured using device tree. It uses the same GPIO implementaton, just with different pins.

DigitalRead and DigitalWrite are implemented, but are not currently tested.

GetPin references on this driver return the pin numbers that are on the headers. Pin 0 is unimplemented.

## How it Works

Hardware drivers implement the interface that allows different devices to
implement the I/O handling as appropriate. This allows for drivers that use
different techniques to access the I/O. e.g. some drivers may use GPIO (a file
system approach), whereas for maximum performance another driver might use
direct memory access to I/O registers, but is less portable.

Some general principles the library attempts to adhere to include:

 *	Pin references are logical, and are mapped to hardware pins by the driver. The pin
    numbers you get back from GetPin are, unless otherwise specified, related to the pin numbers
    on extension headers.
 *	Drivers provide pin names, so you can look them up by meaningful names
	instead of relying on device specific numbers.
 *	The library does not implement Arduino functions for their own sake if go's
	framework naturally supports them better, unless we can provide a simpler interface
 	to those functions and keep close to the Arduino semantics.
 *	Drivers are very thin layers; most of the I/O functionality is provided by **modules**.
    These aim to be as generic as possible so that different drivers on similar kernels can
    assemble the modules that are enabled in device tree, with appropriate pin configuration.
    This also makes it easier to add in new modules to support various SoC functions.
 *	Make no assumption about the state of a pin whose mode has not been set.
 	Specifically, pins that don't have mode set may not on a particular hardware
 	configuration even be configured as general purpose I/O. For example, many
 	beaglebone pins have overloaded functions set using a multiplexer, and some may be pre-assigned.
 	Any pin whose mode is set by PinMode can be assumed to be general purpose I/O, and
 	likewise if it is not set, it could have any multiplexed behaviour assigned
 	to it. A consequence is that unlike Arduino, PinMode *must* be called before
 	a pin is used.
 *	The library should be as fast as possible so that applications that require
 	very high speed I/O should achieve maximal throughput, given an appropriate
 	driver.
 *	The library should be tolerant towards beginners, and give meaningful errors
 	when the hardware is requested to do things it can't. But this checking
 	should be able to be disabled for maximum performance.
 *	Make simple stuff simple, and harder stuff possible. In particular, while
 	Arduino-like methods have uniform interface and semantics across drivers,
 	we don't hide the driver itself or the modules it uses, so special features of a driver or module
 	can still be used, albeit non-portably.
 *	Sub-packages can be added as required that approximately parallel Arduino
 	libaries (e.g. perhaps an SD card package). Where possible, these
 	implementations should be generic across I/O pins. e.g. not assuming
 	specific pin behaviour as happens with Arduino.


### Pins

Pins are logical representation of physical pins on the hardware. To provide
some abstraction, pins are numbered, much like on an Arduino. Unlike Arduino,
there is no single mapping to hardware pins - this is done by the hardware
driver. To make it easier to work with, drivers can give one or more names to
a pin, and you can use GetPin to get a reference to the pin by one of those
names.

Each driver must implement a method that defines the mapping from logical pins
to physical pins as understood by that piece of hardware. Additionally, the
driver also publishes the modules that the hardware configuration supports, so
that hwio can ensure that constraints of the hardware are met. For example, if a
pin implements PWM and GPIO in hardware, it is associated with two modules. When
the PWM module is enabled, it will assign the pin to itself. Because each
pin can have a different set of capabilities, there is no distinction between
analog and digital pins as there is in Arduino; there is one set of pins, which
may support digital and/or analog capabilities.

The caller generally works with logical pin numbers retrieved by GetPin.


## Things to be done

 *	PWM output support (BeagleBone, R-Pi)
 *	Interupts (lib, BeagleBone and R-Pi)
 *	Serial support for UART pins (lib, BeagleBone and R-Pi)
 *	SPI support; consider augmenting ShiftIn and ShiftOut to use hardware pins
 	if appropriate (Beaglebone and R-Pi)
 *	LCD, particularly HD44780 (lib)
 *	Servo (lib)
 *	Stepper (lib)
 *	I2C (lib)
 *	TLC5940 (lib)
