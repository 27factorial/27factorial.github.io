
The PF virtual machine comes with its own assembly language and assembler to make reading and writing virtual machine code directly easier.

# Identifiers




# Data types

As mentioned in the [[Types]] document, the PF virtual machine has 7 basic types. Each one has a specific mnemonic in PFVM assembly.

| Rust `Value` variant | ASM mnemonic |
| -------------------- | ------------ |
| Int                  | int          |
| Float                | float        |
| Bool                 | bool         |
| Char                 | char         |
| Symbol               | sym          |
| Function             | func         |
| Reference            | ref          |

Values in assembly must be prefixed by their type and a space (e.g. `int 64`). Integer-like types (Int, Symbol, Function, and Reference) can all be expressed in either decimal, hexadecimal, or binary. Decimal has no prefix, hexadecimal requires the `0x` prefix, and binary requires the `0b` prefix. Hexadecimal values can also be either uppercase, lowercase, or a mix of both, although picking one or the other and staying consistent should be preferred.

# Instructions

Instructions in PFVM assembly generally take the form `mnemonic operand1 operand2 ...`. Arithmetic instructions should have their operands read from left to right. Arithmetic operations which only take one operand always have the value at the top of the stack (`top`) as their first parameter.

For a few examples:
- Subtract 2 from `top` => `subi int 2`
- Divide `top` by 12.25 => `divi float 12.25`

Instructions can also be expressed using their machine code representation as an integer (which follows the same rules as integer-like types above). For example, the `halt` instruction can also be written as simply `1`, `0x1`, or `0b1`.

### Operands

Operands labeled `any` are simply values of any type.

### List

The current instruction listing is as follows:

