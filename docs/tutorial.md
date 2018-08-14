# Quick tutorial

## Prerequisite

Make sure you have built our docker image as explained [here](https://github.com/Inception-framework/docker).

## A simple example

### Run docker

```docker run -i -t inception/inception /bin/bash```

### A simple program

To get starded with Inception, we will use a very simple program that calls a recursive Fibonacci function.
We have two version of the function, one is in C and serves as a golden model, the other one is written in assembly.
The program calls both and compares the result.

```
#include <assert.h>

__attribute__((naked))
int fibonacci_asm(int n){
  __asm volatile("push	{r4, r7, lr}");
  __asm volatile("sub	sp, #12");
  __asm volatile("add	r7, sp, #0");
  __asm volatile("str	r0, [r7, #4]");
  __asm volatile("ldr	r3, [r7, #4]");
  __asm volatile("cmp	r3, #3");
  __asm volatile("bhs	continue");
  __asm volatile("movs	r3, #1");
  __asm volatile("b.n	stop");
  __asm volatile("continue:");
  __asm volatile("ldr	r3, [r7, #4]");
  __asm volatile("subs	r3, #1");
  __asm volatile("mov	r0, r3");
  __asm volatile("bl	fibonacci_asm");
  __asm volatile("mov	r4, r0");
  __asm volatile("ldr	r3, [r7, #4]");
  __asm volatile("subs	r3, #2");
  __asm volatile("mov	r0, r3");
  __asm volatile("bl	fibonacci_asm");
  __asm volatile("mov	r3, r0");
  __asm volatile("add	r3, r4");
  __asm volatile("stop:");
  __asm volatile("mov	r0, r3");
  __asm volatile("adds	r7, #12");
  __asm volatile("mov	sp, r7");
  __asm volatile("pop	{r4, r7, pc}");
}

int fibonacci_golden(int n){
  if(n <= 2) return 1;
  return fibonacci_golden(n-1)+fibonacci_golden(n-2);
}

int main(void){

  int x = 10;
  int y_asm = fibonacci_asm(x);
  int y_golden = fibonacci_golden(x);  

  #ifdef KLEE
  printf("fibonacci1(%d) = %d\n",x,y_asm);
  printf("fibonacci_golden(%d) = %d\n",x,y_golden);
  assert(y_asm == y_golden);
  printf("ok\n\n");
  #endif
}

```

### Compilation for the target device

First, we need to compile the program for the native platform, to obtain the binary code of the assembly function.
For that we also need a linker script and a startup file.
For example, you can use the following ones. The most important thing is to define the .interrupt_vector and .stack sections,
which are expected by Inception-analyzer for running the program. 

```
OUTPUT_FORMAT("elf32-littlearm")

OUTPUT_ARCH(arm)

ENTRY(Reset_Handler)

MEMORY
{
  VECTOR (rx) : ORIGIN = 0x10000000, LENGTH = 0x400   /* 256k */
  FLASH (rx) : ORIGIN = 0x10000400, LENGTH = 0xFFFBFF /*0xFFFFFF  256k */
  RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 0xFFFF   /*  32k */
}

STACK_SIZE = 8200;

SECTIONS
{
  .interrupt_vector :
  {
    *(.cs3.interrupt_vector);
  } > VECTOR

 .code :
 {
   CREATE_OBJECT_SYMBOLS
   *(.text .text.*);
 } > FLASH

 .data :
 {
   *(.rodata .rodata.*);
   *(.data .data.*);
   *(.bss .bss.*);
 } > RAM

 .bss : {
    *(.bss .bss.*)
    *(COMMON)
    . = ALIGN(4);
 } > FLASH

 .stack : {
  _stack_start_ = .;
  KEEP(*(STACK))
  . += STACK_SIZE;
  . = ALIGN (4);
  _stack_end_ = .;
 } > RAM

 STACK = _stack_start_;
}
```

```
.syntax unified

/* Vector table definition */
 .section ".cs3.interrupt_vector"
 .long  STACK                       /* Top of Stack                 */
 .long  Reset_Handler               /* Reset Handler                */
 .long  NMI_Handler                 /* NMI Handler                  */
 .long  HardFault_Handler           /* Hard Fault Handler           */
 .long  MemManage_Handler           /* MPU Fault Handler            */
 .long  BusFault_Handler            /* Bus Fault Handler            */
 .long  UsageFault_Handler          /* Usage Fault Handler          */
 .long  0                           /* Reserved                     */
 .long  0                           /* Reserved                     */
 .long  0                           /* Reserved                     */
 .long  0                           /* Reserved                     */
 .long  SVC_Handler                 /* SVCall Handler               */
 .long  DebugMon_Handler            /* Debug Monitor Handler        */
 .long  0                           /* Reserved                     */
 .long  PendSV_Handler              /* PendSV Handler               */
 .long  SysTick_Handler             /* SysTick Handler              */

.section ".text"
.weak Reset_Handler
.type Reset_Handler, %function
.thumb_func

.global Reset_Handler
Reset_Handler:
	MOV R0, R0
	B .

/* This is how the lazy guy doing it: by aliasing all the
 * interrupts into single address
 */
.thumb_func
.weak NMI_Handler
NMI_Handler:
	B .

.thumb_func
HardFault_Handler:
        B .

.thumb_func
MemManage_Handler:
        B .

.thumb_func
BusFault_Handler:
        B .

.thumb_func
UsageFault_Handler:
        B .

.thumb_func
.weak SVC_Handler
.type SVC_Handler, %function
SVC_Handler:
        B .

.thumb_func
DebugMon_Handler:
        B .

.thumb_func
PendSV_Handler:
        B .

.thumb_func
SysTick_Handler:
B .
```

You can then use the arm toolchain to generate an elf.

```arm-none-eabi-gcc -march=armv7-m -mthumb -mcpu=cortex-m3 -Wa,-mimplicit-it=thumb -g -c main.c -o main.o```
```arm-none-eabi-ld -T ./link.ld main.o -o main.elf```

### Configuration and execution
