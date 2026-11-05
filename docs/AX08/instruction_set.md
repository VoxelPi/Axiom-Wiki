# Instruction Set

## Instruction

Every instruction is encoded in a 32-bit word with the following layout:

![instruction_word](./assets/instruction_word.png)


### Operation

This field specifies the operation to perform. Each operation receives two 8-bit input values, `A` and `B`, and produces an 8-bit result that is stored in the output reigster. Depending on the operation one or both inputs may be unused.

See the [Operations](#operations) section for a list of supported operations.


### Condition

Each instruction defines a condition for its execution using two fields:
- `CONDITION TEST` specifies the check to perform
- `CONDITION REGISTER` specifies which of the eight general-purpose registers supplies the value `C` to check.

The condition is evaluated immediately before the instruction executes, using the current value `C` of the selected register and producing either `true` or `false`. This happens before the instruction modifies any computer state, so any changes it makes to the condition register do not affect its own condition evaulation.

- If the condition evaluates to `true`, the instruction is executed.
- If the condition evaluates to `false`, the instruction is skipped. The program counter advances to the next instruction, but all other computer state remains unchanged.

There are 8 possible condition tests.
For any comparisons with 0, `C` is interpreted as a signed 8-bit two's complement integer ranging from -128 to 127.

Condition | Value | Evaluates to true when
:---:| :---: | ---
ALWAYS | `0b000` | Always no matter the input.
NEVER | `0b001` | Never no matter the input.
= 0 | `0b010` | `C = 0`
≠ 0 | `0b011` | `C ≠ 0`
< 0 | `0b100` | `C < 0`
≥ 0 | `0b101` | `C ≥ 0`
\> 0 | `0b110` | `C > 0`
≤ 0 | `0b111` | `C ≤ 0`


### Output Register

This part of the instruction specifies bits 0:2 of the address of the register in which the operations result is stored.
Bit 3 of the register address is determined by the operation: It is set to 1 for load_2 and cleared to 0 for all other operations.

There that the two operations `MEMORY.STORE` and `IO.WRITE` do not produce an output and therefore do no write to the output register.


### Operands A & B

Each instruction has two operands, A and B, which provide the inputs to the operation. Each operand consists of a 8-bit field, `OPERAND` and a mode bit, `MODE`, which determines how that field is interpreted:

- **Immediate mode (MODE = 0)**: The 8-bit field is used directly as the input value.

- **Register mode (MODE = 1)**: The lower 4 bits specify the address of the register whose current value should be used as the input. The upper 4 bits are unused and should be cleared to 0.

![Operand](./assets/operand.png)



## Operations

### Load

Load operations write their input value directly to the output.

Operation | Value | Description
:---:| :---: | ---
LOAD2 | `0b00000` | Creates a 16-bit word using the value of `A` for the lower, and the value of `B` for the upper 8 bits. The value is then stored in the special function register, specified by the output address.
LOAD | `0b00001` | Output the value of `A`.

### Logic

The logic operations produce their result by performing a bitwise logical operation on `A` and `B`.

Operation | Value | Description
:---:| :---: | ---
AND | `0b00010` | Bitwise and.
NAND | `0b00011` | Bitwise nand.
OR  | `0b00100` | Bitwise or.
NOR | `0b00101` | Bitwise nor.
XOR | `0b00110` | Bitwise xor.
XNOR | `0b00111` | Bitwise xnor.

### Arithmetic

The arithmetic operations produce their result by performing an arithmetic operation on `A` and `B`. Only the lower eight bits of the calulcated value are retained.

Each operation also updates the carry bit. For addition, it is set if the calcualted value exceeds `256`, otherwise it is cleared. For subtraction, it is set if no unsigned borrow is required, otherwise it is clared.

In the table below, `carry` refers to the previous value of the carry bit.

Operation | Value | Calculation
:---:| :---: | ---
ADD | `0b01000` | `A` + `B`.
SUBTRACT | `0b01001` | `A` - `B`.
ADD WITH CARRY | `0b01010` | `A` + `B` + `carry`.
SUBTRACT WITH CARRY | `0b01011` | `A` - `B` - (1 - `carry`).

### Shift

The shift operations produce their result by shifting `A` left or right by one bit. The input `B` is unused.
The bit shifted out (bit 7 for a left shift and bit 0 for a right shift) is stored in the carry bit.

Operation | Value | Description
:---:| :---: | ---
SHIFT LEFT | `0b01100` | Shifts `A` left by one bit and sets bit 0 to `0`. Equivalent to multiplying the unsigned value of A by 2, keeping only the lower eight bits.
SHIFT RIGHT | `0b01101` | Shifts `A` right by one bit and sets bit 7 to `0`. Eqivalent to dividing the unsigned value of A by 2, rounding down.
ROTATE LEFT | `0b01110` | Shifts `A` left by one bit and sets bit 0 to the previous value of the carry bit.
ROTATE RIGHT | `0b01111` | Shifts `A` right by one bit and sets bit 7 to the previous value of the carry bit.


### Decode

The decode operations interprete bits 0-2 of `A` as an unsigned 3-bit integer that selects a bit position from 0 (least significant) to 7 (most significant). They produce an 8-bit mask with only the selected bit set or cleared. Bits 3-7 of `A` and the whole input `B` are unused.

Operation | Value | Description
:---:| :---: | ---
NDECODE | `0b10000` | Produces a mask with all bits set except for the selected bit, which is cleared.
DECODE | `0b10001` | Produces a mask with all bits cleared except the selected bit, which is set.


### Memory

The memory operations read from or write to the computers memory. They construct a 16-bit memory address by using `A` as the lower 8 bits and the current value of the `PAGE` register for the upper 8 bits.

Operation | Value | Description
:---:| :---: | ---
LOAD | `0b10100` | Reads the 8-bit value at the specified memory address and outputs it as the result. Input `B` is unused.
STORE | `0b10101` | Writes the value of `B` to the specified memory address. No value is written to the output register.


### IO

There are two IO units, each supporint the following three operations. Bit 2 of the opcode selects the unit: `0` selects IO unit 0, `1` selects IO unit 1. This bit is marked as `x` in the table.

Operation | Value | Description
:---:| :---: | ---
POLL | `0b11x00` | Outputs a `1` if a value is available to be read, otherwise outputs `0`.
READ | `0b11x01` | Reads a value from the io port and outputs it. If no value is available the returned value is undefined.
WRITE | `0b11x10` | Writes the value of `A` to the io port. No value is written to the output reigster.


### Break

The break instruction causes the computer to halt at the end of the instruction. Combined with an instruction condition, it can be used as a conditional software breakpoint.

Operation | Value | Description
:---:| :---: | ---
BREAK | `0b11111` | Halts the computer at the end of the instruction.
