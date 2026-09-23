# AIRV CVA6 MNIST Accelerator — MAC4 Approach 1 User and Implementation Guide

This guide documents the **MAC4 Approach 1** implementation used during the AIRV MNIST accelerator development on CVA6.

The purpose of this repository is not only to provide a runnable project, but also to preserve one important intermediate architecture in a form that can be understood and reproduced independently.

In **Approach 1**, MAC4 is integrated directly into the normal CVA6 execution path. It is decoded by the CPU, reads its operands from the CVA6 register file, executes inside the multiplier functional unit, and writes the result back to `rd`.

---

## 1. MAC4 Overview

The software syntax is:

```asm
mac4 rd, rs1, rs2
```

The two explicit source registers contain four packed 8-bit values each:

```text
rs1 = four unsigned 8-bit input values
rs2 = four signed 8-bit weight values
```

The destination register is also used as the accumulator source:

```text
rd = rd
   + input[0] * weight[0]
   + input[1] * weight[1]
   + input[2] * weight[2]
   + input[3] * weight[3]
```

Therefore MAC4 has three architectural register values to read:

```text
rs1
rs2
old rd
```

although the assembler syntax contains only the usual R-type fields:

```text
rd, rs1, rs2
```

To make this possible, the CVA6 integer register file is configured with a third read port.

---

## 2. Instruction Encoding

The custom instruction uses the RISC-V `CUSTOM-0` opcode space.

```text
funct7 = 0000000
funct3 = 001
opcode = 0001011
```

Instruction layout:

```text
31          25 24      20 19      15 14   12 11       7 6          0
+--------------+----------+----------+-------+-----------+------------+
| funct7 = 0   |   rs2    |   rs1    |  001  |    rd     |  0001011   |
+--------------+----------+----------+-------+-----------+------------+
```

The GNU toolchain represents the same encoding as:

```text
mac4 rd rs1 rs2 31..25=0 14..12=1 6..2=0x02 1..0=3
```

and the generated match/mask values are:

```c
#define MATCH_MAC4 0x100b
#define MASK_MAC4  0xfe00707f
```

---

## 3. Project Structure

The important source locations are:

```text
core/
├── cva6.sv
├── decoder.sv
├── issue_read_operands.sv
├── mult.sv
├── multiplier.sv
└── include/
    └── ariane_pkg.sv

sw/
└── app/
    └── mnist/
        └── NetworkPropagate.c

util/
├── riscv-opcodes/
│   └── extensions/
│       └── rv_i
└── gcc-toolchain-builder/
    └── src/
        └── binutils-gdb/
            ├── include/opcode/riscv-opc.h
            └── opcodes/riscv-opc.c
```

Set `PROJECTROOT` to the directory that contains `core/`, `sw/`, and `util/`.

---

## 4. Build and Simulation

Load the AIRV environment used on the development machine, then compile the MNIST application:

```bash
cd $PROJECTROOT/sw/app
make mnist
```

The build should generate:

```text
mnist.riscv
```

To run the RTL simulation:

```bash
cd $PROJECTROOT
make sim APP=mnist
```

The important validation points are:

```text
MNIST predicted class
instruction count
cycle count
UART output
Questa waveform
```

When modifying the implementation, first confirm that the functional result remains identical to the validated MAC4 Approach 1 version.

---

## 5. MAC4 End-to-End Data Path

The complete path is:

```text
NetworkPropagate.c
       │
       │  "mac4 rd, rs1, rs2"
       ▼
GNU assembler / opcode definitions
       │
       │  32-bit custom instruction
       ▼
decoder.sv
       │
       ├── rs1
       ├── rs2
       └── rd selected as third source
       ▼
issue_read_operands.sv
       │
       ├── register-file port 0 -> rs1
       ├── register-file port 1 -> rs2
       └── register-file port 2 -> old rd
       │
       │ old rd transported through the existing imm/third-operand path
       ▼
mult.sv
       │
       ▼
multiplier.sv
       │
       │ 4 × INT8 products + old rd
       ▼
registered MAC4 result
       │
       ▼
normal CVA6 write-back to rd
```

This is the main characteristic of **Approach 1**: the accelerator is inserted directly into the CPU multiplication datapath rather than being implemented as a CV-X-IF coprocessor.

---

# 6. Implementation Guide — Modified Files and Line Locations

> **Important:** the line numbers below refer to the cleaned MAC4 Approach 1 snapshot prepared together with this guide.  
> If the files are reformatted later, line numbers can move. Always use the **anchor / symbol name** together with the line number.  
> After validation, it is recommended to create a Git tag so these references remain stable.

A quick summary is:

