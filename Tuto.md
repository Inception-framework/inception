# Tutorial

In this tutorial, we present one use case to demonstrate the usefulness of using Inception for the analysis of a real-world firmware soure-code.
The firmware under test is a Web server for the board LPC1850-DB1 from Diolan. This board hosts an NXP LPC1850 chip based on a ARM Cortex-M3.
The firmware contains 3 main elements that are:

* A light TCP/IP stack.
* A light HTTP server.
* Bare-metal drivers.

## Requirements 

You can download the code directly from Diolan website, however we recommand you to use our version for two simple reasons.
First, we added compilation scripts that avoid installing more than necessary. Second, we modify the source code to add a vulnerability.

```
git clone https://github.com/Inception-framework/lpc18xx-demos.git
```

Before going futher in this tutorial, you need to install Inception on your machine natively or through a Docker container.
We recommand you to use our Docker container that avoids dependencies issues.

## First - cross-compilation

To perform its analysis, Inception requires two inputs. A target ELF binary with symbols and a bitcode in LLVM IR. 
The bitcode is the intermediate representation used by the LLVM compiler framework.
We use the LLVM frontend for C/C++ code that is Clang. Our script build for you all the required files.

```
cd lpc18xx-demos

./build.sh Web
```

## Second - setting the memory model

Remember that Inception, is a re-hosting system. Therefore, it needs some information about the firmware environment.
There is different kind of memory for a firmware. For instance, memory mapped registers. When accessing mapped addresses, 
memory bus transactions are seamlessly forwarded to hardware peripherals. 
This is a common communication channel between firmware and hardware.
In some cases, you may need to keep this interaction while testing on Inception. 
This is generaly a good way to prune the explored states to realistic cases. 
In some other, you can decide to randomized or to consider all the possible values during the analysis (i.e. symbolic value).
This is generaly the case when inputs come from an untrusted element (e.g. ethernet buffer).
To summarize, this feature is by far the most important as it enables you to define the test strategy to follow during the analysis.
This strategy is mainly based on modeling the memory model of the firmware.

In the purpose of testing the Web server, we will focus on the symbolic strategies but you can find below all the configurations.

```
"MemoryModel": [
  {"name": 'Ethernet_buffer', "base": '0x40000000', "size": '256', "strategy": 'fixed|randomized|realistic|concrete'}
]
```

## Third - starting the analyzer

inception --elf web.elf --bitcode web.bc 

## Inspection the analysis results



