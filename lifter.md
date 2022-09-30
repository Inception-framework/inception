# Assembly support

## Notes

* we have C code with asm stubs. We want to compile C into high level llvm (llvm-src), compile asm (always decompile each entire function containing asm) to binary and decompile it to llvm (llvm-asm). Finally, we want to merge them to make them coexist and execute properly inside Klee.
* Klee is an interpreter (plus selective symbolic execution, forwarding etc.) for llvm. Therefore the access to global variables and functions is done by symbol (actually mapped to an object stored at at location in the memory of the host). There is a calling convention to pass parameters to functions by putting them is some variables. Functions can define some local variables.
* On the contrary asm uses registers, its instructions can have side effects, it uses addresses etc. etc.
* we have a powerful test framework for comparing results with the real device&#x20;

## Overview

* **instructions support**: load/stores (all types), additions/subtractions, moves, shifts, compares, logical, extend, b label / bx lr / bl target (no target outside function) / bxl rn (only to entry point of functions in the symbol table), msr/mrs (for SP only), mul/umlal. Flags working. IT-blocks working. Rough measure: 120 supported opcodes out of 510.
* **registers and flags** general purpose registers implemented as global variables in llvm. the same for flags (only behavior inside-outside IT blocks is missing because they are not implemented yet). The SP is initialized as it should be on the device. During decompilation, we know the value of PC, so when it is used we can statically compute it. An interesting example is: to access a global variable, we do a pc-relative load to load a pointer (stored in the code), with which we can access the location. We can find this address during decompilation and properly save it into a register.
* **stack**: the stack is implemented as a global variable. llvm-asm functions directly read and write it. llvm-src function do not access it directly.
* **.data,.bss,.interrupt\_vector** are implemented as global variables. Compiler adds the code to initialize .data with the values read from the binary.
* **access to global variables**: we modified the memory management in klee. Now each global variable (including the stack) is "allocated" at the address that can be found in the symbol table (i.e. the same of the device). This way, llvm-asm code that access memory with low level operations (registers used as pointers and low level instructions), can access the right memory object in klee. In other words, llvm-src code r/w a memory object with high-level operations "by name", while llvm-asm code r/w a memory object with low-level operations "by address". To allow the use of external functions, klee allocates  some memory at the same address on the real host, and copied the memory object there. For now, we do not do it. That means we cannot call external functions with hardcoded addresses. E.g. we can call printf("%d",\&var) and printf will access the right var object on the host memory, but we cannot do printf("%d",0xdeadbeef).&#x20;
* **parameters, local variables**: llvm-asm functions directly use the stack and registers for this. On the contrary, llvm-src uses the normal llvm mechanism. The communication between llvm-asm and llvm-src functions happens through a clever use of the calling convention, as explained in the following.
* **llvm-src calls llvm-asm** we add a prologue and epilogue to map the llvm-src calling convention with the low-level convention expected by the llvm-asm code. We can handle all types, but floats and some C++ arrays.
* **llvm-asm calls llvm-src** we need to add a basic block before the call, that maps the low level calling convention with the high-level one (and also for the return). We can handle all types but floats.
* **branches/calls** in llvm-src we have br, call,ret; in asm we have only branches and direct manipulations of the pc (mov, load, add); how to decide whether we are doing a b or a call is not so easy. For now we simply say: b -> branch; bl -> call. If we have a b to the entry point of a function (a way to optimize/share code) we simply transform it into a call to that function, immediately followed by a return.
* **interrupt/exception** we create a special function inception\_interrupt\_handler(address). Klee looks up in the interrupt vector for a given id and calls this function with the right address. This function saves the context, calls the right handler ( inception\_icp(address), see indirect call promotion ) and then restores the context, like the real processor does.
* **indirect calls promotion**: we want to support bxl Rm for calls. We use a simple version of the indirect call promotion technique. Fracture creates an icp function which receives Rm as input and uses a switch statment to compare it with every function address. For each case, Fracture can create a proper direct call to the function, handling args and return parameters. Fracture can do this because it has access to the symbol table and to function objects in llvm. Base cases are working. We modified klee to "put" (we keep a mapping between host and device) functions at the same address as the one on the device, similarly to what we do for global variables. As in the original Klee, we cannot call code in the middle of a function.
* **IT instruction and conditional execution** here we make the assumption that nobody can modify the context during an interruption, thus nobody can modify ITSTATE during the execution of an IT block. This will be guaranteed by a checker that catches illegal accesses to the stack where the context is saved. Thanks to this assumption, we can know at decompile time which instructions belong to a block and how (condition). For each of them, we add a check on the condition, that skips the instruction if necessary. We need to add it to each of them, because some instructions can change the the flags inside an it block. We keep a dummy IT instruction.

