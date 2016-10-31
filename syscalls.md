A program without a way to communicate with the world might as well not run at all.  Programs use libraries to handle input and output, as well as other features such as pausing execution and handling specialized hardware.  Most embedded hardware platforms link the library code in with the final executable.  While simple, this makes the resulting binary larger, and means the same code is part of every binary that is ever built.

Code that is part of every binary is a prime candidate for making into a callable library of some sort.  There are a number of ways to call libraries, from having a well-defined offset with a function table, to performing some illegal operation such as a divide-by-zero and having the operating system intercept the error.  Most desktop operating systems have a loader that patches addresses when the program starts up, based on text strings in the program.

Syscalls are special library functions that are handled by the operating system.  Syscalls are numbered, and on some platforms are well-defined.  For example, on Linux syscall #1 tells the operating system to quit, syscall #3 reads from a file, and syscall #4 writes to a file.  These syscalls are well-defined, and the number is guaranteed to never change.  There are syscalls for everything from creating directories to setting the hostname, and the fact that they are a single number allows the kernel to use it as an index into a syscall table.

The ARM chip provides a way to make a syscall.  The **svc** instruction causes the chip to call a special interrupt handler.  In effect, the hardware turns the **svc** instruction into a function call. The "svc" instruction has been around since the original chip, although it used to be called **swi** which stands for "software interrupt".

## The **svc** instruction ##

When an **svc** instruction is executed, the chip issues a function call to the SVC_Handler(void) function.  We can use syscalls to implement a library system.  We assign a number for each syscall we want to implement, and load the resulting addresses into a table.

The actual syscall takes no parameters, but as a matter of an implementation detail is written "svc #XX" and is encoded as a two-byte value: 0xdfXX.  The eight-bit "XX" field can be any value, although we can still read the actual instruction and we have access to the data.

When a function call is made, even an interrupt call, the address of the thing calling the function (the "callee") is stored on the stack.  To get the eight-bit value, all we need to do is peek at the address of the calling function, then step backwards a bit and read the instruction as data.  We can then use that as an index into a table and resolve our syscall to a real address. 

## Generating the syscall table ##

In LtC, the table of syscalls is stored in a file called syscalls-db.txt.  It's a simple text file that defines function names and the symbols they point to.  Each line of the file that is not blank and doesn't begin with a "#" represents one syscall.

Each syscall can have multiple names.  For example, some C++ code insists on calling \_atexit() or \_\_aeabi\_atexit() in order to register functions to call when the program quits.  Since our program never quits, we don't need to actually implement these functions.  So it becomes useful to map each of these to a function that does nothing.  The format of the text file allows for this many-to-one mapping, and allows for multiple functions in the application code to call the same function in the operating system.

An earlier version of the syscall system supported C++, including constructors, destructors, and member functions.  However, this idea was abandoned as C++ does not guarantee name mangling, and constructors and destroctors don't have addresses.  Because of these limitations, support for C++ was deprecated, and will be removed in a subsequent version.

A perl script reads the file and generates two output files: One for application software to link to, and one for the operating system to link against.

The application code consists of a large number of functions.  Each function is largely the same: A naked function that simply calls asm("svc #num").  The "naked" attribute ensures that no extra setup and teardown, so the compiler will not push any data onto the stack.  The function will just consist of the "svc" call.

The operating system table simply lists functions.  The linker resolves these values to addresses when everything is compiled.  That way, the operating system simply needs to find the start of the table, add the value from the **svc** instruction, and call the function.

## SVC_Handler Implementation ##

The actual implementation of the handler is not very long, and most of it is taken up with housekeeping.  The function is written in Thumb2 assembly, and is in svc\_handler.s.

The function starts by checking the contents of the link register.  If it's 4, then the **svc** instruction was issued by an interrupt handler.  If it's not, then it was called by a regular function.  Dependong on which, the stack will be stored in the **PSP** register.  Otherwise, it will be in the **MSP** register.  The appropriate value is loaded into **r0**, and execution continues.

Once it knows where to look, it actually starts looking.  The following registers are available as offsets from r0:

    [r0 + 0] - r0
    [r0 + 4] - r1
    [r0 + 8] - r2
    [r0 + 12] - r3
    [r0 + 16] - r12
    [r0 + 20] - r14 (pc)

We're interested in **pc**, so we load [r0 + 20] into **r2**.  This gives us the address where the program will return to when it returns from the interrupt handler. 

The **svc** instruction is two bytes before the return value, so we subtract 2 from the **pc** value in **r3**.  We then load one byte, which gives us the value of the **svc** instruction.

All that's left is to use the value as an index into the syscall table.

The table is stored in a symbol called SysCall\_Table.  We load the address of SysCall\_Table, multiply the **svc** index by two, then add the result together.  This gives us the entry point for the target function.

Throughout this, we've kept the original return **pc**.  We can load the target function into the return **pc**, then exit the interrupt handler, and execution will continue in the target function.

Because the function signatures are the same, the compiler has set up arguments when it called our naked function.  When the handler returns, it will be as if the code had called the original function.


## Compatibility and Efficiency ##

The syscall interface allows for us to keep compatibility across devices.  If a faster product is released, then calls to serialWrite() and delayMicroseconds() can continue to work.  A future device could add USB support, and applications would work the same.

Additionally, syscalls are only two bytes.  They are as efficient as function calls, but require significantly less space.  For this reason, a simple LED Blink loop ends up at under 256 bytes, including the program header and checksums.

Syscalls give LtC the best of both worlds: compatibility and efficiency, with no downside.