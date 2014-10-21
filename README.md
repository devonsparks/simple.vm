
* Git Repository:
    * http://github.com/skx/simple.vm
* Git mirror:
    * http://git.steve.org.uk/skx/simple.vm


simple.vm
---------

This repository contains the implementation for a simple virtual-machine, along with a driver which will read a program from a file and execute it via that virtual machine.

In addition to the virtual machine itself you'll also find:

* [A simple compiler](compiler).
    * This will translate from assembly-source into binary-opcodes.
* [A simple decompiler](decompiler).
    * This will translate in the other direction.
* Several [example programs](examples/).
* An example of [embedding](embedded.c) the virtual machine in a C host program.
    * Along with the definition of a custom-opcode handler.

This particular virtual machine is intentionally simple, but despite that it is hopefully implemented in a readable fashion.  ("Simplicity" here means that we support only a small number of instructions, and the registers the virtual CPU possesses can store strings and integers, but not floating-point values.)
This particular virtual machine is register-based, having ten registers which can be used to store strings or integer values.


Implementation Notes
--------------------

Implementing a basic virtual machine, like this one, is a pretty well-understood problem:

* Load the bytecode that is to be interpreted into a buffer.
* Fetch an instruction from from the buffer.
   * Decode the instruction, usually via a long `case` statement.
   * Advance the index such that the next instruction is ready to be interpreted.
   * Continue until you hit a `halt`, or `exit` instruction.
* Cleanup.

The main complications of writing a virtual machine are:

* Writing a compiler to convert readable source into the binary opcodes which will be actually executed.
    * Users want to write "`goto repeat`" not "`0x10 0x03 0x00`".
* Writing a disassembler for debugging purposes.
* Implementing a useful range of operations/opcodes.
    * For example "add", "dec", "inc", etc.

This particular virtual machine contains only a few primitives, but it does include the support for labels, looping, conditional jumps, calls and returns.  There is also a stack which can be used for storing temporary values, and used for `call`/`ret` handling.

The handling of labels in this implementation is perhaps worthy of note, because many simple/demonstration virtual machines don't handle them at all.

In order to support jumping to labels which haven't necessarily been defined yet our compiler keeps a running list of all labels (i.e. possible jump destinations) and when it encounters a jump instruction, or something else that refers to a label, it must outputs a placeholder-address, such as:

* `JUMP 0x000`
    * In our bytecode that is the three-byte sequence `0x10 0x00 0x00` as the JUMP instruction is defined as `0x10`.

After compilation is complete all the targets should have been discovered and the compiler is free to update the generated bytecodes to fill in the appropriate destinations.

>**NOTE**:  In our virtual machines all jumps are absolute, so they might look like "`JUMP 0x0123`" rather than "`JUMP -4`" or "`JUMP +30`".

>**NOTE**: The same thing applies for other instructions which handle labels, such as storing the address of a label, making a call, etc.


Embedding
---------

This virtual machine is designed primarily as a learning experience, but it is built with the idea of embedding in mind.

The standard `simple-vm` binary, which will read opcodes from a file and interpret them, is less than 25k in size.

Because the processing of binary opcodes is handled via a dispatch-table it is trivially possible for you to add your own application-specific opcodes to the system which would allow you to execute tiny compiled, and efficient, programs which can call back into your application when they wish.

There is an example of defining a custom opcode in the file `embedded.c`.  This example defines a custom opcode `0xCD`, and executes a small program which uses that opcode for demonstration purposes:

     $ ./embedded
     [stdout] Register R01 => 16962 [Hex:4242]
     Custom Handling Here
         Our bytecode is 8 bytes long




Instructions
------------

There are several instruction-types implemented:

* Storing string/int values into a given register.
* Mathematical operations:
    * add, and, sub, multiply, divide, incr, dec, or, & xor.
* Output the contents of a given register. (string/int).
* Jumping instructions.
    * Conditional and unconditional
* Comparison of register contents.
    * Against registers, or string/int constants.
* String to integer conversion, and vice-versa.
* Stack operations
    * PUSH/POP/CALL/RETURN

The instructions are pretty basic, as this is just a toy, but adding new ones isn't difficult and the available primitives are reasonably useful as-is.

The following are examples of all instructions:

    :test
    :label
    goto  0x44ff      # Jump to the given address
    goto  label       # Jump to the given label
    jmpnz label       # Jump to label if Zero-Flag is not set
    jmpz  label       # Jump to label if Zero-Flag is set

    store #1, 33      # store 33 in register 1
    store #2, "Steve" # Store the string "Steve" in register 1.
    store #1, #3      # register1 is set to the contents of register #3.

    exit              # Stop processing.
    nop               # Do nothing

    print_int #3      # Print the (integer) contents of register 3
    print_str #3      # Print the (string) contents of register 3

    system #3         # Call the (string) command stored in register 3

    add #1, #2, #3    # Add register 2 + register 3 contents, store in reg 1
    sub #1, #2, #3    # sub register 2 + register 3 contents, store in reg 1
    mul #1, #2, #3    # multiply register 2 + register 3 contents, store in reg 1
    concat #1, #2,#3  # store concatenated strings from reg2 + reg3 in reg1.

    dec #2            # Decrement the integer in register 2
    inc #2            # Increment the integer in register 2

    string2int #3     # Change register 3 to have a string from an int
    is_integer #3     # Does the given register have an integer content?
    int2string #3     # Change register 3 to have an int from a string
    is_string  #3     # Does the given register have a string-content?

    cmp #3, #4        # Compare contents of reg 3 & 4, set the Z-flag.
    cmp #3, 42        # Compare contents of reg 3 with the constant 42.  sets z.
    cmp #3, "Moi"     # Compare contents of reg 3 with the constant string "Moi".  sets z.

    peek #1, #4       # Load register 1 with the contents of the address in #4.
    poke #1, #4       # Set the address stored in register4 with the contents of reg1.
    random #2         # Store a random integer in register #2.

    push #1           # Store the contents of register #1 in the stack
    pop  #1           # Load register #1 with the contents of the stack.
    call 0xFFEE       # Call the given address.
    call my_label     # Call the defined label
    ret               # Return from a called-routine.


Simple Example
--------------

The following program will just endlessly output a string:

     :start
          store #1, "I like loops"
          print_str #1
          goto start

This program first stores the string "`I like loops`" in register 1, then prints that register, before jumping back to the start of the program.

To actually compile this program into bytecode run:

      $ ./compiler ./examples/simple.in

This will produce an output file full of binary-opcodes in the file `simple.raw`:

      $ od -t x1 -An ./examples/simple.raw
      01 01 0c 49 20 6c 69 6b 65 20 6c 6f 6f 70 73 03
      01 06 00 00

Now we can execute that series of opcodes:

      ./simple-vm ./examples/simple.raw

If you wish to debug the execution then run:

      DEBUG=1 ./simple-vm ./examples/simple.raw

There are more examples stored beneath the `examples/` subdirectory in this repository.   The file [examples/quine.in](examples/quine.in) provides a good example of various features - it outputs its own opcodes.

Steve
--
