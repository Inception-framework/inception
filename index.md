# Inception : System-wide Security Testing of Real-World Embedded Systems Software

Inception is a framework to perform security testing of real world firmware programs.
It is build on top of Klee, a symbolic virtual machine that is able to achieve high path coverage for C/C++-based desktop applications. We extended Klee to support testing of mixed languages such as inline assembly, C/C++ and binary code. Futhermore, Inception offers the possibility to precisely define a test stategy depending on the attacker mean. In fact, the memory layout is fully configurable. 

For trusted/complex peripherals, Inception can keep the interactions with the real device to get access to peripherals. This case is usefull to limit the exploration to realistic behaviors. 

However, in some cases peripherals may be under the control of an attacker (UART, USB, SPI, etc...) ,therefore, hardware interactions can be replaced with an abstract model. In this case, Inception will evaluate any possible values returned by this peripherals.


## Authors
Author: Corteggiani, Nassim and  Camurati, Giovanni and Francillon, Aurélien

## Building

There are two ways to install Inception:

* The easiest way is to use our Docker image. 
More details [here](https://github.com/Inception-framework/docker).
* It can be build from source with LLVM 3.6.2
More details [here](https://github.com/Inception-framework/inception).

## Getting started 
[Official samples](https://github.com/Inception-framework/samples).

## Publications
Inception: System-wide security testing of real-world embedded systems software
USENIX Security 2018
