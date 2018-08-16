---
layout: default
title: Reactive Python on RaspberryPi
category: avr
tags:
- rxpy
- raspberryPi
- python
---

Reactive programming is by no measure a new paradigm. Admittedly, I've not tried my hand at writing an application solely with reactive principals. If you don't have prior knowldege of reactive programming, I highly suggest [this fantastic introduction](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) by [Andre Staltz](https://twitter.com/andrestaltz). In this article we'll look at building an interactive application using a Raspberry Pi, Python, and some external inputs.

For our project, the user will be presented a button and a numeric keypad. When the button is activated, and the correct key pressed, we'll play an audio clip through the speakers.

Parts used in this article:
- [3x4 Keypad](https://www.adafruit.com/product/419)
- Tactile switch button
- Raspberry Pi

![fritzing]({{ site.url }}/reactive_python_fritzing.svg)

A few dependencies to outline before getting started. Firstly, we'll make use of [RxPy](https://github.com/ReactiveX/RxPY), the python port for a reactive programming toolkit called ReactiveX. Secondly, we'll need a library for abstracting the implementation details for handling keypad input. There are a few open source packages available, however for this project I decided to write the keypad utility myself. I've included the code below, with some comments outlining the implementation.

```python
import RPi.GPIO as GPIO

# Simple function to set up our keypad pins for input
def setupPins(rowPins, colPins):
    for pin in rowPins:
        GPIO.setup(pin, GPIO.IN, pull_up_down=GPIO.PUD_UP);

    for pin in colPins:
        GPIO.setup(pin, GPIO.OUT)
        GPIO.output(pin, GPIO.LOW)

# Function to get pressed key
def getPressedKey(rowPins, colPins, keypad):
    def handle(rowChannel):
        rowVal = rowPins.index(rowChannel)
        colVal = None
        for i in range(len(colPins)):
            pin = colPins[i]
            GPIO.output(pin, GPIO.HIGH)
            if (GPIO.input(rowChannel) == GPIO.HIGH):
                GPIO.output(pin, GPIO.LOW)
                colVal = i
                break
            GPIO.output(pin, GPIO.LOW)

        return keypad[rowVal][colVal]
    return handle

# Function for pushing keypad interrupts into our observable
def pushKeyPadPress(rowPins, bouncetime=200):
    def handle(observer):
        for pin in rowPins:
            GPIO.add_event_detect(pin, GPIO.FALLING, callback=lambda p: observer.on_next(p), bouncetime=bouncetime)
    return handle
```

One final thing to cover before we get to coding. Our application will make use of three data streams. We'll require a stream of button events, a stream of keypad events, and a stream which is a composition of both keypad and button events. Our application will rely on the combination stream to determine whether or not to play the audio file. Let's visualize these.

![streams]({{ site.url }}/reactive_python_streams.svg)

First, let's create our switch stream. RxPy streams are created using the ReactiveX `Observable` class. From the RxPy [README.md](https://github.com/ReactiveX/RxPY#the-basics):

> An Observable is the core type in ReactiveX. It serially pushes items, known as emissions, through a series of operators until it finally arrives at an Observer, where they are consumed.

```python
switchStream = Observable.create(lambda observer: GPIO.add_event_detect(SWITCH_PIN, GPIO.BOTH, callback=lambda p: observer.on_next(p), bouncetime=50))\
    .map(GPIO.input)
```

The `Observable.create` factory accepts a function that hands items to the observer. We make use of `GPIO.add_event_detect` to receive a callback for both `GPIO.HIGH` and `GPIO.LOW` events. `callback=lambda p: observer.on_next(p)` is the snippet of code which is responsible for sending our GPIO events into the observer. The value being passed is the pin number. In order to get the pin value, we chain a `.map` to our stream which simply executes `GPIO.input`.

Next, we need to configure our keypad observable.

```python
  keypadStream = Observable.create(padUtils.pushKeyPadPress(ROW_PINS))\
      .map(padUtils.getPressedKey(ROW_PINS, COL_PINS, KEYPAD))
```

The code functions similar to our previous stream. We create the observable using keypad events, and then attach a `.map` which returns the pressed key value.

Our final stream, as we outlined before, combines values from our existing streams. We use the `Observabe.with_latest_from` method to combine the keypad observable with the latest value from the swtich observable. `with_latest_from` takes a lambda function which is responsible for combining the two stream values. In our case, we simply return a tuple of both the latest keypad and switch value. Once we have data in our stream, we can make use of the `Observable.filter` method to enforce our sound playing criteria. Our rules state that the button must be pressed, and that the value of the keypad must correspond to the current day of the week. So, if it were Monday, our user would need to hold down the button and select the number `1` on the keypad for a successful audio response.

```python
keypadStream = Observable.create(padUtils.pushKeyPadPress(ROW_PINS))\
    .map(padUtils.getPressedKey(ROW_PINS, COL_PINS, KEYPAD))

keypadStream.with_latest_from(switchStream, lambda x, y: (x, y))\
    .filter(lambda k: k[1] == GPIO.LOW and k[0] == (datetime.datetime.today().weekday() +1))\
    .subscribe(lambda k: sound.play())

```
If you're having a difficult time wrapping your head around `with_latest_from`, I recommend checking out the interactive diagram over at [rxmarbles](http://rxmarbles.com/#withLatestFrom).

!["rx marbles withlatestFrom"]({{ site.url }}/rxmarbles_with_latest_from.png)

Our full application includes code for importing and configuring our dependencies, handling program interruption, and cleaning up before exiting. Check it out below:

```python
import time
import datetime
import pygame
import RPi.GPIO as GPIO
import padUtils
from rx import Observable

SWITCH_PIN = 23
KEYPAD = [
    [1,2,3],
    [4,5,6],
    [7,8,9],
    ["*",0,"#"]
]
ROW_PINS = [26, 19, 13, 6]
COL_PINS = [5, 11, 9]

pygame.init()
pygame.mixer.init()
sound = pygame.mixer.Sound("countdown.wav")

try:
    GPIO.setmode(GPIO.BCM)
    GPIO.setup(SWITCH_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    padUtils.setupPins(ROW_PINS, COL_PINS)

    switchStream = Observable.create(lambda observer: GPIO.add_event_detect(SWITCH_PIN, GPIO.BOTH, callback=lambda p: observer.on_next(p), bouncetime=50))\
        .map(GPIO.input)

    keypadStream = Observable.create(padUtils.pushKeyPadPress(ROW_PINS))\
        .map(padUtils.getPressedKey(ROW_PINS, COL_PINS, KEYPAD))

    keypadStream.with_latest_from(switchStream, lambda x, y: (x, y))\
        .filter(lambda k: k[1] == GPIO.LOW and k[0] == (datetime.datetime.today().weekday() +1))\
        .throttle_first(sound.get_length() * 1000)\
        .subscribe(lambda k: sound.play())

    while True:
        time.sleep(1)
except KeyboardInterrupt:
    print("Goodbye")
finally:
    GPIO.cleanup()
```

Overall, I enjoyed modeling the application around independent streams of user input. The code is free of global state or complicated nested event handling. You can check out the complete source code on GitHub: [NoumanSaleem/pi-seven-segment-display](https://github.com/NoumanSaleem/reactive-python-raspberry-pi).

