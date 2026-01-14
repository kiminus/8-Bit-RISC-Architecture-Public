# The Memory Map
# Instruction ROM
- Where the compiled machine code is stored, populated from the `mach_code.txt` when loading the program.
- Since the program we need to run on this architecture is small, it is decided that `PC` will be 16 bits wide. This allows for a maximum of 65536 instructions, which is more than enough for our needs.
# Data RAM

# Jump LUT (JLUT)
- A small lookup table that holds the target addresses for jump instructions.
- Since the `Jmp (10_11_XXXXX)` instrction only have 5 bits for the immediate value, We can only jump to 32 different locations.