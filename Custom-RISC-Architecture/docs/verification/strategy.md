# Hardware Verification Strategy

- Unit test are done on each hardware components to ensure its functionality. for example:

### ALU Unit Test

![1768504103927](image/results/1768504103927.png)


- Then, some test scripts are written to test the entire datapath, for example:

### Simple Control Flow Assembly Test Script:

- scripts.ISA_tests.control_flow.sample_control_flow
- This script tests the control flow instructions such as BE, BG, BL, and J. It checks if the program counter (PC) is updated correctly based on the conditions specified in the instructions.

```assembly
MOVI 0          // PC=1: Set R0 to 0
MOV R10         // PC=2: R10 = 0 (Address for result in MEM[0])
MOVI 2          // PC=3: Set R0 to 2
MOV R12         // PC=4: R12 = 2 (Address for num1 in MEM[2])
MOVI 3          // PC=5: Set R0 to 3
MOV R13         // PC=6: R13 = 3 (Address for num2 in MEM[3])
LOAD R12        // PC=7: R0 = DMem[R12] (Load byte from MEM[2] into R0)
MOV R1          // PC=8: R1 = R0 (Move num1 from R0 to R1)
LOAD R13        // PC=9: R0 = DMem[R13] (Load byte from MEM[3] into R0)
MOV R2          // PC=10: R2 = R0 (Move num2 from R0 to R2)
LOAD R12        // PC=11: R0 = DMem[R12] (Load num1 from MEM[2] into R0 for comparison)
BG R2           // PC=12: Compare R0 (num1) > R2 (num2).
J #SUB_PATH             // PC=13: J #SUB_PATH (Unconditionally jump to SUB_PATH if BG was FALSE)
ADD_PATH: LOAD R12        // PC=14: R0 = DMem[R12] (Load num1 into R0 again for addition)
ADD R2          // PC=15: R0 = R0 + R2 (R0 now holds num1 + num2)
STORE R10       // PC=16: DMem[R10] = R0 (Store the calculated sum into MEM[0])
J #END_PROGRAM             // PC=17: J #END_PROGRAM (Jump to end of program after addition)
SUB_PATH: LOAD R12        // PC=18: R0 = DMem[R12] (Load num1 into R0 again for subtraction)
SUB R2          // PC=19: R0 = R0 - R2 (R0 now holds num1 - num2)
STORE R10       // PC=20: DMem[R10] = R0 (Store the calculated difference into MEM[0])
END_PROGRAM: DONE            // PC=21: Signal the end of the program execution
```

Then use the **custom assembler written in Python** to convert this assembly code into machine code, JLUT, and initial DMEM

```
111000000
001111010
111000010
001111100
111000011
001111101
001001100
001110001
001001101
001110010
001001100
100110001
101100000
001001100
000100010
001011010
101100001
001001100
000110010
001011010
111111111
```

And verify the Result in the Modelsim