| Assembly mnemonic | Operands         | Stack layout                                           | Rust `OpCode` variant | Description                                                                                                                                                      |
| ----------------- | ---------------- | ------------------------------------------------------ | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nop`             |                  |                                                        | `NoOp`                | No operation; do nothing                                                                                                                                         |
| `halt`            |                  |                                                        | `Halt`                | Immediately halt virtual machine execution                                                                                                                       |
| `push`            | `any`            | -> `any`                                               | `Push`                | Push an immediate value onto the top of the stack.                                                                                                               |
| `pushc`           | `int`            | -> `any`                                               | `PushConst`           | Push the constant at a given index to the top of the stack.                                                                                                      |
| `pop`             |                  | `any` ->                                               | `Pop`                 | Pop a value from the stack, discarding it.                                                                                                                       |
| `load`            | `int`            | `int` ->                                               | `Load`                | Load a local variable onto the top of the stack.                                                                                                                 |
| `stor`            | `int`            | `int` ->                                               | `Store`               | Store the value at the top of the stack into a local variable.                                                                                                   |
| `dup`             |                  | `any` -> `any`, `any`                                  | `Dup`                 | Duplicates the value at the top of the stack, pushing a copy of it to the top of the stack.                                                                      |
| `rsrv`            |                  | `int` ->                                               | `Reserve`             | Pop an integer from the stack and reserve that number of stack values as local variables for the current function, starting from the current stack base address. |
| `rsrvi`           | `int`            |                                                        | `ReserveImm`          | Reserve a given number of stack values as local variables for the current function, starting from the current stack base address.                                |
| `add`             |                  | `int` or `float`, `int` or `float` -> `int` or `float` | `Add`                 | Pop the top two values from the stack and add them, pushing the result onto the top of the stack.                                                                |
| `addi`            | `int` or `float` | `int` or `float` -> `int` or `float`                   | `AddImm`              | Pop a value from the top of the stack, and add the given value to it, pushing the result onto the top of the stack.                                              |
| `sub`             |                  | `int` or `float`, `int` or `float` -> `int` or `float` | `Sub`                 | Pop the top two values from the stack and subtract them, pushing the result onto the top of the stack.                                                           |
| `subi`            | `int` or `float` | `int` or `float` -> `int` or `float`                   | `SubImm`              | Pop a value from the top of the stack, and sub the given value from it, pushing the result onto the top of the stack.                                            |
| `mul`             |                  | `int` or `float`, `int` or `float` -> `int` or `float` | `Mul`                 | Pop the top two values from the stack and multiply them, pushing the result onto the top of the stack.                                                           |
| `muli`            | `int` or `float` | `int` or `float` -> `int` or `float`                   | `MulImm`              | Pop a value from the top of the stack, and multiply the given value by it, pushing the result onto the top of the stack.                                         |
| `div`             |                  | `int` or `float`, `int` or `float` -> `int` or `float` | `Div`                 | Pop the top two values from the stack and divide them, pushing the result onto the top of the stack.                                                             |
| `divi`            | `int` or `float` | `int` or `float` -> `int` or `float`                   | `DivImm`              | Pop a value from the top of the stack, and divide it by the given value, pushing the result onto the top of the stack.                                           |
| `rem`             |                  | `int` or `float`, `int` or `float` -> `int` or `float` | `Rem`                 | Pop the top two values from the stack and take the remainder of the first value from the second, pushing the result onto the top of the stack.                   |
| `remi`            | `int` or `float` | `int` or `float` -> `int` or `float`                   | `RemImm`              | Pop a value from the top of the stack, and take the remainder of the first value from the given value, pushing the result onto the top of the stack.             |
| `and`             |                  | `int` or `bool`, `int` or `bool` -> `int` or `bool`    | `And`                 | Pop two values from the stack, and compute the bit-wise AND operation between them, pushing the result to the top of the stack.                                  |
| `andi`            | `int` or `bool`  | `int` or `bool` -> `int` or `bool`                     | `AndImm`              | Pop a value from the stack, and compute the bit-wise AND operation between the value and the immediate, pushing the result to the top of stack.                  |
| `or`              |                  | `int` or `bool`, `int` or `bool` -> `int` or `bool`    | `Or`                  | Pop two values from the stack, and compute the bit-wise OR operation between them, pushing the result to the top of the stack.                                   |
| `ori`             | `int` or `bool`  | `int` or `bool` -> `int` or `bool`                     | `OrImm`               | Pop a value from the stack, and compute the bit-wise OR operation between the value and the immediate, pushing the result to the top of stack.                   |
| `xor`             |                  | `int` or `bool`, `int` or `bool` -> `int` or `bool`    | `Xor`                 | Pop two values from the stack, and compute the bit-wise XOR operation between them, pushing the result to the top of the stack.                                  |
| `xori`            | `int` or `bool`  | `int` or `bool` -> `int` or `bool`                     | `XorImm`              | Pop a value from the stack, and compute the bit-wise XOR operation between the value and the immediate, pushing the result to the top of stack.                  |
| `not`             |                  | `int` or `bool`, `int` or `bool` -> `int` or `bool`    | `Not`                 | Pop a value from the top of the stack, and compute the bit-wise NOT operation of it, pushing the result onto the top of the stack.                               |
| `shr`             |                  | `int`, `int` -> `int`                                  | `Shr`                 | Pop two values from the stack, and right-shift the first value by the second value, pushing the result onto the top of the stack.                                |
| `shri`            | `int`            | `int` -> `int`                                         | `ShrImm`              | Pop a value from the stack, and right-shift the value by the immediate value, pushing the result onto the top of the stack.                                      |
| `shl`             |                  | `int`, `int` -> `int`                                  | `Shl`                 | Pop two values from the stack, and left-shift the first value by the second value, pushing the result onto the top of the stack.                                 |
| `shli`            | `int`            | `int` -> `int`                                         | `ShlImm`              | Pop a value from the stack, and left-shift the value by the immediate value, pushing the result onto the top of the stack.                                       |
| `eq`              |                  | `any`, `any` -> `bool`                                 | `Eq`                  | Pop two values from the stack, and push a boolean onto the top of the stack indicating whether or not the two values are equal.                                  |
| `eqi`             | `any`            | `any` -> `bool`                                        | `EqImm`               | Pop a value from the stack, and push a boolean onto the top of the stack indicating whether the value is equal to the given immediate value.                     |
| `gt`              |                  | `any`, `any` -> `bool`                                 | `Gt`                  | Pop two values from the stack, and push a boolean onto the stack indicating whether or not the first value is greater than the second value.                     |
| `gti`             | `any`            | `any` -> `bool`                                        | `GtImm`               | Pop a value from the stack, and push a boolean onto the stack indicating whether the value is greater than the given immediate value.                            |
| `ge`              |                  | `any`, `any` -> `bool`                                 | `Ge`                  | Pop two values from the stack, and push a boolean onto the stack indicating whether or not the first value is greater than or equal to the second value.         |
| `gei`             | `any`            | `any` -> `bool`                                        | `GeImm`               | Pop a value from the stack, and push a boolean onto the stack indicating whether the value is greater than or equal to the given immediate value.                |
| `lt`              |                  | `any`, `any` -> `bool`                                 | `Lt`                  | Pop two values from the stack, and push a boolean onto the stack indicating whether or not the first value is less than the second value.                        |
| `lti`             | `any`            | `any` -> `bool`                                        | `LtImm`               | Pop a value from the stack, and push a boolean onto the stack indicating whether the value is less than the given immediate value.                               |
| `le`              |                  | `any`, `any` -> `bool`                                 | `Le`                  | Pop two values from the stack, and push a boolean onto the stack indicating whether or not the first value is less than or equal to the second value.            |
| `lei`             | `any`            | `any` -> `bool`                                        | `LeImm`               | Pop a value from the stack, and push a boolean onto the stack indicating whether the value is less than or equal to the given immediate value.                   |
| `jmp`             |                  | `int` ->                                               | `Jump`                | Pop a value from the stack, and jump forward or backward by the given number of instructions                                                                     |
| `jmpi`            | `int`            |                                                        | `JumpImm`             | jump forward or backward by the given number of instructions                                                                                                     |
| `jmpc`            |                  | `bool`, `int` ->                                       | `JumpCond`            | Pop two values from the stack, and jump forward or backward by the given number of instructions if the first value is true.                                      |
| `jmpci`           | `int`            | `bool` ->                                              | `JumpCondImm`         | Pop a value from the stack, and jump forward or backward by the given number of instructions if the given value is true.                                         |
| `rslv`            |                  | `sym` -> `func`                                        | `Resolve`             | Pop a symbol from the top of the stack, and push the function corresponding to the symbol onto the top of the stack.                                             |
| `rslvi`           | `sym`            | -> `func`                                              | `ResolveImm`          | Push the function corresponding to the given symbol onto the top of the stack.                                                                                   |
| `call`            |                  | `func` ->                                              | `Call`                | Call the function at the top of the stack.                                                                                                                       |
| `calli`           | `func`           |                                                        | `CallImm`             | Call the given function.                                                                                                                                         |
| `callb`           | `int`            | -> `...`*                                              | `CallBuiltin`         | Call the given builtin function.                                                                                                                                 |
| `calln`           | `sym`            | -> `...`*                                              | `CallNative`          | Call the native function corresponding to the given symbol.                                                                                                      |
| `ret`             |                  |                                                        | `Ret`                 | Return from the current function.                                                                                                                                |
| `casti`           |                  | `int` or `float` or `bool` or `char` -> `int`          | `CastInt`             | Pop a value from the top of the stack, and push the value cast to an integer onto the top of the stack.                                                          |
| `castf`           |                  | `int` or `float` or `bool` or `char` -> `float`        | `CastFloat`           | Pop a value from the top of the stack, and push the value cast to a float onto the top of the stack.                                                             |
| `castb`           |                  | `int` or `float` or `bool` or `char` -> `bool`         | `CastBool`            | Pop a value from the top of the stack, and push the value cast to a boolean onto the top of the stack.                                                           |
| `dbg`             | `int`            |                                                        | `Dbg`                 | Print the given value in the stack to stderr.                                                                                                                    |
| `dbgvm`           |                  |                                                        | `DbgVm`               | Print the debug representation of the VM to stderr.                                                                                                              |



