EEPROM
=======

Arduino has a class called "EEPROM" that simply exposes the AVR EEPROM space.  This is very handy for saving data, as EEPROM is byte-addressable.  Thus, it is possible to update a value in storage and have it persist.

The MCU used in Love-to-Code does not have an EEPROM, but we can approximate one using flash memory with the right API glue.


Arduino API
-----------

Arduino contains a complicated C++ object called "EEPROM".  This object uses templates to support reading and writing of arbitrary chunks of memory to the EEPROM at arbitrary offsets.  The EEPROM object implements two functions to accomplish this: EEPROM.get() and EEPROM.put().  Ultimately, the object is an elaborate wrapper around two functions: eeprom_read_byte() and eeprom_write_byte().

The API assumes that data is committed to storage immediately whenever EEPROM.put() is called.  Some example code shows tight loops calling EEPROM.put() repeatedly, or setting a large number of values repeatedly.  This must be taken into consideration when implementing eeprom_read_byte() and eeprom_write_byte().


Flash Limitations
-----------------

Flash memory is fast, but it suffers from a key difference from EEPROM memory: It is not byte-eraseable.  Instead, flash must be addressed using an erase block, which on Love-to-Code is 1 kilobyte.  That means every time the flash is committed, it must first be erased.

Additionally, flash has a limited lifetime.  It can only be erased a finite number of times.  The flash present in Love-to-Code is rated for several thousand erase cycles.  Beyond that, and the flash risks aving stuck bits.

This presents two challenges:

1. We can't program individual bits without first backing up the region and erasing it, and
2. We don't want to burn through the chip's erase cycles whenever we hit a bunch of sequential EEPROM.put() calls.

With these limitations in mind, a compatible design was created.


RAM-backed Flash
----------------

The solution is to buffer writes in RAM before commiting them to flash.  When eeprom_read_byte() is called, we copy the entire flash chunk into a RAM buffer and return the value from that buffer.  If eeprom_write_byte() is subsequently called, we simply update the RAM buffer.  That way, when eeprom_read_byte() is called in the future, it gets the newly-updated value.


Committing Back to Flash
------------------------

There are a number of heuristics we can take to commit the modified RAM buffer into flash.  We could count the number of bytes written, or update the flash at the end of every loop().  Or we 