<!-- 
.. title: Running Programs on Epiphany
.. slug: running-programs-on-epiphany
.. date: 2015-09-01 13:13:00 UTC-05:00
.. tags:  parallella, epiphany
.. category: 
.. link: 
.. description: 
.. type: text
-->


In the [last post](../communicating-with-epiphany/index.html), we looked at the memory layout and ran a simple demo.
Let's take a closer look at how a program starts running.

For starters, compile and run the program from last time.  We're going to run the program again, this time manually using the low-level mechanism that `e-loader` uses to start programs.

Writing a 1 (SYNC) value to the ILATST register will trigger the Sync interrupt, and cause an existing program to run on the core.

Remember that the registers from each core are memory mapped to the host.
The register addresses are defined in `/opt/adapteva/esdk/tools/host/include/epiphany-hal-data.h`.
We want E_REG_ILATST, which is 0xf042C.

To see if this works, change the input values.
```
parallella@parallella:~$ e-write 0 0 2000 4 5 0
[0x00002000] = 0x00000004
[0x00002004] = 0x00000005
[0x00002008] = 0x00000000

```

Now, write the ILATST register to run the program
```
parallella@parallella:~$ e-write 0 0 0xf042c 1

```

And check the result
```
parallella@parallella:~$ e-read 0 0 0x2000 3
[0x00002000] = 0x00000004
[0x00002004] = 0x00000005
[0x00002008] = 0x00000009

```

It worked!

Okay, now let's look at the details of what happened.

The first 40 bytes of the core-local memory are reserved for the Interrupt Vector Table (IVT).
Each entry is 4 bytes long, and should contain a jump (branch) instruction to the desired code.
The first entry in the table is the Sync interrupt, used to start the program.
(See the [Epiphany Architecture Reference](http://adapteva.com/docs/epiphany_arch_ref.pdf) for the rest of the IVT entries)

We can disassemble the compiled object file with `e-objdump -D` and look for address 0 
(we need -D instead of -d to disassemble all the sections, not just the normal executable sections).
 
This looks promising.  Address of 0, in a section called `ivt_reset`.
```
Disassembly of section ivt_reset:

00000000 <_start>:
   0:   2ce8 0000       b 58 <.normal_start>

```
After the Sync interrupt, control transfers to address 0, and then branches to location 0x58.
The next section in the objdump output has that address.

```
Disassembly of section .reserved_crt0:

00000058 <.normal_start>:
  58:   720b 0002       mov r3,0x90
  5c:   600b 1002       movt r3,0x0
  60:   0d52            jalr r3
```
This loads address 0x90 and jumps to it.
The `mov` instruction loads the lower 16 bits of r3 and the `movt` instruction loads the upper 16 bits.


Now look for that address
```
Disassembly of section .text:

00000090 <_epiphany_start>:
  90:   be0b 27f2       mov sp,0x7ff0
  94:   a00b 3002       movt sp,0x0

   ... 
```

This sets up the stack by loading the stack pointer with 0x7ff0, which is the top of the 32K local address space.
The code calls other routines, which eventually call `main`, but I won't trace it all here.







  