| File | Current line(s) | Main modification |
|---|---:|---|
| `sw/app/mnist/NetworkPropagate.c` | 45–185, 224–680 | Integrates MAC4 into CNN dot products and handles aligned / unaligned data |
| `core/include/ariane_pkg.sv` | 457–458 | Adds `MAC4` to the functional-unit operation enum |
| `core/cva6.sv` | 173–176 | Configures three GPR read ports |
| `core/decoder.sv` | 1204–1229 | Decodes CUSTOM-0 MAC4 and dispatches it to `MULT` |
| `core/issue_read_operands.sv` | 181–185, 223–260, 461–469, 577, 609–615 | Reads old `rd` as a third source and handles hazards/forwarding |
| `core/mult.sv` | 39–41, 68–70 | Accepts MAC4 in the multiplier path and forwards the accumulator |
| `core/multiplier.sv` | 90–121, 153–155, 179–193 | Implements and pipelines the four-lane MAC4 arithmetic |
| `util/riscv-opcodes/extensions/rv_i` | 39–43 | Defines the MAC4 instruction encoding |
| `util/gcc-toolchain-builder/src/binutils-gdb/include/opcode/riscv-opc.h` | 39–51, 2814–2815 | Defines match/mask and instruction declaration |
| `util/gcc-toolchain-builder/src/binutils-gdb/opcodes/riscv-opc.c` | 332–337 | Adds `mac4` to the GNU opcode table |

---

## 6.1 Software Integration — `sw/app/mnist/NetworkPropagate.c`

This file integrates the custom instruction into the generated MNIST inference code.

### Lines 45–84 — `macsOnRange_with_mac4()`

Anchor:

```c
static void macsOnRange_with_mac4(...)
```

This helper processes four elements per iteration.

The software loads four input bytes and four weight bytes into two 32-bit registers:

```asm
lw t1, 0(%[p_in])
lw t2, 0(%[p_wt])
mac4 %[sum], t1, t2
```

The accumulator `sum` is both the input and output operand of MAC4:

```c
: [sum] "+r" (sum)
```

Any remaining elements after the groups of four are processed with scalar MAC operations.

### Lines 86–132 — `macsOnRange_no_alined()`

Anchor:

```c
static void macsOnRange_no_alined(...)
```

This is the alignment-safe helper.

Before issuing two 32-bit `lw` instructions, it checks:

```c
(addr_in & 0x3) == 0
(addr_wt & 0x3) == 0
```

If both pointers are 32-bit aligned, MAC4 is used.

If either pointer is unaligned, the group of four values falls back to four scalar operations:

```c
sum += inputs[iter + 0] * weights[iter + 0];
sum += inputs[iter + 1] * weights[iter + 1];
sum += inputs[iter + 2] * weights[iter + 2];
sum += inputs[iter + 3] * weights[iter + 3];
```

This avoids unsafe unaligned 32-bit loads.

### Lines 134–185 — `macsOnRange_no_alined_for_fc2()`

Anchor:

```c
static void macsOnRange_no_alined_for_fc2(...)
```

FC2 uses a dedicated helper because the starting weight address may not be 32-bit aligned.

The function checks the weight base address:

```c
(addr_wt & 0x3) == 0
```

If it is aligned, groups of four use MAC4. Otherwise the range falls back to scalar multiplication-accumulation.

### Lines 224–368 — `convcellPropagate1()`

Conv1 calls:

```c
macsOnRange_no_alined(...)
```

around lines 327 and 353.

Conv1 is handled with the alignment-safe path because sliding the convolution kernel can produce addresses that are not always suitable for a 32-bit `lw`.

### Lines 371–511 — `convcellPropagate2()`

Conv2 uses:

```c
macsOnRange_with_mac4(...)
```

around lines 472 and 497.

The contiguous Conv2 ranges can therefore execute four MAC operations with each custom instruction.

### Lines 514–596 — `fccellPropagateUDATA_T()`

This is the accelerated FC1 path.

It uses `macsOnRange_with_mac4()` around lines 569 and 586.

### Lines 598–680 — `fccellPropagateDATA_T()`

This is the FC2 path.

It uses:

```c
macsOnRange_no_alined_for_fc2(...)
```

around lines 653 and 670 to account for its weight alignment behavior.

---

## 6.2 Functional-Unit Operation — `core/include/ariane_pkg.sv`

### Lines 457–458

The `fu_op` enumeration is extended with:

```systemverilog
// MAC4: four parallel 8-bit multiplications accumulated into rd.
MAC4,
```

This identifier is used by the decoder, issue logic, `mult.sv`, and `multiplier.sv` to distinguish MAC4 from the normal multiplication operations.

