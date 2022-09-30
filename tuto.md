# Tutorial

In this tutorial, we present one use case to demonstrate the usefulness of using Inception for the analysis of a real-world firmware soure-code. The firmware under test is a Web server for the board LPC1850-DB1 from Diolan. This board hosts an NXP LPC1850 chip based on a ARM Cortex-M3. The firmware contains 3 main elements that are:

* A light TCP/IP stack.
* A light HTTP server.
* Bare-metal drivers.

## Requirements

You can download the code directly from Diolan website, however we recommand you to use our version for two simple reasons. First, we added compilation scripts that avoid installing more than necessary. Second, we modified the source code to add a vulnerability.

```
git clone https://github.com/Inception-framework/lpc18xx-demos.git
```

Before going futher in this tutorial, you need to install Inception on your machine natively or through a Docker container. We recommand you to use our Docker container that avoids dependencies issues. This tutorial requires Inception v2 that is not publicly available yet.

## First - cross-compilation

To perform its analysis, Inception requires two inputs. A target ELF binary with symbols and a bitcode in LLVM IR. The bitcode is a semantic model in LLVM IR. We use the LLVM frontend for C/C++ code that is Clang. Our Makefile script will automatically build all the required files.

```
cd lpc18xx-demos

PROJECT=Web make
```

To get more details about this, we describe below the 3 main steps:

* Classic compilation using GCC or Clang. During our experiment, we use GCC because of its prevalence. However, we recommend using Clang that performs interesting sanity checks, and avoids errors due to symbols naming convention. We observed this last case when testing C++ source code. This step emits an ELF file for ARM32 target with debug information. These debug information are not always required, Inception can analyze binary code but requires debug symbols to support execution of high level programming language (C/C++) and binary code at the same time.
* High-IR compilation. Inception uses the LLVM intermediate representation (LLVM IR) to create a semantic model that is a representation of the firmware semantic. We used Clang to emit this from source-code (C/C++).
* Low-IR compilation. During last step we emit LLVM IR for C/C++ only. However, our firmware may have binary dependencies or assembly functions. For this reason, we designed the Inception lift-and-merge approach that is implemented by the Inception translator. During this step all low level code (assembly, binary) are translated into LLVM IR. Then a glue code is inserted to enables High-IR and Low-IR to interact together. The resulting code is then compiled to form a bitcode file named bin.bc.

## Second - setting the memory model

Remember that Inception, is a re-hosting system. Therefore, it needs some information about the firmware environment. There is different kind of memory for a firmware. For instance, memory mapped registers. When accessing mapped addresses, memory bus transactions are seamlessly forwarded to hardware peripherals. This is a common communication channel between firmware and hardware. In some cases, you may need to keep this interaction while testing on Inception. This is generaly a good way to prune the explored states to realistic cases. In some other, you can decide to randomized or to consider all the possible values during the analysis (i.e. symbolic value). This is generaly the case when inputs come from an untrusted element (e.g. ethernet buffer). To summarize, this feature is by far the most important as it enables you to define the test strategy to follow during the analysis. This strategy is mainly based on modeling the memory model of the firmware that are the programs inputs.

In the purpose of testing the Web server, we will focus on the symbolic strategies but you can find below all the supported configurations.

* default    : when nothing is specified, the memory area is local to the analyzer and initialized to zero.
* randomized : the memory area is initialized to a random value.
* symbolic   : use constraint instead of concrete value.
* forwarded  : redirect i/o to the real device (this requires the Inception debugger to be attached).

```
"memory_model": [
  {"name": 'REGISTER', "base": '0x40000000', "size": '256', "strategy": 'randomized|realistic|forwarded|symbolic'}
]
```

## Third - setting the interrupt model

One second and important challenge when analyzing firmware programs is the interrupt mechanim. Hardware peripherals uses interrupt signals to notify the firmware that a task completed. To process interrupt, the CPU (Cortex M) stops current code execution, stacks context (registers) on the stack, resolve the interrupt handler address using the interrupt vector and then update the instruction pointer to start executing the interrupt handler. To support interrupt, the analyzer needs to emulate all these parts. Two behaviors are supported:

*   real communication with the interrupt controler. This mode is automatically used when the Inception debugger is attached.

    However, it needs to run a software stub on the device to catch and forward interrupt. Since the set-up is not easy, we will focus on the second method.
*   interrupt modeling. This is a new feature of Inception where interrupt are inserted during the analysis following a user-defined frequency. This enables users to choose how complete this analysis should work. A too short frequency can lead in an exponential&#x20;

    growth in the number of states.

The example below will generate systick interrupt very 1000 instructions.

```
{
    "interrupt_model" : [
      { "id": 15, "frequency" : 1000}
    ]
}
```

## Fourth - starting the analyzer

```
inception --elf ./lpc18xx-demos/bin.elf --bitcode ./lpc18xx-demos/bin.bc \
--allocate-determ --allocate-determ-start-address 0x90000000 --allocate-determ-size 10000 \
--mem_conf_file ./mem.json --interrupt_conf_file ./irq.json
```

During the analysis, inception triggers SysTick interrupt every 1000 instructions and print a debug message on the screen.

![Analysis](.gitbook/assets/analysis.svg)

**Note** options starting with 'allocate-determ' are used to force 32bits address space when interpreting High-IR LLVM IR. This avoids pointer missmatch.

## Analysis Results

On the original code, at each SysTick interrupt a counter is incremented. When this timer is above 20, the TCP stack is refreshed. Our synthetic vulnerability arrives at this moment.

After running inception for a few second, it prints an error message that indicates the presence of an out of bound pointer on line 35 in main.c. This experiment demonstrates how Inception can be configured to detect memory corruption on firmware programs using a custom interrupt model. Even simple, this example is representative of issues on real world firmware where bug triger condition are often related to interrupt and hardware i/o.

```
KLEE: ERROR: lpc18xx-demos/Examples/Web/main.c:35: memory error: out of bound pointer
```
