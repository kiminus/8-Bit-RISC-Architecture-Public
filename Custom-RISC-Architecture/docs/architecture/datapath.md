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
* **Control Unit:** Decodes the `mach_code` into control signals for the ALU and Memory.

## The Accumulator Constraint

To fit instructions into 9 bits, the design uses an **Accumulator** pattern.

* **Constraint:** R-Type instructions do not specify a destination register.
* **Solution:** The Control Unit hardwires the Write Destination (`Wd`) and Read Address A (`Ra`) to `R0_ADDR` (0x0) for arithmetic operations.

## Component breakdown

### ALU (Arithmetic Logic Unit)

The ALU handles 4-bit operations including `ADD`, `SUB`, `AND`, `OR`, and Bitwise Shifts (`LSL`, `RSL`).

* **Parity Flag:** The ALU calculates parity (`^Rslt`) in real-time.
* **Branch Feedback:** The ALU outputs a dedicated `ALU_Condition_True` signal directly to the Control Unit and PC, enabling single-cycle branch decisions.

### Register File

* **Size:** 16 x 8-bit registers.
* **Special Handling:** `R0` is the dedicated accumulator. Immediate values (like constants) are often loaded directly into `R0`.
