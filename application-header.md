Application Header Backround
----------------------------

Files on a disk are just bits.  They need some sort of metadata to describe what they are.  For example, if a series of bits in a file on disk describes raw pixels in a picture, any sort of viewer program still needs to know at a minimum the picture's width and height, plus ideally other information such as the format of the pixels, the number of bits, and sometimes extra data such as where a photograph was taken or what sort of lens was used.

Likewise, binary programs stored as a file on a disk normally need headers describing the contents of the file.  There's more to that .EXE file than just binary code.  Modern executable formats including Linux ELF and macOS Mach-O can contain multiple entry points for different architectures, have embedded resource, support code signatures, shared library importa, debugging symbols, and more.  All of this is accomplished by placing an advanced header around the program code, and comes at the cost of a larger file size.

At the other end of the spectrum are DOS .COM files, which have no header and a lot of assumptions.  In a .COM file, the data is loaded to memory offset 0x100 and the CPU is set to execute the first instruction.  The program must take care of all of its own initialization, but in a way the extension of .COM tells the system everything it needs to know about the file.

Embedded platforms tend to take the .COM approach rather than the .EXE approach, as they don't need to dynamically load any symbols at runtime, don't need code signing, and usually only need to support the one architecture.  As a result, when you build an image for an Arduino, the end result is usually a .BIN file ready to be burned into flash.

The Love-to-Code environment opts for a minimal header that allows many common setup functions to be relegated to the ROM, and allows for smaller binary sizes.


LtC Header format
-----------------

The header is placed before any program code, and the ROM reads the header file from storage.  The header format is defined as a struct inside app.h, and is a series of 32-bit values:

 +--------+------------+-----------------+---------------------------------------+
 | Offset | Type       | Name            | Description                           |
 +--------+------------+-----------------+---------------------------------------+
 |  0x00  | Flash      | data_load_start | Data section source address           |
 |  0x04  | RAM        | data_start      | Start of data section target address  |
 |  0x08  | RAM        | data_end        | End of data section target address    |
 +--------+------------+-----------------+---------------------------------------+
 |  0x0c  | RAM        | bss_start       | Start of BSS address                  |
 |  0x10  | RAM        | bss_end         | End of BSS address                    |
 +--------+------------+-----------------+---------------------------------------+
 |  0x14  | Flash/RAM  | entry           | Program entrypoint                    |
 +--------+------------+-----------------+---------------------------------------+
 |  0x18  | 0xd3fbf67a | magic           | Magic signature number                |
 |  0x1c  | 0x00000100 | version         | Version number 0.0.1.0                |
 +--------+------------+-----------------+---------------------------------------+
 |  0x20  | Flash/RAM  | const_start     | First C++ constructor address         |
 |  0x24  | Flash/RAM  | const_end       | Last C++ constructor address          |
 +--------+------------+-----------------+---------------------------------------+
 |  0x28  | RAM        | heap_start      | Start of heap                         |
 |  0x2c  | RAM        | heap_end        | End of heap                           |
 +--------+------------+-----------------+---------------------------------------+

The first set of addresses have to do with values in the .data section of the program.  This is any values that need to be modified at runtime and have a default value.  For example, this would define a value in the data section:

    int val = 42;

This value can be modified, and must be set to a value.  Pre-initialized values are stored in flash, beginning at data_load_start.  As part of its initialization setup, the LtC loader will copy values to data_start located in RAM.  The loader knows how many bytes to copy by continuing until it reaches data_end.  In this way, the data section is initialized.

The next sections have to do with the .bss section.  This is like a data section, except all values are 0.  An example of how to create data in the .bss section is to do something such as:

    int emptyval;
    int emptyval2 = 0;
    int array[256];

Each one of these values will be initialized to zero.  And thanks to the fact that the .bss is just a range of addresses, these values take up zero space in the resulting binary.  The LtC loader can use these addresses to pre-zero the .bss section for the program, saving the program from needing to do that on its own.

The next value in the header is the program entrypoint.  This is an address either in flash (for .text data) or in RAM (for code that was put in the .data section).  This is simply the address the ROM will jump to once initialization is performed.  It is usually a program's main() function, but in the case of Arduino it can be different.

