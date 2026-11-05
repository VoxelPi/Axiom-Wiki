# I/O Units

The AX08 can communicate with up to two external devices through its two I/O units. Each unit implements the `IO.POLL`, `IO.READ` and `IO.WRITE` operations.

Each I/O unit is an external module that plugs into a socket on the computer. The standard I/O unit is implemented using a microcontroller running custom firmware and exchanges data with external devices over UART. Both I/O ports are mounted on the front panel.

Custom I/O units can use other protocols to communicate with external devices provided they implemented all three AX08 I/O operations.



## RX & TX buffers

Each standard I/O unit 512-byte receive buffer and a 256-byte transmit buffer. To change either size, update the I/O units firmware configuration and reflash the module.

The receive buffer holds incoming bytes until the program reads them. This lets the AX08 receive bursts of data even at lower clock speeds reliably. However, if incoming data fills the buffer before the program reads enough bytes to free space, additional bytes may be lost.
