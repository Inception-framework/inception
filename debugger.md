# inception debugger

The Inception debugger is a powerful device that answer to 2 main needs:

* Low latency during read/write operations from host to device.
* Low latency interrupt forwarding.

We acknowledge that the debugger has not a wide scope of supported targets such as commercial JTAG probes.
However, it is designed for performance that is a requirement when performing firmware rehosting while keeping real hardware interactions.

![Using the python binding](./img/demo_debugger_lib.svg)
