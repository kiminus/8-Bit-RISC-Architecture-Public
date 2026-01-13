# Instruction Set Architecture (ISA)

The processor utilizes a custom 9-bit ISA designed for high code density.
It supports **R-Type** (Register), **I-Type** (Immediate), and **J-Type** (Jump) formats.

## opcode Map

| Mnemonic      | Opcode   | Type | Description                  | Hardware Implementation                                  |
| :------------ | :------- | :--- | :--------------------------- | :------------------------------------------------------- |
| **AND** | `0000` | R    | `R0 = R0 & Rx` | [cite: 125]                                              |
| **OR**  | `0001` | R    | `R0 = R0                     | Rx`                                                      |
| **ADD** | `0010` | R    | `R0 = R0 + Rx` | [cite: 128]                                              |
| **SUB** | `0011` | R    | `R0 = R0 - Rx` | [cite: 129]                                              |
| **LDR** | `0100` | R    | `R0 = Mem[Rx]`             | Load from Data Memory [cite: 131]                        |
| **STR** | `0101` | R    | `Mem[Rx] = R0`             | Store to Data Memory [cite: 137]                         |
| **NOT** | `0110` | R    | `Wd = ~Rx`                 | Bitwise NOT [cite: 140]                                  |
| **MOV** | `0111` | R    | `Rx = R0`                  | Move R0 to Register (via ALU_OR) [cite: 141]             |
| **LSL** | `1010` | R    | `R0 = R0 << Rx`            | Logical Shift Left [cite: 145]                           |
| **RSL** | `1011` | R    | `R0 = R0 >> Rx`            | Logical Shift Right [cite: 146]                          |

## Control Flow (The "JLUT" Mechanism)

Conventional RISC architectures use PC-relative offsets. To compress 16-bit addressing into 9-bit instructions, this core uses a **Jump Look-Up Table (JLUT)**.

| Instruction                 | Code       | Description                          | Logic                                                           |
| :-------------------------- | :--------- | :----------------------------------- | :-------------------------------------------------------------- |
| **BE** (Branch Equal) | `J-Type` | Skip next instr if `R0 == Operand` | `PC = (Cond) ? PC+2 : PC+1` [cite: 191]                       |
| **BL** (Branch Less)  | `J-Type` | Skip next instr if `R0 < Operand`  | `ALU_Condition_True` check [cite: 156]                         |
| **JUMP**              | `J-Type` | `PC = JLUT[Ptr]`                   | Indirect 16-bit targeting via 5-bit ptr [cite: 157]            |
