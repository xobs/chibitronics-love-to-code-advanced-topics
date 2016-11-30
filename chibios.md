ChibiOS
=======

The Love-to-Code platform uses ChibiOS as the underlying operating system.  This operating system is designed for small microcontrollers, and provides many of the basic necessities that desktop programmers have become acustomed to.

LtC uses ChibiOS/RT specifically, which means it takes full advantage of the RTOS provided.  This is in contrast to ChibiOS/NIL, which is a severely stripped-down static implementation.  LtC also takes advantage of ChibiOS/HAL, which provides a hardware abstraction layer over many common hardware peripherals.

Threading
--------------

At a minimum, ChibiOS provides operating system basics that are exposed via syscalls as discussed in the [Syscalls](/syscalls.md) chapter.  Here, "basics" means support for preemptive and cooperative multithreading, allowing us to use the system timer to run multiple threads at once.

On the current LtC, user programs have 3072 bytes available.  Each thread takes a minimum of 64 bytes, meaning we could in theory have 48 threads running.  We'll discuss threading more in the [Multithreading](/multithreading.md) chapter.

When threading, it is important to prevent two threads from modifying the same data at the same time.  Mutexes are one way of doing this, where each thread locks the mutex when updating and unlocks it when done.  ChibiOS provides mutex support, which gets exposed directly to the user program via Syscalls.


Memory Management
-----------------

Modern programming languages are very dynamic.  Languages such as Java and Python free users from needing to worry about freeing objects when they're done.  Importantly, it also means they can make decisions at runtime, such as how many lights to attach to their project.

Memory is usually allocated in C by calling malloc(), and freed by calling free().  Embedded systems tend to avoid malloc() and free(), because it means that the amount of memory required at runtime is no longer fixed.  Still, Arduino code makes heavy use of malloc() and free(), so we must provide the functionality.

ChibiOS has a built in allocator that we can take advantage of.  This allocator must be first initialized, but the heap's range is stored in the program header, and the OS loader takes care of initializing the ChibiOS heap allocator.

From that point onward, the normal ChibiOS malloc(), realloc(), and free() commands function as one would expect.


Hardware Abstraction Layer
--------------------------

An important advantage of ChibiOS over alternatives such as FreeRTOS is the incusion of ChibiOS/HAL, which provides a hardware abstraction layer.  The underlying hardware can be swapped out and the HAL replaced, and code will continue to function as it did before.  This is preferrable to writing registers directly, which has a number of drawbacks:

 * Direct register writes are prone to copy-and-paste bugs
 * It's more difficult to share the hardware between threads
 * Code must be placed in the binary, making it larger
 * The OS may already use the hardware, resulting in duplicated code
 * If the hardware changes, the code must also change

This last point is subtle, but is extremely important.  Many variants of the LtC sticker were produced during testing, some of which moved hardware pins around, sometimes swapping Tx and Rx, or using entirely different pins for PWM output.  Because of the HAL, the exact same program can run on all variants of hardware.