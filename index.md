# Inception : System-wide Security Testing of Real-World Embedded Systems Software

Inception is a framework to perform security testing of real world firmware programs.

***Inception-translator*** generates a unified intermediate representation from C/C++ source code that also contains inline
assembly (or statically linked binary code). This representation is based on LLVM bytecode preserves as much as possible of the high-level semantics of the source code. This is useful to detect memory vulnerabilities. For example, knowing the size of a buffer makes it possible to detect an overflow. We call the intermediate representation resulting from this process mixed-IR.
 
***Inception-analyzer*** is a symbolic virtual machine built as an extension of the well knonw [Klee](https://klee.github.io/) tool. We have extended the memory management system of Klee in order to support our mixed-IR code, and to consider the memory mapped registers of peripherals. Inception may consider them as untrusted unconstrained symbolic inputs, or map those registers to the physical registers of a real device (using Inception-debbuger). Furthermore, we have extended Klee in order to support interrupts and multithreading.  

***Inception-debugger*** is a fast USB3.0 JTAG debugger (currently supporting ARM AHB-AP debug port). 
Inception-analyzer uses it to redirect read/write memory accesses and interrupts to a real target device.
Inception-debugger requires off-the-shelf hardware (Cypress Semiconductor FX3, Digilent ZYBO Zinq-7000 FPGA).

## Authors

Nassim Corteggiani, Giovanni Camurati, Aurélien Francillon.

## Getting started

### Installing

The easiest way is to use our Docker image. 
More details on how to compile it [here](https://github.com/Inception-framework/docker).
Alternatively, you can bulld the necessary components on your machine (you can use our docker file as a reference). 

## Publications

> Inception: System-wide security testing of real-world embedded systems software   
> Nassim Corteggiani, Giovanni Camurati, Aurélien Francillon   
> USENIX Security 2018

## Other resources/coverage 

> [Open-Source Framework Detects Memory Corruptions in Embedded Software](https://www.maximintegrated.com/en/design/blog/open-source-framework.html)