---

## 6.3 Register-File Configuration — `core/cva6.sv`

### Lines 173–176

Anchor:

```systemverilog
localparam NrRgprPorts = 3;
```

A normal two-source integer instruction needs:

```text
rs1
rs2
```

MAC4 additionally needs the previous value of `rd`:

```text
rs1
rs2
old rd
```

Therefore the integer register file is configured with three read ports.

This is one of the central architectural modifications of Approach 1.

---

## 6.4 Custom Instruction Decode — `core/decoder.sv`

### Lines 1204–1229

The decoder recognizes:

```text
opcode = 0001011
funct3 = 001
funct7 = 0000000
```

The instruction is sent to the normal multiplier functional unit:

```systemverilog
instruction_o.fu = MULT;
```

The normal source and destination register fields are extracted:

```systemverilog
instruction_o.rs1[4:0] = instr.rtype.rs1;
instruction_o.rs2[4:0] = instr.rtype.rs2;
instruction_o.rd[4:0]  = instr.rtype.rd;
```

The important additional selection is:

```systemverilog
imm_select = RS3;
```

This activates the existing third-operand path so that the current value of `rd` can later be transported to the multiplier.

The cleaned snapshot explicitly checks both `funct3` and `funct7`, keeping the RTL decoder consistent with the GNU assembler encoding.

---

## 6.5 Operand Reading and Dependency Handling — `core/issue_read_operands.sv`

This is one of the most important files in MAC4 Approach 1.

### Lines 181–185 — use `rd` as the third source register

Anchor:

```systemverilog
rs3_o = (issue_instr_i.op == ariane_pkg::MAC4) ?
          issue_instr_i.rd[...] :
          issue_instr_i.result[...];
```

For MAC4:

```text
third source address = rd
```

so the processor reads the old accumulator value before the instruction overwrites the same architectural register.

### Lines 223–238 — hazard detection and forwarding

The third operand participates in dependency checking.

For MAC4, the scoreboard checks whether the old `rd` value is still being produced by an older instruction.

If the value is already available:

```systemverilog
forward_rs3 = 1'b1;
```

Otherwise:

```systemverilog
stall = 1'b1;
```

This prevents MAC4 from accumulating from a stale register value.

### Lines 253–260 — reuse the existing `imm` / third-operand datapath

For MAC4:

```systemverilog
imm_n = operand_c_regfile;
```

The signal named `imm` is therefore not an immediate value for MAC4.

It is reused as a convenient XLEN-wide third-operand path carrying:

```text
old rd
```

toward the multiplier functional unit.

### Lines 461–469 — three-port register-file address mapping

The mapping is:

```text
rdata[0] = rs1
rdata[1] = rs2
rdata[2] = old rd
```

The packed address vector is:

```systemverilog
{
    rd,
    rs2,
    rs1
}
```

for MAC4.

### Line 577 — map the third read-port data

Anchor:

```systemverilog
operand_c_regfile
```

The third GPR read-port output is selected as the third operand used by MAC4.

### Lines 609–615 — register-file configuration assertion

The assertion accepts either two or three integer read ports:

```systemverilog
CVA6Cfg.NrRgprPorts == 2 || CVA6Cfg.NrRgprPorts == 3
```

The error message was also updated so that it no longer describes the third port as CV-X-IF-specific.

---

## 6.6 Multiplier Functional-Unit Routing — `core/mult.sv`

### Lines 39–41

MAC4 is added to the operations accepted by the multiplication path:

```systemverilog
fu_data_i.operation inside {
    MUL,
    MULH,
    MULHU,
    MULHSU,
    MULW,
    CLMUL,
    CLMULH,
    CLMULR,
    ariane_pkg::MAC4
}
```

### Lines 68–70

The third operand is connected to the multiplier as:

```systemverilog
.operand_c_i(fu_data_i.imm)
```

For MAC4:

```text
fu_data_i.imm = old rd
```

This completes the path:

```text
third GPR read port
      ↓
operand_c_regfile
      ↓
imm_n / fu_data_i.imm
      ↓
operand_c_i
```

---

## 6.7 MAC4 Arithmetic — `core/multiplier.sv`

### Lines 90–110 — four-lane arithmetic

The MAC4 result is computed as:

```systemverilog
mac4_res_d =
    input0 * weight0 +
    input1 * weight1 +
    input2 * weight2 +
    input3 * weight3 +
    operand_c_i;
```

The implementation treats the four input bytes as unsigned values and the four weight bytes as signed values.

This matches the quantized CNN data representation:

```text
input  -> unsigned 8-bit
weight -> signed 8-bit
sum    -> 32-bit accumulator
```

