# Custom RISC Architecture Documentation

[Source Code Repo (private)](https://github.com/kiminus/CPU-Design)

## Project Overview

This project involves the architecture, design, and verification of a custom **Single-Cycle RISC Processor**. Designed from scratch in **SystemVerilog**, the core utilizes an **Accumulator-based architecture** to minimize instruction width to 9 bits while maintaining a 16-bit addressable memory space. Designed from python -> C++ -> SystemVerilog.

## Key Features

* **9-Bit Custom ISA:** Efficiently packs 4-bit opcodes and operands, maximizing code density.
* **Accumulator Architecture:** Implicit `R0` targeting to reduce instruction overhead.
* **JLUT (Jump Look-Up Table):** A decoupled logic unit handling 16-bit addressing within 9-bit constraints.
* **Skip-on-Condition:** Branching logic that decouples evaluation from execution, reducing control complexity.
