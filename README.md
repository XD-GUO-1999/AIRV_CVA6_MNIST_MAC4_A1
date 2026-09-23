# AIRV CVA6 MNIST Accelerator — MAC4 Approach 1

This repository contains the **MAC4 Approach 1** version of the AIRV MNIST acceleration project on the CVA6 RISC-V processor.

In this version, the custom `MAC4` instruction is integrated **directly into the CVA6 execution path**. One instruction performs four parallel INT8 multiply-accumulate operations, while the destination register `rd` is reused as the accumulator source.

Main features:

- Custom RISC-V instruction: `mac4 rd, rs1, rs2`
- Four parallel 8-bit multiplications per MAC4 instruction
- `rd` reused as both accumulator source and destination
- Third integer register-file read port for the accumulator
- MAC4 execution inside the CVA6 multiplier unit
- Modified GNU assembler / opcode definitions
- MNIST software updated to use MAC4
- Alignment-aware scalar fallback for unsafe 32-bit loads

## Documentation

For environment setup, simulation, instruction encoding, the complete MAC4 data path, and a **file-by-file implementation guide with line numbers**:

👉 [User and Implementation Guide](USER_GUIDE_AIRV.md)

## Main Data Path

```text
NetworkPropagate.c
        ↓
Modified GNU assembler / opcode tables
        ↓
decoder.sv
        ↓
issue_read_operands.sv
        ↓
3-port integer register file
        ↓
mult.sv
        ↓
multiplier.sv
        ↓
rd write-back
```

## Note

This repository represents an intermediate architecture in the AIRV accelerator development. It is kept as a standalone implementation to document the evolution from scalar execution toward wider MAC instructions and the final buffered accelerator architecture.
