# About Love-to-Code

Love-to-Code is a new kind of crafting supply available from Chibitronics.  It is designed to make it easy to program and even easier to deploy.  It uses a low-power microcontroller much like that provided by Arduino.  The key difference is that it is programmed via audio, which means it can be programmed using a standard laptop or mobile phone.

Audio programming is done via a web page, meaning any modern web browser is able to 

Ordinarily, programs require a large amount of overhead.  An ArduinoDue program that simply blinks an LED can be over 10 kilobytes.  When programming over an error-prone audio link, every byte counts, and a base of 10 kilobytes is simply unacceptable.

On larger desktop systems this is not an issue because most of the code is present in the form of shared libraries. Microcontrollers must bundle together all code they might possibly use, so they don't get the benefit of shared libraries.  On a smaller microcontroller such as the one used in an ArduinoUno, this is not as large an issue because the chip is not as capable, and so it ships with less supporting code.  Both the Arduino Due and LtC are approximately the same class of processor, and therefore share similar restrictions and requirements.

In order to function well in this limited environment, the Love-to-Code environment uses many tricks.  These range from defining a common toolbox with a syscall interface, to running a multithreading microkernel under the hood, to defining a special loader header that greatly limits bookkeeping overhead.

As a result, a simple "light blink" demo on an Arduino Due clocks in at 10 kilobytes, whereas the same sketch on LtC weighs in at 200 bytes, fitting comfortably into a single 256-byte audio packet.

Audio programming is the big innovation with LtC, and is designed to compile and program a device within the attention span of a six year old.  To do this, compilation needs to be fast, but programming also needs to be fast.  Importantly, since microphone support may not be present, programming is a one-way write-only operation.  Hashing needs to be used to prevent programming corrupt data, but hashes need to be fast enough to be run on data as it's received, even on a 48 MHz CPU.

This book discusses the tricks that are used to limit the amount of data that gets sent via the audio channel while still producing an environment that is familiar to those who are familiar with the Arduino ecosystem.  In short, a custom executable format, some clever libraries, extensive use of link-time-optimization, and a carefully-crafted syscall interface all work in concert to allow LtC to function with 32 kilobytes of flash and 4 kilobytes of RAM.

