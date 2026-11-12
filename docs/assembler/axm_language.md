# AXM Language

Axiom ASM (AXM) is an assembly like language designed for the axiom computing platform. The language is a mix of classical assembly languages, with a a few features from higher-level languages. Additionally a few features like _scopes_ are added, which are intended to simplify programming in the language.

## Language design

A axiom assembly program consists of multiple "statements". Statements are text separated by newlines or semicolons. Note that if a statement includes a comment ('#' symbol), the remaining line is skipped. The following block shows a program with three statements.

```axm
R1 = 3
R2 = R1; R4 = 5 # This is a comment
```

### Statements

The statements can be categorized into two groups:

- **Instruction statements** These are statements that correspond 1 to 1 to actual machine code. They are instructions like `jump 5`, `R1 = 3`,  or `R2 = R1 + 5 if R2 > 0`. A table containing all available instruction statements can be found below.

- **Assembler statements** These define meta data to be used by the assembler to produce the final program binary. This includes but is not limited to: Variable definitions, Label definitions, Scope statements, ...

### Condition

A condition always has the form `<register> <comparison operator> 0`, where `<register>` is placeholder for any register and `<comparison operator>` is one of the following operators. `<value>` always refers to the current value of the given `<register>`:

Operator|Description
---|---
`==`|Checks if `<value>` is equal to 0.
`!=`|Checks if `<value>` is not equal 0.
`<`|Checks if `<value>` is less than 0, which is indicated by the MSb of the binary representation being set.
`>`|Checks if `<value>` is greater than 0.
`<=`|Checks if `<value>` is less than or equal to 0.
`>=`|Checks if `<value>` is greater than or equal to 0.

Note that some architectures have restrictions on which registers can be used for conditions.

### Scopes

A scope is a collection of statements that can have additional context state like variables that are only defined inside the scope. Additionally scopes come with the `if`, `repeat` and `exit` statements which allow implementing common loop and condition patterns.

#### Definition

A scope is defined by enclosing a collection of statements in curly brackets. A simple scope could look like this

```
{ # Begin of a scope
    R1 = 0
} # End of the scope
```

Scopes can also be named by prefixing their opening bracket with an '@' followed by the name of the scope. This can look like this:

```
@my_scope {
    R2 = 1 + 24
}
```

#### Scoped jumps

Whilst the above examples are valid cases of scopes, they do not yet use any of the special functionality scopes provide and behave identically to the corresponding programs without scopes.

One of the two main benefits benefits that scopes provide are _scoped jumps_, which are jumps that implicitly use the start / end of a scope as jump target. There are two kinds of scoped jumps. They are:

- **Repeat statements**, which jump to the beginning of the scope, to the first instruction that is defined inside the statement.
- **Exit statements**, which jump to the ending of the scope, to the first instruction that is defined after the scope has ended.

Both of them can be suffixed with a _condition_, which will be described in the instructions chapter. The condition is used exactly for the generated jump instruction.

For example the following code snippet counts down from 5 to 1 and prints every step.
```
R1 = 5                 # Initialize register 1 to 5.
{
    write R1           # Write the current value of register R1 to the output.
    dec R1             # Decrement the register R1 by 1.
    repeat if R1 != 0  # Repeat the scope if R1 is not equal 0.
}
write 0                # Write a 0 to the output.
```
It does that by initializing the register R1 to 5 and then, inside of the scope, decrementing R1 by 1 after each print. At the end of the scope is a conditional repeat statement that only jumps to the beginning of the scope if the register R1 is 0. If the condition is not met, the scope is exited and the next statement after the scope is executed, which in this case outputs a 0.

The same behavior can be achieved similar with an exit statement the following way:
```
R1 = 5               # Initialize register 1 to 5.
{
    write R1         # Write the current value of register R1 to the output.
    dec R1           # Decrement the register R1 by 1.
    exit if R1 == 0  # Exit the scope if R1 is 0.
    repeat           # Repeat the scope.
}
write 0              # Write a 0 to the output.
```

Here the repeat statement is unconditional and instead we have an additional conditional exit statement that jumps to the end of the scope.

Scopes can be nested. Scoped jump statements always refer to the inner most scope. If one whishes to repeat or exit an outer scope, one has to give that outer scope a name and specify that name in the repeat / exit statement.

```
@outer {
    {
        exit @outer # Jumps to the first instruction after the outer scope.
    }
    write 1
}
write 0
```
The above code only prints a 0 because of the exit jump.

#### Scoped name spaces

The second main benefit of scopes is that they provide a way to manage the namespace of variables and labels. For this we must first look at what variables and labels are.

### Variables

A variable is a placeholder for a value. It is defined via a variable definition statement, `$my_var := <some_value>`, where `<some_value>` can be replaced with anything "value-like", that are immediate values, register names, labels, or even other variables as long as no cyclic dependency chains are created. Variable are always prefixed with a '$' symbol.

During compilation the assembler evaluates the value of the variable, potentially going through multiple other variables, and replaces all references to the variable with its value.

