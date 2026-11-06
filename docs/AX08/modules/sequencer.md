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

These presets can be modified via the [clock upload command](#upload_clocks-0b1000_0000) of the the [control Interface](#control-interface).



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



## Control interface

The sequencer module exposes a UART control interface. An external device can use it to control execution, query the sequencer state, and upload custom clock timings.

### Packet format

Data is exchanged in packets. Each packet begins with a one-byte id. Commands sent to the sequencer and responses sent back use separate id mappings. Bit 7 of the id determines the packet format. If it is cleared, the packet consists of the id byte only, if it is set, the id is followed by a one-byte unsigned payload length, `n_payload_bytes`, and exactly that many payload bytes. If the external device takes more than ~32ms the sequencer aborts the packet and responds with an `ERROR_TIMEOUT` packet. The payload length counts only the payload bytes. A packet with bit 7 set and `n_payload_bytes=0` still has a total size of 2.



### Sequencer responses

Specific to response ids is, that bit 6 of the id marks an error. Except for the `SEQUENCER_STATE` response, all sequencer responses are id-only packets. They are:

Alias|Address|Value
:---:| :---: | ---
`ACKNOWLEDGE`|`0b00000001`|The sequencer processed the command successfully.
`ERROR_UNHANDLED`|`0b01000000`|An unhandled error occurred.
`ERROR_UNKNOWN_CMD`|`0b01000001`|Tje sequencer received an unknown command id.
`ERROR_TIMEOUT`|`0b01000010`|The sequencer timed out wile waiting for the rest of the packet.
`ERROR_INVALID_ARGS`|`0b01000011`|The command contained invalid arguments.
`SEQUENCER_STATE`|`0b10000xxx`|Returns the requested sequencer state. The low four bits match those of the `QUERY_STATE` command.



### ACTION_TOGGLE_HOLD (0b0000_0001)

Equivalent to pressing the `ACTION` button. When the computer is enabled, this command halts or resumes execution. When the computer is disabled, it resets the computer state.

The sequencer replies with `ACKNOWLEDGE` after processing the command.



### ACTION_RUN_INSTRUCTION (0b0000_0010)

Equivalent to pressing the `CYCLE` button. Runs one complete instruction or finishes the current instruction if it is already in progress, then halts. When the computer is disabled, the command has no effect.

The sequencer replies with `ACKNOWLEDGE` after processing the command.



### ACTION_RUN_STEP (0b0000_0011)

Equivalent to pressing the `STEP` button. Runs one sequencer step then halts. When the computer is disabled, the command has no effect.

The sequencer replies with `ACKNOWLEDGE` after processing the command.



### QUERY_STATE (0b0110_xxxx)

Queries the current sequencer state. The lower 4 bits of the command id select which information to return.

Bit|Flag|Description
:---:| :---: | ---
`0`|`state`|Whether the computer is enabled, the current sequencer state and the selected clock index.
`1`|`instruction`|The current step within the instruction.
`2`|`config`|The configured clock timings.
`3`|`version`|The version of the sequencer firmware.

The sequencer replies with `SEQUENCER_STATE` (0b1000_xxxx). The lower four bits are directly copied from the command id. The payload contains the requested segments in the order below. If no segment is requested, the response consists of the id byte and a zero length payload.

If the `state` flag is set, the response includes one byte with the following layout:

Bits|Name|Description
:---:| :---: | ---
`0`|`enabled`|`1` if the computer is enabled, otherwise `0`.
`1:3`|`state`|Index of the current sequencer state.
`4:7`|`selected_clock`|Index of the selected clock speed.

If the `instruction` flag is set, the response includes the current step within the instruction.

If the `config` flag is set, the response includes the following fields in order:

1. The current `T2CON` register, containing information about the hardware timers pre & post scaler.
2. The configured hardware timer period, one byte.
3. The number of configured software timer periods, from 1 to 16.
The configured software timer periods in clock index order, each encoded as an unsigned 16-bit little-endian integer.

If the `version` flag is set, the response includes the following fields in order:

1. The protocol version of the sequencer as a single 8-bit unsigned integer.
2. The length `n_version` of the version string as a single 8-bit unsigned integer.
3. The `n_version` bytes of the version string (excluding the `0` byte).
4. The length `n_commit` of the git description string as a single 8-bit unsigned integer.
5. The `n_commit` bytes of the git description string (excluding the `0` byte).


### SELECT_CLOCK (0b0111_xxxx)

Selects the clock speed whose index, `i_clock` is encoded as an unsigned integer in the lower 4 bits of the command id. Clock index `0` selects turbo mode (20kHz).

The sequencer replies with `ACKNOWLEDGE` after processing the command.



### UPLOAD_CLOCKS (0b1000_0000)

Configures between 1 and 16 clock speeds. The configured clocks can be selected with `SELECT_CLOCK` or cycled through with the `DELAY` button. The configuration is stored in EEPROM and therefore persists across restarts.

Clock index `0` is reserved for the turbo mode. When clock `0` is selected, continuous execution runs at 20kHz regardless of the uploaded timings. The uploaded timings for clock 0 only applies when running a single instruction or a single sequencer step from the halted state.

The payload contains the following fields in order:

1. One byte containing the timer prescaler and postscaler settings, using the `T2CON` register layout.
2. One byte containing the hardware timer period.
3. One unsigned 16-bit little-endian software timer period for each configured clock, starting at index 0.

The payload length is `2 + 2*N` bytes for `N` configured clocks, where N is between 1 and 16. If this is not the case, the handler aborts and responds with `ERROR_INVALID_ARGS`.

The sequencer replies with `ACKNOWLEDGE` after processing the command.