## Main missing points

* **instruction atomicity** if an instruction is atomic, then the sequence of llvm-asm instructions with which we model it must be atomic. Possible solution: call inception\_mask interrupts before the sequence.
* **Support .s files (not only asm inline)** not complete for now
* **MSR,MRS instruction** for now we have only SP working
* **mul/div** (we only have mul/umlal for embedtls)
* **PSP,MSP stack**
* **modes etc. in general**

## More details on our approach

### Classification of Thumb/Thumb2 instruction set:

Instructions of the ARMV7-M instructions set are either 16bit(Thumb) or 32bits(Thumb2). The encoding of an instruction depends on which registers are accessed.

Instructions can be divided into groups following how they influence the state of the system (Registers, Memory, Flags, Internal Core State).

Conditionally executed:

* Executed if the flag satisfy the condition.
* If the condition is not satisfy, a NOP is executed.
* Instructions are : 16-bit B, 32-bit B, CBNZ, CBZ (Compare and Branch on Zero/Non Zero), a 16-bit If-Then instruction that makes up to four following instructions conditional.
* Non Thumb instruction excepted the list above can be conditional.

Memory addressing :

* Load/Store & LTC/STC are the only way to access memory.

Register access :

* Only MSR/MRS & LTC/STC can access to status the Application Program Status Register(APSR)

Data-processing instructions :

* Only access to top and low level registers (R0-R7, R8-R12, sometimes more SP/PC).
* Shift, Mul, Div, Saturating, Packing/unpacking, miscellaneous instructions.
* Updates the flags

Control flow instruction :

* Direct branch to target address
* Compare and branch : CBNZ, CBZ
* BL/BLX call to a subroutine
* Instruction affecting PC (ADD, SUB, LOAD, MOV)
* Supervisor Call : SVC&#x20;
*   Table Branch Byte causes a PC-relative forward branch using a table of single byte&#x20;

    offsets.
* BKPT : can generate a debug monitor exception

### For each group of instructions, how does Inception model the ARMV7-M architecture behavior ?

#### How the system is modeled ?

Inception requires the binary under test and an abstract representation of this binary which is called bitcode. The bitcode contains all the different memory areas present on the binary code (such as the stack, heap, bss, data) in order to preserve the memory layout.

| Name                  | Model            | Address        | Relocatable | Access Right | Comment                                                                                                   |
| --------------------- | ---------------- | -------------- | ----------- | ------------ | --------------------------------------------------------------------------------------------------------- |
| Stack                 | Global array obj | same as device | no          | Unsupported  |                                                                                                           |
| Heap                  | Global array obj | same as device | no          | Unsupported  | still some collisions between variables declared into the assembly and c-source : symbols share addresses |
| Bss                   | Global array obj | same as device | no          | Unsupported  |                                                                                                           |
| Data                  | Global array obj | same as device | no          | Unsupported  |                                                                                                           |
| Peripheral            | External Object  | same as device | no          | Unsupported  |                                                                                                           |
| Co-processor Register | Global Object    |                | Unsupported |              |                                                                                                           |
| General Registers     | Global Object    |                |             |              |                                                                                                           |
| Interrupts table      | Global Object    | same as device | no          | Unsupported  |                                                                                                           |
| Flags                 | Global Object    |                |             |              |                                                                                                           |

In general, one assembly instruction is translated in a sequence of bitcode instructions where the size vary from 2 to XX. **This difference affects the analysis in different way ?**

Memory addressing instructions can access to the overall memory that have been declared above. Actually there is no support for Memory Protection Unit (MPU) or execution right (privileged/unprivileged). Moreover, bitcode generated from assembly code inherit from the poverty of syntactic information. The precision is affected and makes 'impossible' the detection of buffer-overflow when executed bitcode generated from assembly source.

Data-processing instructions updates the flag which affects indirectly the control-flow of the execution. Flags have the same behavior than onto the device but require much more time to be executed.
