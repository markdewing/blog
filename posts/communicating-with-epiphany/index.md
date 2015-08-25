<!-- 
.. title: Communicating with Epiphany
.. slug: communicating-with-epiphany
.. date: 2015-08-25 14:19:00 UTC-05:00
.. tags: parallella, epiphany
.. category: 
.. link: 
.. description: 
.. type: text
-->


This post is going look at the Epiphany memory map, and give a very simple example demonstration.
I will skim over some background information that is covered elsewhere.
See the following posts and resources that describe the Parallella and Epiphany architectures.

Resources
---------
Parallella Chronicles

* [Part One: Introduction](https://www.parallella.org/2014/11/25/parallella-chronicles-part-one-2/)
* [Part Two: The Software Development Kit](https://www.parallella.org/2014/12/15/parallella-chronicles-part-two-2/)
* [Part Three: "Hello World"](https://www.parallella.org/2015/01/14/parallella-chronicles-part-three/)
* [Part Five: The Epiphany Memory Map](https://www.parallella.org/2015/02/28/parallella-chronicles-part-five/)

Technical Musings

* [Overview of Epiphany Architecture](http://suzannejmatthews.github.io/2015/06/02/epiphany-overview/)
* [Hello Epiphany](http://suzannejmatthews.github.io/2015/06/03/epiphany-hello-world/)

Manuals

* [Epiphany Architecture Reference Manual](http://adapteva.com/docs/epiphany_arch_ref.pdf)
* [Epiphany SDK Reference Manual](http://adapteva.com/docs/epiphany_sdk_ref.pdf)

Source code

* [epiphany-libs](https://github.com/adapteva/epiphany-libs) repo on github with the e-hal and various utilities.
The epiphany-sdk repo contains download and build script for the GNU toolchain.


This post is going to be written from the perspective of a PC programmer.
Desktop operating systems use virtual memory, and programmers don't have to think about hardware addresses very much. 
The relative addresses inside each process matter most.
Many of the addresses on the Epiphany are fixed, or fixed relative to each core, and require more 'hard-coding' of addresses.
Although most of that is accomplished through the toolchain, it is useful to understand when programming the board.
(Programmers of embedded systems are more used to this sort of memory layout control.)

Memory Layout
-------------

The Epiphany contains onboard RAM (32K per core). This called 'local' or 'core-local' memory, and is the fastest to access.

There is a larger section of memory (32MB) that is reserved from top of the SDRAM and shared with the Epiphany.
This is called 'external memory' from the perspective of the Epiphany.  It's also called 'shared memory'.
It is much slower to access from the Epiphany.

The shared memory and memory locations inside the chip are both mapped in the address space of the host, and can be access by the host.
Locations in the chip include

* 32K local to each core
* Registers on each core
* Chip-wide registers 

Something interesting here is that the registers all have memory mappings.
That means the registers can be accessed by the host (and other cores) by simply reading or writing a specific memory location.
(It is important to note that register values are only valid when the core is halted.)

Epiphany Software Development Kit
---------------------------------
The ESDK contains some utilities to access these memory regions from the command line.
The commands `e-read` and `e-write` are used to read and write the locations.
To access the core-local memory, use row/column coordinate of the core (0-3 for each), followed by the offset.
For reading, optionally add the number of 4-byte words to read.  For writing, give a list of 4-byte word values.

For example, to read 8 bytes from core (0,0)
```
parallella@parallella:~$ e-read 0 0 0x100 2
[0x00000100] = 0x782de028
[0x00000104] = 0x0d906c89

```

To access the external memory, use a single -1 instead of the row,col coordinates.
```
parallella@parallella:~$ e-write -1 0 1 2 3
[0x00000000] = 0x00000001
[0x00000004] = 0x00000002
[0x00000008] = 0x00000003

```

Upon power up, it appears the memory is filled with random values.
The `epiphany-examples` directory contains some useful utilities in the `apps` directory.
To fill memory with some values, use `e-fill-mem` (build it by running the `build.sh` script first)

To zero all the local memory in core 0, 0:
```
parallella@parallella:~/epiphany-examples/apps/e-fill-mem$ ./bin/e-fill-mem.elf  0 0 1 1 8192 0
```

Verify a few locations
```
parallella@parallella:~/epiphany-examples/apps/e-fill-mem$ e-read 0 0 0 4
[0x00000000] = 0x00000000
[0x00000004] = 0x00000000
[0x00000008] = 0x00000000
[0x0000000c] = 0x00000000

```

Nostalgia sidebar: If you want to reminisce about the days of Commodore 64, Apple II's and other microcomputers, alias `e-read` and `e-write` to `peek` and `poke`. (For the bash shell that would be `alias peek=e-read` and `alias poke=e-write`)


Simple example of local memory access
-------------------------------------

To solidify understanding of how this works, let's try a simple program that adds two numbers in core-local memory,
and saves the result to another location in core-local memory.  We will use the command line tools to set memory and verify
the operation. 

The 32KB of local memory puts all the offsets in the range 0x0000 - 0x8000.   Let's choose a base location 0x2000, which will be above the executable code, and below the stack.

Start with the following C program (mem.c)
```c
// Demonstrate local memory access at a low level

int main()
{
    // Location in local memory will not interfere
    // with the executable or the stack
    int *outbuf = (int *)0x2000;
    int a = outbuf[0];
    int b = outbuf[1];
    outbuf[2] = a + b;

    return 0;
}
```

Compile with 
```
e-gcc mem.c -T /opt/adapteva/esdk/bsps/current/fast.ldf -o mem.elf
```

The -T option refers to a linker script, which controls where various pieces of the executable are placed in memory.

Set the initial memory locations
```
parallella@parallella:~$ e-write 0 0 0x2000 1 2 0
[0x00002000] = 0x00000001
[0x00002004] = 0x00000002
[0x00002008] = 0x00000000
```

Load and run the program (the -s option to e-loader runs the program after loading)
```
parallella@parallella:~$ e-loader -s mem.elf 
Loading program "mem.elf" on cores (0,0)-(0,0)
e_set_loader_verbosity(): setting loader verbosity to 1.
e_load_group(): loading ELF file mem.elf ...
e_load_group(): send SYNC signal to core (0,0)...
e_start(): SYNC (0xf042c) to core (0,0)...
e_start(): done.
e_load_group(): done.
e_load_group(): done loading.
e_load_group(): closed connection.
e_load_group(): leaving loader.

```

Now verify the program produced the expected result:

```
parallella@parallella:~$ e-read 0 0 0x2000 3
[0x00002000] = 0x00000001
[0x00002004] = 0x00000002
[0x00002008] = 0x00000003

```
Yes.  It worked!


Now we've seen some concrete low-level details on how memory works on the Parallella and Epiphany.
Next time I want to look the e-loader in more detail, and how programs start running on the cores.
