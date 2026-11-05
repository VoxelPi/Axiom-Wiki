# Sequencer

The sequencer is the heartbeat of the computer. It drives instruction execution by setting and clearing six control signals in sequence. It also initializes the computer at startup and responds to the `BREAK` operation, halting execution when executed.

The sequencer is a external module plugs into a socket on the computer. The standard sequencer is implemented using a microcontroller running custom firmware.



## Front panel inputs

The sequencer is connected to four buttons on the front panel that allow the user to control program execution. They are:

- **ACTION**: If the computer is turned halts / resumes execution. Otherwise it resets the computer state.
- **CYCLE**: Runs a single full instruction.
- **STEP**: Runs a single sub-instruction step.
- **DELAY**: Cycles between the different clock speeds.



## Available clock speeds

The standard sequencer firmware comes with four preset different clock speeds:

- 20000 Hz (Turbo mode)
- 1000 Hz
- 100 Hz
- 1 Hz

Adding clock speeds requires changing the sequencer firmware configuration and reflashing the sequencer.



## Sequence pins

The sequencer controls six state pins. They are:

- **FREEZE_WORD**: On a rising flag the current values of operand A, operand B, the output register address as well as the condition evaluation result are latched.

- **FREEZE_OPCODE**: On a rising flag the opcode is latched.

- **FREEZE_RESULT**: On a rising flag the currently value of the output bus is latched in the output buffer.

- **PC_SOURCE_INC**: When high, selects `PC + 1` to be written to the program counter.

- **AD_RGSET_OVERRIDE**: On a rising edge the selected value is written to the program counter.

- **STORE**: On a rising flag the current value of the output buffer is written to the configured output register. Note that internally the hardware forwards the pulse only if the instruction's condition evaluated to `true` and the operation is none of the following four: `MEMORY.STORE`, `IO0.WRITE`, `IO1.WRITE`, `BREAK`.



## Instruction execution steps

During the execution of an instruction the computer goes through the following states:

- **FETCH**: Starts by pulsing `FREEZE_WORD` to latch the operand values `A` and `B`, the output register, and the condition result.

- **EXECUTE**: Starts by pulsing `FREEZE_OPCODE` to latch the opcode. The selected operation. From this point on the selected operation uses the latched inputs and writes its result onto the output bus.

- **CAPTURE OUTPUT**: Pulses `FREEZE_RESULT` to capture the current output bus value, then sets the `PC_SOURCE_INC` flag to select `PC + 1`.

- **INCREMENT PC**: Pulses `AD_RGSET_OVERRIDE` to write `PC + 1` to the program counter. It then clears `PC_SOURCE_INC` and waits briefly for the signal to settle.

- **STORE**: Pulses store. If the hardware forwards the pulse, the value currently held in the output buffer is written to the selected output register. The sequencer then waits briefly for the signals to settle.



## Computer Initialization

To initialize the computer the sequencer pulses `AD_RGSET_OVERRIDE`, which causes the program counter to be reset.



## Bridge connection

The sequencer module has an UART interface for connecting to the bride module. This connection is intended to support more advanced debugging. The latest firmware does not yet support this feature.
