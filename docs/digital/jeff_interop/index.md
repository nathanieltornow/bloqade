# Interoperability with jeff

[jeff](https://github.com/unitaryfoundation/jeff) is an exchange format for quantum programs.
A jeff file holds functions with qubits, gates, classical values, loops, branches and calls.
A program therefore keeps its structure when it moves from one compiler to another.

Bloqade reads and writes jeff for [squin](../dialects_and_kernels/squin.md) kernels.
You can write a squin kernel as a `.jeff` file, and you can load a `.jeff` file from another tool as a squin kernel.

For the API reference, please see the `jeff` submodule in the API reference [here](../../reference/bloqade-circuit/src/bloqade/jeff/).

## Installation

Reading and writing jeff files needs the [`jeff-format`](https://pypi.org/project/jeff-format/) package.
It comes with the `jeff` extra of bloqade-circuit:

```
pip install bloqade-circuit[jeff]
```

## Example

Here's a program that is made of three kernels.
`plus_state` puts a qubit in the plus state, and `rotate` applies a rotation to a qubit.
`main` allocates a register of qubits and calls both kernels on each qubit in a loop.

```python
import math

from bloqade import squin
from bloqade.types import Qubit


@squin.kernel
def plus_state(q: Qubit):
    squin.h(q)


@squin.kernel
def rotate(q: Qubit, theta: float):
    squin.rz(theta, q)


@squin.kernel
def main(theta: float):
    q = squin.qalloc(4)
    for i in range(len(q)):
        plus_state(q[i])
        rotate(q[i], theta)
        squin.h(q[i])
    return squin.broadcast.measure(q)
```

### Writing a jeff file

`SquinToJeff().emit` converts the kernel to a jeff program, and `emit_jeff` returns a `jeff.JeffModule` that you can write to a file.

```python
from bloqade.jeff import SquinToJeff, emit_jeff

module = emit_jeff(SquinToJeff().emit(main))

# let's have a look
print(module)

module.write_out("main.jeff")
```

The conversion keeps the structure of your program.
The module holds three jeff functions, one for each kernel.
The loop over the register is a jeff `for`, and its body calls the other two functions.
Nothing is inlined or unrolled.

### Loading a jeff file

`load_jeff` reads and checks the file, and `JeffToSquin().emit` converts the jeff program to a squin kernel.

```python
from bloqade.jeff import JeffToSquin, load_jeff

roundtrip_kernel = JeffToSquin().emit(load_jeff("main.jeff"))
roundtrip_kernel.print()
```

The result is a squin kernel like any other, with the loop and the two calls in place.
You can call it from another kernel or run it via a simulator.
A rotation by pi between two `h` gates flips a qubit, so every measurement gives 1:

```python
from bloqade.pyqrack import DynamicMemorySimulator

result = DynamicMemorySimulator().run(roundtrip_kernel, args=(math.pi,))
print(result)
```

## Limitations

Both conversions map each statement to its direct counterpart.
If there is no counterpart, the conversion refuses the program with a `ValidationErrorGroup` that names each unsupported statement.
It never rewrites your program into a different one, so it does not decompose gates for you.

Some things to keep in mind:

* jeff measures one qubit at a time, so a kernel can only measure a list of qubits whose length is known when the conversion runs.
* A gate cannot take the same qubit twice.
* squin cannot express `while` loops, Pauli-product rotations, or gates with more controls than `cx`, `cy`, `cz` and a doubly controlled `z`.
* A jeff integer has 32 bits, whereas a squin integer is a Python integer.

If you run into an issue that you think should be supported, please [report an issue on the GitHub repository](https://github.com/QuEraComputing/bloqade-circuit/issues).
