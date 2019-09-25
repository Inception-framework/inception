# inception debugger

The Inception debugger is a powerful device that answers to 2 main needs:

* Low latency during read/write operations from host to device. When rehosting firmware execution while keeping hardware interactions, one main challenge it is to avoid that this communication become the bottleneck. We show this limitation with Avatar [1].

* Low latency interrupt forwarding. Firmware programs are generaly interrupt-driven which means that the firmware execution can be interrupted by hardware peripherals to execute certain routines. Inception is the first work that correctly forward the interrupt signal between device and rehosted platform while keeping the device state and the rehosted platform synchornized. 

**Note** We acknowledge that the debugger has not a wide scope of supported targets such as commercial JTAG probes. Therefore, we are working on a new version that enables users to easily customize the debugger's FPGA behavior without modfying its circuitry.

## System Description

The debugger is not a single piece but a set of 3 off the shelf components.

* Cypress FX3 CYUSB3 development board that has a Programmable Logic Circuit that enables fast communication with external FPGA.
* FPGA development board, we used the Avnet Zedboard that is relatively affordable. We choose this board for two simple reasons. First, we had one. Second, it has a FMC connector that makes the connection with the FX3 easier and more stable.
* And finally the FX3 to FMC connector.

**Note** The pinouts for the zedboard is avaliable ![here]()
**Note** We will release soon a JTAG multiplexer that enables you to connect Inception and another JTAG probes on the same device.
Using debugger commands, you would be able to select which debugger is active. 

## Using the Debugger

Once connected, you can test if the debugger is running correctly using the python binding script.
This script supports many features such as core halt, core resume, read/write system memory, dumping/loading binary.

![Using the python binding](./img/demo_debugger_lib.svg)


[1]: http://www.s3.eurecom.fr/tools/avatar/