These values are all that are really needed to get a program up and running.  Most well-behaved C programs would be fine with only these values.  The remaining fields add robustness and language features that are used by the LtC ecosystem.

Following the entrypoint are two 32-bit values.  The magic number is simply a well-defined 32-bit value that indicates the program is an LtC program.  Frequently, programs put this signature at the start of the format, but LtC opts to put it here.  The value is 0xd3fbf67a.  If you're looking at a file and it has this value at offset 0x18, chances are it's an LtC program.  Following the magic number is a version field, which is currently 0x00000100, or 0.0.1.0.  Future versions may change this.

After the entrypoint comes C++ constructor addresses.  In C++, it is possible to instantiate an object much like one instantiates a variable, except instead of placing it into the data section, the constructor needs to run to create the object.  This constructor must run before the main function, and there may be multiple objects that need constructing.  To accomplish this, the loader checks to see if const_start != const_end, and if not will treat it as an array of function calls.  These get called after the .data and .bss sections are set up, but before the entrypoint is called.

Finally, the header includes addresses for the heap.  When a program is compiled, it should know how much memory is available on the chip, and can pass this information to the bootloader and operating system.  This can be used to get malloc() to function as expected.

Code and data may follow the header, but because the actual code's entrypoint is pointed to by the entrypoint field, it doesn't really matter where it goes.


Program Entrypoint
------------------

Strictly speaking, the "entrypoint" is the first thing in the file, not where code starts executing.  The first thing in the file is data and not code, but GCC will happily take care of this for us.

The header described in the table above is defined in C like so:

    struct app_header {
      /* 0x00 */
      uint32_t *data_load_start;  /* Start of data in ROM */
      uint32_t *data_start;       /* Start of data load address in RAM */
      uint32_t *data_end;         /* End of data load address in RAM */
      uint32_t *bss_start;        /* Start of BSS in RAM */
    
      /* 0x10 */
      uint32_t *bss_end;          /* End of BSS in RAM */
      void (*entry)(void);        /* Address to jump to */
      uint32_t magic;             /* 32-bit signature, defined below */
      uint32_t version;           /* BCD version number */
    
      /* 0x20 */
      uint32_t *const_start;      /* Start of C++ constructors */
      uint32_t *const_end;        /* Start of C++ constructors */
      uint32_t *heap_start;       /* Start of heap */
      uint32_t *heap_end;         /* End of heap */
    } __attribute__((__packed__));

Notice the "packed" attribute at the end.  That is a special GCC construct that ensures the compiler won't insert optimizing space anywhere in the struct.  This struct is word-aligned, so it's not likely to cause any problems.

However, the actual loader.c file looks like this:

    /* Variables provided by the linker */
    extern uint32_t _textdata;
    extern uint32_t _data;
    extern uint32_t _edata;
    extern uint32_t _bss_start;
    extern uint32_t _bss_end;
    extern uint32_t __init_array_start;
    extern uint32_t __init_array_end;
    extern uint32_t __heap_base__;
    extern uint32_t __heap_end__;
    
    __attribute__((noreturn, naked))
    void Esplanade_Main(void) {
        setup();
        while (1) {
            runCallbacks();
            loop();
        }
    }
    
    __attribute__ ((used, section(".progheader")))
    struct app_header app_header = {
        .data_load_start  = &_textdata,
        .data_start       = &_data,
        .data_end         = &_edata,
        .bss_start        = &_bss_start,
        .bss_end          = &_bss_end,
        .entry            = Esplanade_Main,
        .magic            = APP_MAGIC,
        .version          = APP_VERSION,
        .const_start      = &__init_array_start,
        .const_end        = &__init_array_end,
        .heap_start       = &__heap_base__,
        .heap_end         = &__heap_end__,
    };

There's a lot going on here, but let's take it in steps.

To begin with, there are a large number of "extern uint32_t" values.  We'll see what these are later, but suffice it to say they're linker magic.

Following these values is a function implementation, defined as "noreturn" and "naked".  The "noreturn" attribute is self-explanatory, and means the compiler will warn us if we try to return from the function.  The "naked" is more interesting, and means the compiler will avoid saving state from the previous function.  Since this is the main function, we don't need to save any values from the previous caller, so we can safely omit that.

