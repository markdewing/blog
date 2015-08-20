<!-- 
.. title: Introduction to Parallella
.. slug: introduction-to-parallella
.. date: 2015-08-20 15:08:00 UTC-05:00
.. tags: parallella, epiphany, zynq
.. category: 
.. link: 
.. description: 
.. type: text
-->

The Parallella is a single board computer with a dual core ARM and a 16 core Epiphany coprocessor. 
I've had some boards sitting around after backing the Kickstarter, and now I've finally started to play with them.

The main purpose of the board is to highlight the Ephiphany coprocessor, but it has other interesting
resources as well.  I'd like to look into how to use each of them.

Resources to program:

* Xilinx Zynq (7010 or 7020), which contains
    * dual core ARM Cortex A9 processors (with NEON SIMD instructions)
    * FPGA
* Epiphany 16 core coprocessor (simple cores in a grid)

See the website ([parallella.org](http://parallella.org)) for more [details of the board](http://parallella.org/board).

After getting the system set up and running according to the [directions](https://www.parallella.org/quick-start/), the first question is how 
to compile code?   Since there are two architectures on the board, it gets a bit more complex.

* Regular PC (in my case, 64 bit, running Ubuntu) - the host for cross compilation, targeting either the ARM cores or the Epiphany.
* ARM on Zynq - can be a cross-compilation target, can compile for itself, or can compile for the Epiphany
* Epiphany - only a target

While code can be compiled on the board, compiling on host PC can have some definite advantages with much larger resources of disk space, disk speed, etc.
However, setting up projects for cross-compiliation can be more challenging.


#Cross compiling to ARM

On Ubuntu, this turns out to be fairly easy - the compiler packages that target ARM are already available in the repository.

Using the Ubuntu Software Center (or Synaptic, or the apt- tools, as you prefer), install the following packages

* gcc-arm-linux-gnueabihf
* binutils-arm-linux-gnueabihf

Selecting these should install the necessary dependencies (some listed here):

* libc6-armhf-cross
* libc6-dev-armhf-cross
* cpp-arm-linux-gnueabihf
* gcc-4.8-multilib-arm-linux-gnueabihf

(By the way, the 'hf' at the end stands for 'Hard Float' - it means the processor has floating point in hardware)

See this [forum post](https://parallella.org/forums/viewtopic.php?f=13&t=935) for more information.  That post also contains instructions for setting up Eclipse (I'm more partial to the command line).

To cross compile using the command line, all the normal compiler tools are prefixed with `arm-linux-gnueabihf`.  Use `arm-linux-gnueabihf-gcc -o hello hello.c` to compile a simple example.

Run `file` on the output file to verify it compiled as an ARM executable.


## Clang
Compiling with clang needs at least the include and lib files from the 'libc6-*-armhf-cross' packages.

Assuming the version of clang is built to output the 'arm' target, the following should work
```
clang -target arm-linux-guneabihf -I /usr/arm-linux-gnueabihf/include hello.cpp
```


#Cross compiling to Epiphany

These are the tools in the ESDK.

If using the ARM as a host, the ESDK is already in the microSD images and the tools are in the path (`\opt\adapteva\esdk\tools\e-gnu\bin`)
The tools are prefixed with `e-`.  Use `e-gcc` to invoke the compiler.

For a Linux host, download and install the ESDK from the website (under `Software -> Pre-built -> Epiphany SDK`)([direct link](ftp://ftp.parallella.org/esdk)).  Look for 'linux_x86_64' in the file name.

The ESDK has examples you can compile and run.  Sometime later I want to take a closer look at how the Epiphany files are loaded to the coprocessor and run.