### Line 121 — MAC4 valid-operation handling

MAC4 is included in the multiplier valid-operation set:

```systemverilog
operation_i inside {..., ariane_pkg::MAC4}
```

### Lines 153–155 — result selection

The output mux selects the registered MAC4 result when:

```systemverilog
operator_q == ariane_pkg::MAC4
```

### Lines 179–193 — pipeline register

The MAC4 result follows the existing multiplier pipeline:

```systemverilog
mac4_res_q <= mac4_res_d;
```

This keeps the custom operation integrated with the normal multiplier result timing and transaction tracking.

---

# 7. GNU Toolchain Modifications

MAC4 uses the normal R-type assembler operand format:

```text
d,s,t
```

Therefore no new custom GAS operand parser is required.

Only the instruction encoding and opcode tables need to be extended.

---

## 7.1 `util/riscv-opcodes/extensions/rv_i`

### Lines 39–43

Adds:

```text
mac4 rd rs1 rs2 31..25=0 14..12=1 6..2=0x02 1..0=3
```

This defines:

```text
funct7 = 0000000
funct3 = 001
opcode = 0001011
```

---

## 7.2 `util/gcc-toolchain-builder/src/binutils-gdb/include/opcode/riscv-opc.h`

### Lines 39–51

Defines:

```c
#define MATCH_MAC4 0x100b
#define MASK_MAC4  0xfe00707f
```

### Lines 2814–2815

Registers the instruction declaration:

```c
DECLARE_INSN(mac4, MATCH_MAC4, MASK_MAC4)
```

---

## 7.3 `util/gcc-toolchain-builder/src/binutils-gdb/opcodes/riscv-opc.c`

### Lines 332–337

Adds the assembler/disassembler opcode-table entry:

```c
{"mac4", 0, INSN_CLASS_I, "d,s,t",
    MATCH_MAC4, MASK_MAC4, match_opcode, 0 },
```

The operand format:

```text
d,s,t
```

means:

```text
d -> rd
s -> rs1
t -> rs2
```

No special parser modification is needed because MAC4 keeps the standard R-type register layout.

---

# 8. Final Register Mapping

The complete mapping is:

```text
instruction rs1
      ↓
register-file port 0
      ↓
operand_a
      ↓
four packed uint8 inputs

instruction rs2
      ↓
register-file port 1
      ↓
operand_b
      ↓
four packed int8 weights

instruction rd
      ↓
register-file port 2
      ↓
operand_c_regfile
      ↓
imm / third-operand datapath
      ↓
operand_c_i
      ↓
32-bit accumulator
```

The result returns through the normal multiplier write-back path to the same `rd`.

---

# 9. Recommended Debugging Order

When MAC4 produces an incorrect result, check the design in this order:

```text
1. Inspect the generated disassembly
   ↓
2. Verify MAC4 opcode / funct3 / funct7
   ↓
3. Check decoder.sv
   ↓
4. Check issue_read_operands.sv raddr_pack
   ↓
5. Verify rs1 / rs2 / old-rd values
   ↓
6. Check forwarding / stall behavior for old rd
   ↓
7. Check fu_data_i.imm
   ↓
8. Check multiplier operand_c_i
   ↓
9. Check mac4_res_d
   ↓
10. Check mac4_res_q and final rd write-back
```

Useful signals include:

```text
issue_instr_i.op
rs1_o
rs2_o
rs3_o
raddr_pack
operand_a_regfile
operand_b_regfile
operand_c_regfile
imm_n
fu_data_i.operation
fu_data_i.operand_a
fu_data_i.operand_b
fu_data_i.imm
mac4_res_d
mac4_res_q
result_o
```

---

# 10. Validation Before Pushing a Change

After editing the implementation:

```bash
git diff
```

Then rebuild and simulate:

```bash
cd $PROJECTROOT/sw/app
make clean
make mnist

cd $PROJECTROOT
make sim APP=mnist
```

Compare at least:

```text
Predicted class
CNN output values
instruction count
cycle count
MAC4 waveform behavior
```

For RTL changes intended for FPGA use, also rerun synthesis / implementation and check:

```text
LUT
FF
WNS
TNS
```

---

# 11. Repository Freeze

Because this document contains exact source-line references, freeze the validated repository state after the files and guide are confirmed:

```bash
git status
git add README.md USER_GUIDE_AIRV.md
git commit -m "Add MAC4 Approach 1 user and implementation guide"
git push
```

Then optionally create a tag:

```bash
git tag mac4-a1-v1.0
git push origin mac4-a1-v1.0
```

The tag provides an immutable reference for all line numbers documented above.