Finally, the file contains a "struct app_header" that implements the struct described earlier.  Interestingly, it gets the "used" and "section(.progheader)" attributes.  "used" simply means the garbage collector shouldn't throw it away if it thinks we're not using it.  That is the case, since we don't refer to it anywhere else.  The other attribute tells it to place the symbol into the "progheader" section.

Earlier on, we talked about the .bss and .data sections.  C generally decides which section things go in, but sometimes it becomes important to specify exactly where to place data.  You can, for example, mark functions as residing in the .data section, which will cause them to run from RAM.  Here we specify the .progheader section, which we will refer to in the linker script.

Linker Script
--------------

The linker script is where magic happens.  These scripts tell the compiler where to place data and code on disk, in RAM, and in code.  When you take the address of a variable, it is the linker script that maps that to a real value.  Before linking, symbols don't have values.

Consider the following code:

    int func11(void) {
        return 11;
    }
    int var1;
    int * func12(void) {
        return &var1;
    }

Compile the code into an intermediate object file with the following:

    gcc -c file1.c -o file1.o

We can then use the readelf command to get symbol addresses:

    $ readelf --syms file1.o | grep GLOBAL
        Num:    Value          Size Type    Bind   Vis      Ndx Name
          8: 0000000000000000    11 FUNC    GLOBAL DEFAULT    1 func11
          9: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM var1
         10: 000000000000000b    11 FUNC    GLOBAL DEFAULT    1 func12

Notice how the first function starts at offset 0, the first variable at offset 4, and the second function at offset 0xb.

If we add a main() function and actually link in file1.o, we can see that the linker has assigned real addresses to the symbol:

    $ readelf --syms file | grep '\(func\|var\)'
       50: 0000000000400517    11 FUNC    GLOBAL DEFAULT   13 func11
       59: 000000000060103c     4 OBJECT  GLOBAL DEFAULT   25 var1
       63: 0000000000400522    11 FUNC    GLOBAL DEFAULT   13 func12

The GNU linker is called ld.bfd and uses special scripts to determine what sections go where.  It is also capable of creating new symbols that can be referred to by C code.  It also has a few other niceties such as being able to define the entry point and allowing for discarding of sections.

Love-to-Code uses KL02Z32-app.ld to define a Love-to-Code application for the KL02Z32 used in the LtC sticker.

The start defines an entrypoint, which is advice to the debugger as to where program entry starts.  It does not necessarily reflect where the hardware starts execution, and doesn't need to be at any particular address.

The first section of a linker script is the MEMORY section.  This defines all of the memories available to the program.  In the LtC script, we define four sections: flash and ram, which are user-accessible flash storage space and memory, and osflash and osram, which are OS-reserved memory spaces.

We then define three variables that can be used by C code to refer to areas of RAM.  If we take the address of any of these variables in C code, they will give us the values as seen by the linker.

After these variables, the real meat of the script begins.  The SECTIONS section describes where each of the program section in the source ELF file will end up in the output binary file.  The linker provides a register called "." that refers to the current address.  It can be read from or written to, an increments as symbols are loaded into a section.

The first thing we do, naturally, is assign the program origin.  Since we want to start loading code into the flash, we assign "." to be "ORIGIN(flash)".  Now, any symbols will be assigned into the "flash" section.

We then load data into the text section, ensuring it's word-aligned.  The very first thing we put in there is .progheader, which is the special section we put our app_header into.  Because this is the only symbol in this section, it is guaranteed to be at the top of the section, and because we've just started loading data into the flas, the app_header will end up at the top of flash.

Following the progheader we define another symbol, __init_array_start.  This indicates the start of the C++ Constructor array, which is why it is immediately followed by any symbols in the .init_array section, and a termination pointer.  In this way, we obtain an array of function pointers to call in order to initialize the program, and we have pointers to the function as well.

The "data" section is similar to the "text" section, except symbols are loaded into " > ram AT > flash".  This causes data to point to offsets in RAM, but actually be placed into flash.  In this way, initialized values are stored in flash.

The linker file is specified by calling gcc with the -Wl,--script=KL02Z32-app.ld, and a .bin file suitable for writing onto a ROM is created simply by running "objcopy -O binary file.elf file.bin".

Review
------

In this chapter, we've covered the application header, which describes how the program is loaded into RAM.  We've also covered the C linker, which allows us to perform the magic that generates the program header directly from C code.