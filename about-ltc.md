# About Love-to-Code

Love-to-Code is a new kind of crafting supply available from Chibitronics.  It is designed to make it easy to program and even easier to deploy.  It uses a low-power microcontroller much like that provided by Arduino.  The key difference is that it is programmed via audio, which means it can be programmed using a standard laptop or mobile phone.

Ordinarily, programs require a large amount of overhead.  An ArduinoDue program that simply blinks an LED can be over 10 kilobytes.  When programming over an error-prone audio link, every byte counts, and a base of 10 kilobytes is simply unacceptable.

 On larger desktop systems this is not an issue because most of the code is present in the form of shared libraries. Microcontrollers must bundle together all code they might possibly use, so they don't get the benefit of shared libraries.  On a smaller microcontroller such as the one used in an ArduinoUno, this is not as large an issue because the chip is not as capable, and so it ships with less supporting code.  Both the Arduino Due and LtC are approximately the same class of processor, and therefore share similar restrictions and requirements.

In order to function well in this limited environment, the Love-to-Code environment makes extensive use of 