The name of a variable has to be unique in its definition scope. Variables are only visible in their own scope and all their child scopes. If a variable name is already defined in a scope, it can be redefined in any of its child scopes. In the child scope the inner variable is then used. This is demonstrated in the following examples.

```
{
    $my_var := 3       # Defines a new variable
    {
        $my_var := 4   # Defines a new variable
        write $my_var  # Writes a 4.
    }
    {
        $my_var := 5   # Defines a new variable
        write $my_var  # Writes a 5.
    }
    write $my_var      # Writes a 3.

    $my_var := 3       # ERROR, "my_var" is already defined in this scope.
}
```

### Labels

Labels are similar to variables in that they are a placeholder for a value, with the key difference, that instead of an arbitrary value, they store the index of the next instruction following its definition. The are designed for specifying custom jump targets. Labels are always prefixed with an '@' symbol.

The name of a scope is subject to similar rules as those of variables in that the name must be unique in the definition scope. However, in contrast to variables, labels are also visible to their parent scopes and can't be redefined in sub-scopes.

A simple program that uses labels could look like this:

```
{
    R1 = read                       # Read value into register R1.

    jump @error_handler if R1 == 0  # Jump to the "error_handler" label if the input is negative.
    write R1                        # Output the value of register R1.
    repeat                          # Repeat the current scope.

    @error_handler
    write 0                         # Output a 0.
    repeat                          # Repeat the current scope.
}
```

### If condition statements

A if condition statement is another flow control statement. It allows to execute a following statement only if a given condition is met. The following statement can also be a scope in which case the whole scope is skipped if the condition is not met.

```
if R1 != 0
    write R1
write 0
```
In this case only the `write R1` statement is executed conditionally. The `write 0` statement is always executed.`Note that whitespace is ignored by the assembler.

```
if R1 != 0
{
    write R1
    write 0
}
```
Here both statements are skipped if the condition is not met.

## Statements

### Instruction statements

As previously mentioned, instruction statements always map 1 to 1 to a machine code instruction. The syntax of an instruction statement is the syntax of the corresponding operation, which is listed in the following table and an optional condition. An instruction with a condition has the form `<operation> if <register> <condition> 0`, where `<condition>` is any condition. The instruction is then only executed if the condition is met.

#### Load instructions

A load instructions loads a value to a register. That value can be anything "value-like" that resolves to either an immediate value or another register.

 There is also a variant that combines two values and loads that combined value into the target register. This is used for example when setting the value of a 16bit register on an 8bit architecture.

Instruction|Description
---|---
`<register> = <value>`|Loads the given `<value>` into the given `<register>`
`<register> = <value_1>, <value_0>`|Loads the combined value into the given `<register>`

#### Logic instructions

These statements perform bitwise logical calculations. They have the form `<output> = <a> <operator> <b>`, where `<output>` is a register, `<a>` and `<b>` are value-like and `<operator>` is one of the following operators:

Instruction|Description
---|---
`<output> = <a> and <b>`|Calculates bitwise a and b
`<output> = <a> nand <b>`|Calculates bitwise a nand b
`<output> = <a> or <b>`|Calculates bitwise a or b
`<output> = <a> nor <b>`|Calculates bitwise a nor b
`<output> = <a> xor <b>`|Calculates bitwise a xor b
`<output> = <a> xnor <b>`|Calculates bitwise a xnor b

#### Arithmetic instructions

These statements perform arithmetic calculations. They have the form `<output> = <a> <operator> <b>`, where `<output>` is a register, `<a>` and `<b>` are value-like and `<operator>` is one of the following operators:

Instruction|Description
---|---
`<output> = <a> + <b>`|Calculates a + b
`<output> = <a> - <b>`|Calculates a - b

These instructions modify the state of the _carry flag_. The value of the carry flag can also be used as an input by suffixing the instruction with `with carry`, in which case the last carry out value is used as carry in.

#### Shift instructions

These statements perform shift operations. They have the form `<output> = <operator> <value>`, where `<output>` is a register, `<value>` is value-like and `<operator>` is one of the following operators:

Instruction|Description
---|---
`<output> = shift left <value>`|Shifts `<value>` one to the left. the lsb is set to 0.
`<output> = shift right <value>`|Shifts `<value>` one to the right. the msb is set to 0.
`<output> = rotate left <value>`|Shifts `<value>` one to the left. the lsb is set to carry in.
`<output> = rotate right <value>`|Shifts `<value>` one to the right. the msb is set to carry in.

These operations also modify the state of the _carry flag_. It is set to the value of the bit, that is shifted "out". The difference between the _shift_ and the _rotate_ instructions is, that the shift instruction sets the emptied bit to 0, whilst the rotate instructions fill it with the current state of the carry flag.

#### Decode instructions

The decode instructions generate a bitmask where only the selected bit is set / cleared.

Instruction | Description
:---: | ---
`<output> = decode <bit_index>` | Generates a bitmask where only the selected bit is set.
`<output> = ndecode <bit_index>` | Generates a bitmask where all bits except the selected bit are set.
