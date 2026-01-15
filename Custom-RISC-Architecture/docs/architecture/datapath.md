# Datapath & Components

The architecture is built around a centralized bus connecting the Control Unit, Register File, and ALU.

### Overview of the Datapath:
```mermaid
graph TD
    %%--- Styling ---
    classDef central fill:#f9f,stroke:#333,stroke-width:2px,color:black,font-weight:bold;
    classDef component fill:#e1f5fe,stroke:#0277bd,stroke-width:1px,color:black;
    classDef external fill:#fff3e0,stroke:#ef6c00,stroke-dasharray: 5 5,color:black;

    %%--- External Inputs/Outputs ---
    subgraph External Inputs
        CLK(clk)
        START(start)
        RESET(reset)
    end
    DONE(done):::external

    %%--- Center Component ---
    TOP[TOP LEVEL<br/>Central Glue / Bus / Clock Coord]:::central

    %%--- Left Side Components ---
    subgraph memory controllers
        JLUT[JLUT<br/>Jump Look Up Table]:::component
        MEM3[(External Memory)]:::external
        RF[RegFile<br/>Register File]:::component
        MEM2[(External Memory)]:::external
        IROM[InstROM<br/>Instruction ROM]:::component
        MEM1[(External Memory)]:::external
    end

    %%--- Right Side Components ---
    subgraph functional units
        PC[PC<br/>Program Counter]:::component
        ALU[ALU<br/>Arithmetic Logic Unit]:::component
        CTRL[CTRL<br/>Control Decoder]:::component
    end

    %%--- Connections ---

    %% External to TOP
    CLK & START & RESET --> TOP
    TOP --> DONE

    %% PC connections
    TOP -- "..." --> PC
    PC -- PC value --> TOP

    %% JLUT connections
    TOP -- "..." --> JLUT
    JLUT -- jump address --> TOP
    JLUT <--> MEM3

    %% RegFile connections
    TOP -- "..." --> RF
    RF -- RdatA, RdatB --> TOP
    RF <--> MEM2

    %% InstROM connections
    TOP -- "..." --> IROM
    IROM <--> MEM1
    IROM -- mach_code --> TOP

    %% ALU connections
    TOP -- "..." --> ALU
    ALU -- result --> TOP
    ALU -- "flags (zero, parity, carry, condition)" --> TOP

    %% CTRL connections
    TOP -- "..." --> CTRL
    CTRL -- control signals --> TOP

    %%--- Layout hints to force side-by-side ---
    PC ~~~ IROM
    JLUT ~~~ ALU
    RF ~~~ CTRL
```

## Top-Level Connections

The `TopLevel` module coordinates the data flow between the Instruction ROM (`InstROM`) and the execution units.

* **Program Counter (PC):** Drives the address for the Instruction ROM.
* **Instruction ROM:** Outputs the 9-bit `mach_code` based on the PC.
* **Control Decoder:** Decodes the `mach_code` into control signals for the ALU and Memory.
* **ALU:** Executes arithmetic and logic operations based on the control signals and operands from the Register File.
* **Register File:** Provides the source operands for the ALU and receives results to be stored back.
* **Jump Look-Up Table (JLUT):** Provides target addresses for jump instructions, accessed by the Control Unit when needed.
* **Data RAM:** Stores data for load/store operations and intermediate results.
* **MUX:** Selects between different data sources based on control signals.

### Basic Components:

```mermaid
sequenceDiagram
    autonumber
    
    participant CTRL as Control Decoder
    participant DM as Data Memory
    participant RF as RegFile
    participant ALU
    participant JLUT
    participant PC as Program Counter
    participant IROM as Instruction ROM
```
### Data Path `0_0000_0001` (AND R1):

Control Decoder first decodes the `mach_code` from the Instruction ROM, which is `0_0000_0001` for the `AND R1` instruction. It decodes the `ALUop` (the ALU operation code), sends it to the ALU; and the `Ra` address (R1 in this case), sends it to the Register File to read the value of R1; `WenR` (write enable for the Register File) is set to 1, indicating that the result from the ALU should be written back to the Register File.

Register File reads the value of R1 from its internal register array and sends it to the ALU as `RdatA`. Similarly, it reads the value of R0 (the accumulator) and sends it to the ALU as `RdatB`.

The ALU performs the AND operation between `RdatA` (value of R1) and `RdatB` (value of R0, the accumulator), and sends the `result` back to the Register File to be stored in R0 (muxed); and output the `ALU_COND` (branch condition met) flag to 0 to the Program Counter to indicate that the next instruction should be executed sequentially. 

The Register File takes the `result` as the data to be written back to R0, and updates its internal register array accordingly (since `WenR` is set to 1). 

The Program controller, since the `ALU_COND` flag is 0, increments the `PC` by 1 to point to the next instruction in the Instruction ROM for the next cycle.

The `mach_code` for the next instruction is fetched from the Instruction ROM based on the updated `PC`, and the process repeats for the next instruction.

```mermaid
sequenceDiagram
    autonumber
    
    %% Participants in your requested order
    participant CTRL as Control Decoder
    participant DM as Data Memory
    participant RF as RegFile
    participant ALU
    participant JLUT
    participant PC as Program Counter
    participant IROM as Instruction ROM

    Note over CTRL, IROM: Start of Cycle: Fetch & Decode (AND R1)
    
    IROM->>CTRL: mach_code (0_0000_0001)
    
    Note right of CTRL: Decodes ALUop (AND),<br/>Target R1, and WenR=1
    
    CTRL->>ALU: ALUop (AND)
    CTRL->>RF: Ra (R0), Rb (R1), WenR (1)

    Note over RF, ALU: Operand Fetch
    RF->>ALU: RdatA (Value of R0/accumulator)
    RF->>ALU: RdatB (Value of R1) MUXED with Immediate

    Note over ALU: Execute AND Operation
    ALU->>RF: Result (R0 & R1), MUXED with DMem
    
    Note left of RF: Writes Result into R0<br/>(Because WenR=1)

    ALU->>PC: ALU_COND (0)
    
    Note right of PC: Branch condition not met.<br/>Standard Increment.
    PC->>PC: PC = PC + 1

    Note over PC, IROM: Prepare Next Cycle
    PC->>IROM: Next Instruction Address
```

