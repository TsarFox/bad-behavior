![Bad BEHAVIOR][img_1]

A vulnerability was recently discovered that affects several implementations of
ACS, a domain-specific scripting language for Doom maps. As a stack machine, ACS
has several opcodes for incrementing and decrementing a stack pointer. If these
increments and decrements are performed without bounds checking, it is possible
for a maliciously-crafted BEHAVIOR lump to produce an out-of-bounds write. Here
is the implementation of the PUSHBYTE opcode, taken from GZDoom's ACS
interpreter:

```c
case PCD_PUSHBYTE:
        PushToStack (*(uint8_t *)pc);
        pc = (int *)((uint8_t *)pc + 1);
        break;
```

Where PushToStack is a macro defined as:

```c
#define PushToStack(a)	(Stack[sp++] = (a))
```

The stack pointer is incremented, but it does not check to see if the new value
is greater than the stack's maximum size. Because the buffer used by the
interpreter is placed on the process's stack, it is possible to overwrite the
interpreter routine's return pointer, resulting in arbitrary code execution.

**Maintainers of Doom source ports,**

It is critical that you inspect your port's implementation of the ACS
interpreter, and patch it if vulnerable to the exploit presented here. Every
opcode that modifies the stack pointer should perform a bounds check, ensuring
that it does not fall below 0, and does not rise greater than the maximum index
of the stack buffer. A rather crude example of a solution for PUSHBYTE is shown
below, however, brevity could be retained by making use of operator overloading,
or using a class with methods for pushing and popping that do perform bounds
checking.

```c
case PCD_PUSHBYTE:
        if (++sp >= STACK_SIZE) {
            I_Error("Corrupted stack pointer in ACS VM");
        }
        Stack[sp] = (*(uint8_t *)pc);
        pc = (int *)((uint8_t *)pc + 1);
        break;
```

Provided for vulnerability assessment is the file `example.wad`, which contains
a BEHAVIOR lump that will overwrite the interpreter routine's return pointer in
the latest GZDoom development builds. The stack offset should be different
across different source ports, but implementations where the vulnerability has
been dealt with should throw an error upon loading MAP01 regardless. 

Additionally provided is a Python script which will craft a malicious BEHAVIOR
lump to write 0xdeadbeefcafebabe at a given offset from the beginning of the
interpreter's stack buffer. This is intentionally very obtuse to work with,
however if you are a source port maintainer and would like to be able to further
experiment with the vulnerability, it is provided in its full form. A majority
of the code is present to form a valid BEHAVIOR lump; the real exploit is only
in the creation of the `payload` list.

A more detailed and prose-like writeup, with details on the research process is
[available here][1].

[1]: http://jakob.space/blog/post/Bad+BEHAVIOR

[img_1]: https://raw.githubusercontent.com/TsarFox/bad-behavior/master/logo.png
