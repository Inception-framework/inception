# inception debugger

The Inception debugger is a powerful device that answers to 2 main needs:

* Low latency during read/write operations from host to device. When rehosting firmware execution while keeping hardware interactions, one main challenge it is to avoid that this communication become the bottleneck. We show this limitation with Avatar \[1].
* Low latency interrupt forwarding. Firmware programs are generaly interrupt-driven which means that the firmware execution can be interrupted by hardware peripherals to execute certain routines. Inception is the first work that correctly forward the interrupt signal between device and rehosted platform while keeping the device state and the rehosted platform synchornized.

**Note** We acknowledge that the debugger has not a wide scope of supported targets such as commercial JTAG probes. Therefore, we are working on a new version that enables users to easily customize the debugger's FPGA behavior without modfying its circuitry.

## System Description

The debugger is not a single piece but a set of 3 off the shelf components.

* Cypress FX3 CYUSB3 development board that has a Programmable Logic Circuit that enables fast communication with external FPGA.
* FPGA development board, we used the Zedboard Zynq-7000 Development Board that is relatively affordable. We choose this board for two simple reasons. First, we had one. Second, it has a FMC connector that makes the connection with the FX3 easier and more stable.
* And finally the FX3 to FMC connector.

**Note** The pinouts for the zedboard is avaliable  **Note** We will release soon a JTAG multiplexer that enables you to connect Inception and another JTAG probes on the same device. Using debugger commands, you would be able to select which debugger is active.

## Using the Debugger

Once connected, you can test if the debugger is running correctly using the python binding script. This script supports many features such as core halt, core resume, read/write system memory, dumping/loading binary.

![Using the python binding](.gitbook/assets/demo\_debugger\_lib.svg)

## Supported Board

The current version has been tested on Cortex-M3 micro-controller. However, it should work with ARM processors implemented the Coresight design, that is all recent ARM CPU (ARMv7/M/ARMv8/M). Unfortunatly, supporting new Coresight compliant chip requires sometimes design modification as the boundary scan may contains different device with different IR register length. The second version of the debugger fixes this issue.

| Device        | Supported |
| ------------- | --------- |
| LPC1850-DB1   | Yes       |
| STM32-L152RE  | Yes       |
| Zedboard ARM9 | Not yet   |

## Testing Debugger From the Inception Analyzer

Users can find interesting samples and test on the github repository. Among these samples, some programs are provided to verify the debugger. [For instance, the usenix-samples/forwarding\_tests/speed\_io\_8b\_and\_16b/](https://github.com/Inception-framework/usenix-samples) test that the forwarding mechanism is working well by writing a random byte in the device memory and then reading the same address to compare. This is repeated 65536 times and takes less than 8s in average to return.

```
cd
git clone https://github.com/Inception-framework/usenix-samples.git
cd usenix-samples/forwarding_tests/speed_io_8b_and_16b/
make

inception --allocate-determ --allocate-determ-start-address 0x30000000 --allocate-determ-size 10000 --elf ./bin.elf --bitcode ./bin.bc --mem_conf_file ./mem.json --has_debugger
```

To configure the forwarding mechanism, Inception requires a memory model that is described in files mem.conf

```
{
    "memory_model" : [
      { "name": "MMIO", "base" : "2000C000", "size": "FFFF", "read_only": false, "strategy": "forwarded"}
    ]
}
```

![Using the python binding](.gitbook/assets/benchmark\_io.svg)

## References

\[1]: [http://www.s3.eurecom.fr/tools/avatar/](http://www.s3.eurecom.fr/tools/avatar/)
