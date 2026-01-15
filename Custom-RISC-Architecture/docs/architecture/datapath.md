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
    CTRL->>RF: Ra (Addr R1), WenR (1)

    Note over RF, ALU: Operand Fetch
    RF->>ALU: RdatA (Value of R1)
    RF->>ALU: RdatB (Value of R0/Accumulator)

    Note over ALU: Execute AND Operation
    ALU->>RF: Result (R1 & R0)
    
    Note left of RF: Writes Result into R0<br/>(Because WenR=1)

    ALU->>PC: ALU_COND (0)
    
    Note right of PC: Branch condition not met.<br/>Standard Increment.
    PC->>PC: PC = PC + 1

    Note over PC, IROM: Prepare Next Cycle
    PC->>IROM: Next Instruction Address
```

### Data Path `10_00_0_1010` (BE R10):
this checks if the `R0` value is equal to the value in `R10`. If they are equal, the `ALU_COND` flag is set to 1, and the Program Counter will be updated to skip the next instruction (PC += 2). If they are not equal, the `ALU_COND` flag is set to 0, and the Program Counter will increment normally (PC += 1).


### Quartus Synthesis:
![1768468440325](image/datapath/1768468440325.png)
