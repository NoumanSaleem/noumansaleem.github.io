---
layout: default
title: Driving a seven-segment display on Raspberry Pi with Python
category: avr
tags:
- avr
- raspberryPi
- python
---

In this article, we'll look interfacing with a seven-segment display using python and a Raspberry Pi.

Parts used in this article:
- [7-Segment clock display](https://www.adafruit.com/product/865)
- [74HC595 shift register](https://www.adafruit.com/product/450)
- Raspberry Pi
- Breadboard and lots of jumper wires

## Display Module
The display module I'm using is a 4-digit 7-segment display. For the purpose of this article, we'll only enable and manipulate the last digit. The information should be transferable to a single-digit display.

Before getting started, let's review some basic information about seven-segment displays. As the name implies, the display is composed of seven individual light emitting diodes. These segments are referred to by the letters A through G.

![Display Segments]({{ site.url }}/assets/7_Segment_Display_with_Labeled_Segments.svg "By Uln2003 - Own work, CC0, https://commons.wikimedia.org/w/index.php?curid=69807877")

Turning all seven segments on would render the number "8", while turning on only B and C would display the number "1". Each segment is independently controlled, and the layout above is universal across displays. What can differ between modules is how they are internally wired. The two kinds of displays are *Common Cathode (CC)* and *Common Anode (CA)*. In a Common Cathode display, all the cathodes (negative terminals) of each segment are connected together. To turn on a segment, you set the corresponding pin to HIGH. In a Common Anode display, all the anodes (positive terminals) are connected together. To turn on a segment of a CA display, you set the corresponding pin to LOW.

The module I'm using is a common cathode display. The pinout is included below for reference.

![Display pinout]({{ site.url }}/assets/4_digit_seven_segment_display_pinout.svg)

## Wiring and Code
Connect `D4` to ground, and `A-G` to the following GPIO pins with a 100ohm resistor. I've used two breadboards in the fritzing diagram below to avoid overlapping jumper cables.
- `A`: 17
- `B`: 22
- `C`: 6
- `D`: 13
- `E`: 19
- `F`: 27
- `G`: 5

![Frizting layout Raspberry Pi Seven-Segment Display]({{ site.url }}/assets/raspberry_pi_7_segment_display_bb_simple.svg)

With our wiring complete, it's time for code. Our program will count from 0 to 9, and then reset to 0. For each iteration, we'll pass the current number to a function responsible for displaying the digit on the display. Our application will make use of a bit array (or bit map) to map each digit to their required segments. A bit array is a data structure that stores bits. For our app, we'll need 7 bits to represent each individual segment. A bit of value `1` represents on, while a bit of value `0` represents off. Our bit array will have a length of 8, and the right most bit represents the first segment `A`. Let's look at a few examples.

- `0x00000001` - Segment A is on.
- `0x00000110` - Segments B and C are on.
- `0x01111111` - All 7 segments (A-G) are on.

The first thing we'll need in our application is a bit array for the digits 0-9.

```python
digitBitmap = { 0: 0b00111111, 1: 0b00000110, 2: 0b01011011, 3: 0b01001111, 4: 0b01100110, 5: 0b01101101, 6: 0b01111101, 7: 0b00000111, 8: 0b01111111, 9: 0b01100111 }
```

Now that we know which segments need to be enabled for each display digit, we'll need a way for our application to determine whether or not a segment should be enabled for a given bit array. The solution is to use a bit mask and the bitwise `AND` operator. We'll have a bitmask for each of our 7 segments. The bitmask, like the bit array, will have a length of 8; however only the bit corresponding to the individual segment will be set to `1`. Combined with the bitwise `AND` operator, we'll be able to determine if an individual segment should be turned on for a given digit. Here are some examples of using bitwise `AND`.

```python
0x000000110 & 0x00000001 # resolves to 0, which means segment A should be off
0x000000110 & 0x00000010 # resolves to 1, which means segment B should be on
0x000000110 & 0x00000100 # resolves to 1, which means segment C should be on
```

Let's add our bitmasks to the application.

```python
digitBitmap = { 0: 0b00111111, 1: 0b00000110, 2: 0b01011011, 3: 0b01001111, 4: 0b01100110, 5: 0b01101101, 6: 0b01111101, 7: 0b00000111, 8: 0b01111111, 9: 0b01100111 }
masks = { 'a': 0b00000001, 'b': 0b00000010, 'c': 0b00000100, 'd': 0b00001000, 'e': 0b00010000, 'f': 0b00100000, 'g': 0b01000000 }
```

An illustration of our bit array and bitmask
![Bit array bit mask review]({{ site.url }}/assets/bitarray_bitmask_overview.svg)

The last mapping we'll need is for each segment to its GPIO pin.
```python
digitBitmap = { 0: 0b00111111, 1: 0b00000110, 2: 0b01011011, 3: 0b01001111, 4: 0b01100110, 5: 0b01101101, 6: 0b01111101, 7: 0b00000111, 8: 0b01111111, 9: 0b01100111 }
masks = { 'a': 0b00000001, 'b': 0b00000010, 'c': 0b00000100, 'd': 0b00001000, 'e': 0b00010000, 'f': 0b00100000, 'g': 0b01000000 }
pins = { 'a': 17, 'b': 22, 'c': 6, 'd': 13, 'e': 19, 'f': 27, 'g': 5}
```

Now that we have all our mappings in place, let's write the function responsible for toggling the correct pins for a given digit.

```python
def renderChar(c):
    val = digitBitmap[c]

    GPIO.output(list(pins.values()), GPIO.LOW)

    for k,v in masks.items():
        if val&v == v:
            GPIO.output(pins[k], GPIO.HIGH)
```
The functions starts by resolving the bit array which represents the passed in digit. Next, we turn all the 7 segments off. Lastly, we iterate each segment, and enable pins when the bitwise operation returns truthy.

The last bit of code we need is to initialize our counter to 0 and handle incrementing and resetting the value in an endless loop.

```python
val = 0

while True:
    renderChar(val)
    val = 0 if val == 9 else (val + 1)
    time.sleep(1)
```

The complete code includes importing our required libraries, initializing our gpio pins, and handling program interrupt and cleanup.

```python
import time
import RPi.GPIO as GPIO

GPIO.setmode(GPIO.BCM)

digitBitmap = { 0: 0b00111111, 1: 0b00000110, 2: 0b01011011, 3: 0b01001111, 4: 0b01100110, 5: 0b01101101, 6: 0b01111101, 7: 0b00000111, 8: 0b01111111, 9: 0b01100111 }
masks = { 'a': 0b00000001, 'b': 0b00000010, 'c': 0b00000100, 'd': 0b00001000, 'e': 0b00010000, 'f': 0b00100000, 'g': 0b01000000 }
pins = { 'a': 17, 'b': 22, 'c': 6, 'd': 13, 'e': 19, 'f': 27, 'g': 5}

def renderChar(c):
    val = digitBitmap[c]

    GPIO.output(list(pins.values()), GPIO.LOW)

    for k,v in masks.items():
        if val&v == v:
            GPIO.output(pins[k], GPIO.HIGH)

try:
    GPIO.setup(list(pins.values()), GPIO.OUT)
    GPIO.output(list(pins.values()), GPIO.LOW)

    val = 0

    while True:
        renderChar(val)
        val = 0 if val == 9 else (val + 1)
        time.sleep(1)
except KeyboardInterrupt:
    print("Goodbye")
finally:
    GPIO.cleanup()
```

And we're done! Upload your program to your Raspberry Pi and enjoy watching the numbers count up. In future articles, we'll explore using a shift register to reduce the number of GPIO pins required to drive the display, and also how to display more than one character on the display. The code for this article can be found on GitHub: [NoumanSaleem/pi-seven-segment-display](https://github.com/NoumanSaleem/pi-seven-segment-display/tree/master/simple).