### Data Path `10_00_0_1010` (BE R10):
Control Decorder decodes the `mach_code` for the `BE R10` instruction, which is `10_00_0_1010`. It sees it is a J-type instruction, therefore, it will send `ALUop` as subtraction, and `Ra` as R0,  `Rb` as R10, `ALU_branch` as `00` (indicate equal) to the ALU, which ask ALU to perform `R0 - R10`. The `WenR` is set to 0, indicating that the result from the ALU should not be written back to the Register File.

Register File reads the value of R10 from its internal register array and sends it to the ALU as `RdatB`. Similarly, it reads the value of R0 (the accumulator) and sends it to the ALU as `RdatA`.

Inside the ALU, it performs the subtraction, and set internal flags accordingly. It will then based on the the `ALU_branch` input to mux the correct flag (in this case, zero flag) to output as `ALU_Condition` flag to the Program Counter. 

The Program Counter checks the `ALU_Condition` flag. If it is 1 (indicating that R0 == R10), it will update the `PC` to `PC + 2` to skip the next instruction. If it is 0, it will simply increment the `PC` by 1 to point to the next instruction in the Instruction ROM for the next cycle.

The `mach_code` for the next instruction is fetched from the Instruction ROM based on the updated `PC`, and the process repeats for the next instruction.


```mermaid
sequenceDiagram
    autonumber
    
    participant CTRL as Control Decoder
    participant DM as Data Memory
    participant RF as RegFile
    participant ALU
    participant JLUT
    participant PC as Program Counter
    participant IROM as Instruction ROM

    %% 1. Fetch
    IROM->>CTRL: mach_code (10_00_0_1010)

    %% 2. Decode & Dispatch
    %% CTRL tells ALU to subtract and check for equality (ALU_branch = 00)
    CTRL->>ALU: ALUop (SUB), ALU_branch (00)
    
    %% CTRL tells RF which registers to read (Ra, Rb) and NOT to write back (WenR=0)
    CTRL->>RF: Ra (R0), Rb (R10), WenR (0)

    %% 3. Operand Fetch
    %% RF outputs the values. Note: User logic defined R10->RdatA and R0->RdatB
    RF->>ALU: RdatA (R0), RdatB (R10)

    %% 4. Execution & Flag Output
    %% ALU performs subtraction (R0 - R10). If result is 0, Zero Flag is high.
    ALU->>PC: ALU_Condition (Zero Flag)

    %% 5. PC Update Logic
    %% This highlights the specific decision logic for this path
    alt If ALU_Condition == 1 (Match)
        PC->>PC: PC = PC + 2 (Skip next)
    else If ALU_Condition == 0 (No Match)
        PC->>PC: PC = PC + 1 (Next line)
    end

    %% 6. Prepare Next Cycle
    PC->>IROM: Next Instruction Address
```

### Data Path `0_0101_0010` (STORE to Mem[2], or Mem[2] = R0):
Control Decoder decodes the `mach_code` for the `STORE R2` instruction, which is `0_0101_0010`. It will send `Write Enable Dmem` as 1 to the Data Memory, and `Rb` as R2 to the Register File to read the value of R2, `DMemAddr` as 2, indicateing that the value in R0 should be stored into memory address 2. The `WenR` is set to 0, indicating that the result from the ALU should not be written back to the Register File.

Register File reads the value of R2 from its internal register array and sends it to the Data memory and ALU as `RdatB`.

The Data Memory receives the `Write Enable Dmem` signal, the `RdatB` from the Register File (which is the value of R2), and the `DMemAddr` (which is 2) to perform the store operation. It writes the value of R2 into memory address 2 (non blocking).

The ALU is not involved in this instruction, it will output `ALU_Condition` flag as 0 to the Program Counter to indicate that the next instruction should be executed sequentially.

The Program Counter increments by 1 to point to the next instruction in the Instruction ROM for the next cycle.

```mermaid
sequenceDiagram
    autonumber
    
    participant CTRL as Control Decoder
    participant DM as Data Memory
    participant RF as RegFile
    participant ALU
    participant JLUT
    participant PC as Program Counter
    participant IROM as Instruction ROM

    %% 1. Fetch
    IROM->>CTRL: mach_code (0_0101_0010)

    %% 2. Decode & Control Signals
    %% CTRL sets up Data Memory write
    CTRL->>DM: Write Enable Dmem (1), DMemAddr (2)
    
    %% CTRL sets up Register Read (R2) but disables Register Write (WenR=0)
    CTRL->>RF: Rb (Addr R2), WenR (0)

    %% 3. Data Transfer
    %% RF sends value to DM (for storage) and ALU (unused connection)
    RF->>DM: RdatB (Value of R2)
    RF->>ALU: RdatB (Value of R2)

    %% 4. Memory Write Action
    Note right of DM: Writes Value of R2<br/>into Address 2

    %% 5. ALU / PC Update
    %% ALU is bypassed for logic, but still drives the condition line
    ALU->>PC: ALU_Condition (0)
    
    Note right of PC: Sequential Execution
    PC->>PC: PC = PC + 1

    %% 6. Prepare Next Cycle
    PC->>IROM: Next Instruction Address
```


### Quartus Synthesis:
![1768468440325](image/datapath/1768468440325.png)
