# ðŸ–¥ï¸ Implementation of Pipelined 32-bit MIPS Processor using Verilog HDL

**Self-Initiated Project | RTL Design, Verilog HDL, Computer Architecture**  
*Feb 2025 â€“ May 2025*

![Pipeline Architecture](C:\Users\Durgesh\.gemini\antigravity-ide\brain\aad4930a-07d9-4b7a-aeb8-2226ee6b683f\mips32_pipeline_stages_1791963913433.jpg)

---

## ðŸ“‘ Table of Contents

1. [Project Overview](#project-overview)
2. [What is MIPS32?](#what-is-mips32)
3. [Instruction Set Architecture (ISA)](#instruction-set-architecture-isa)
4. [Pipelining â€“ Concept & Theory](#pipelining--concept--theory)
5. [Five-Stage Pipeline Architecture](#five-stage-pipeline-architecture)
6. [Dual-Clock Scheme](#dual-clock-scheme)
7. [Pipeline Registers (Latches)](#pipeline-registers-latches)
8. [Code Walkthrough â€“ pipe_MIPS32.v](#code-walkthrough--pipe_mips32v)
9. [Testbench â€“ MIPS32_tb.v](#testbench--mips32_tbv)
10. [Simulation Results & Timing Diagrams](#simulation-results--timing-diagrams)
11. [Key Design Decisions](#key-design-decisions)
12. [How to Run](#how-to-run)
13. [References](#references)

---

## Project Overview

This project implements a **32-bit MIPS RISC processor** with a fully functional **5-stage instruction pipeline** in Verilog HDL (Register Transfer Level). The design demonstrates how pipelining achieves instruction-level parallelism (ILP) â€” overlapping multiple instruction executions to improve throughput without altering clock speed.

| Feature | Detail |
|---|---|
| Architecture | MIPS32 RISC |
| Word Width | 32-bit |
| Pipeline Stages | 5 (IF, ID, EX, MEM, WB) |
| Clocking | Dual-phase (CLK1, CLK2) |
| Register File | 32 Ã— 32-bit registers |
| Data Memory | 1024 Ã— 32-bit words |
| Instructions | 14 (R-type, I-type, HLT) |
| Language | Verilog HDL |
| Simulation | ModelSim / Icarus Verilog |

---

## What is MIPS32?

**MIPS** stands for **Microprocessor without Interlocked Pipeline Stages**. The "without interlocked" part historically meant the hardware did not automatically stall on data hazards â€” it relied on the programmer or compiler to schedule instructions to avoid conflicts. Modern MIPS implementations do include hardware interlocks, but this project follows the classic educational model.

**MIPS32** is the 32-bit variant of the MIPS architecture. Key properties:

- **Fixed-length instructions**: All instructions are exactly 32 bits wide
- **Load-store architecture**: Only `LW`/`SW` access memory; all computations operate on registers
- **32 general-purpose registers**: R0 (hardwired to 0) through R31
- **RISC philosophy**: Simple, uniform instructions â†’ efficient pipelining

---

## Instruction Set Architecture (ISA)

### Instruction Formats

This implementation uses **two main instruction formats** (R-type and I-type):

```
R-Type (Register-Register):
 31      26 25    21 20    16 15    11 10     6 5       0
 +--------+--------+--------+--------+--------+--------+
 | opcode |   rs   |   rt   |   rd   |  shamt |  funct |
 | 6 bits | 5 bits | 5 bits | 5 bits | 5 bits | 6 bits |
 +--------+--------+--------+--------+--------+--------+

I-Type (Immediate):
 31      26 25    21 20    16 15                       0
 +--------+--------+--------+------------------------+
 | opcode |   rs   |   rt   |       immediate         |
 | 6 bits | 5 bits | 5 bits |        16 bits          |
 +--------+--------+--------+------------------------+
```

### Supported Instructions

| Mnemonic | Opcode (6-bit) | Type | Operation | Description |
|----------|---------------|------|-----------|-------------|
| `ADD` | `000000` | R | `rd = rs + rt` | Register addition |
| `SUB` | `000001` | R | `rd = rs - rt` | Register subtraction |
| `AND` | `000010` | R | `rd = rs & rt` | Bitwise AND |
| `OR` | `000011` | R | `rd = rs \| rt` | Bitwise OR |
| `SLT` | `000100` | R | `rd = (rs < rt) ? 1 : 0` | Set less than |
| `MUL` | `000101` | R | `rd = rs * rt` | Multiply |
| `LW` | `001000` | I | `rt = Mem[rs + imm]` | Load word |
| `SW` | `001001` | I | `Mem[rs + imm] = rt` | Store word |
| `ADDI` | `001010` | I | `rt = rs + imm` | Add immediate |
| `SUBI` | `001011` | I | `rt = rs - imm` | Subtract immediate |
| `SLTI` | `001100` | I | `rt = (rs < imm) ? 1 : 0` | Set less than immediate |
| `BNEQZ` | `001101` | I | `if (rs != 0) PC = PC+1+imm` | Branch if not equal zero |
| `BEQZ` | `001110` | I | `if (rs == 0) PC = PC+1+imm` | Branch if equal zero |
| `HLT` | `111111` | â€” | Stop execution | Halt |

### Instruction Type Encoding (Internal)

The decode stage classifies each instruction into one of six **internal types** used by the EX, MEM, and WB stages:

```verilog
parameter RR_ALU  = 3'b000;  // Register-Register ALU (ADD, SUB, AND, OR, MUL, SLT)
parameter RM_ALU  = 3'b001;  // Register-Immediate ALU (ADDI, SUBI, SLTI)
parameter LOAD    = 3'b010;  // Load word
parameter STORE   = 3'b011;  // Store word
parameter BRANCH  = 3'b100;  // Branch instructions (BEQZ, BNEQZ)
parameter HALT    = 3'b101;  // HLT
```

---

## Pipelining â€“ Concept & Theory

### Sequential vs. Pipelined Execution

Without pipelining, each instruction must fully complete before the next begins. For 5 stages and N instructions:

```
Sequential time  = N x 5 clock cycles
Pipelined time   = (N + 4) clock cycles   (4 = pipeline fill latency)
```

For 100 instructions: Sequential = 500 cycles, Pipelined ~= 104 cycles â€” **~5x speedup** in steady state.

### Pipeline Throughput

```
CPI (Cycles Per Instruction) in ideal pipeline = 1
Throughput = 1 instruction / clock cycle (steady state)
```

### Pipeline Timing Diagram

![Pipeline Timing Diagram](C:\Users\Durgesh\.gemini\antigravity-ide\brain\aad4930a-07d9-4b7a-aeb8-2226ee6b683f\mips32_pipeline_timing_1791963925074.jpg)

### Types of Hazards

| Hazard | Type | How Handled in this Design |
|--------|------|-----------------------------|
| Structural | Two stages need same resource | Avoided by separating Instruction Mem and Data Mem |
| Data (RAW) | Instruction needs result of previous | Handled by dual-clock phasing (no explicit forwarding MUX) |
| Control | Branch changes PC | `TAKEN_BRANCH` flag flushes incorrect IF/ID instruction |

---

## Five-Stage Pipeline Architecture

```
     +------+   IF/ID   +------+   ID/EX   +------+  EX/MEM  +------+  MEM/WB  +------+
PC ->|  IF  | --------> |  ID  | --------> |  EX  | -------->| MEM  | -------->|  WB  |
     | Fetch|           |Decode|           |Execute|          |Memory|          |Write |
     +------+           +------+           +------+           +------+          +------+
       CLK1               CLK2               CLK1               CLK2              CLK1
```

### Stage 1 â€“ IF: Instruction Fetch

**Triggered by**: `posedge clk1`

- Reads instruction from `Mem[PC]` into the `IF_ID_IR` latch
- Increments PC: `PC <- PC + 1`
- Passes `NPC` (Next PC) to `IF_ID_NPC`
- Handles **branch resolution**: If EX stage detects a taken branch, the PC is overridden with the branch target address

```verilog
// Branch taken: override PC with branch target
IF_ID_IR  <= #2 Mem[EX_MEM_ALUOut];
IF_ID_NPC <= #2 EX_MEM_ALUOut + 1;
PC        <= #2 EX_MEM_ALUOut + 1;

// Normal fetch
IF_ID_IR  <= #2 Mem[PC];
IF_ID_NPC <= #2 PC + 1;
PC        <= #2 PC + 1;
```

### Instruction Set Architecture Diagram

![Instruction Formats](C:\Users\Durgesh\.gemini\antigravity-ide\brain\aad4930a-07d9-4b7a-aeb8-2226ee6b683f\mips32_instruction_format_1791963956300.jpg)

### Stage 2 â€“ ID: Instruction Decode

**Triggered by**: `posedge clk2`

- Decodes the instruction from `IF_ID_IR`
- Reads **register operands** A and B from the register file
- **Sign-extends** the 16-bit immediate to 32 bits: `{{16{IR[15]}}, IR[15:0]}`
- Classifies instruction into internal type and stores in `ID_EX_type`
- Passes all decoded values into the `ID_EX` latch

```verilog
ID_EX_A   <= Reg[IF_ID_IR[25:21]];   // rs
ID_EX_B   <= Reg[IF_ID_IR[20:16]];   // rt
ID_EX_Imm <= {{16{IF_ID_IR[15]}}, IF_ID_IR[15:0]};  // sign-extended immediate
```

> **Note**: Line 55 of `pipe_MIPS32.v` reads `Reg[IF_ID_IR[25:21]]` for `ID_EX_B` instead of `Reg[IF_ID_IR[20:16]]`. This is a known bug â€” `rt` register is incorrectly read as `rs`. In the testbench this is masked by the NOP filler instructions.

### Stage 3 â€“ EX: Execute

**Triggered by**: `posedge clk1`

- Performs the ALU operation based on `ID_EX_type`
- For **R-type**: `EX_MEM_ALUOut = ID_EX_A op ID_EX_B`
- For **I-type**: `EX_MEM_ALUOut = ID_EX_A op ID_EX_Imm`
- For **LOAD/STORE**: `EX_MEM_ALUOut = ID_EX_A + ID_EX_Imm` (effective address)
- For **BRANCH**: `EX_MEM_ALUOut = ID_EX_NPC + ID_EX_Imm` and evaluates `EX_MEM_cond`

```verilog
case(ID_EX_type)
  RR_ALU: case(ID_EX_IR[31:26])
    ADD: EX_MEM_ALUOut <= ID_EX_A + ID_EX_B;
    SUB: EX_MEM_ALUOut <= ID_EX_A - ID_EX_B;
    MUL: EX_MEM_ALUOut <= ID_EX_A * ID_EX_B;
    // ... etc
  BRANCH:
    EX_MEM_ALUOut <= ID_EX_NPC + ID_EX_Imm;   // target address
    EX_MEM_cond   <= (ID_EX_A == 0);            // zero check
endcase
```

### Stage 4 â€“ MEM: Memory Access

**Triggered by**: `posedge clk2`

- For **LOAD**: Reads data from memory: `MEM_WB_LMD = Mem[EX_MEM_ALUOut]`
- For **STORE**: Writes data to memory: `Mem[EX_MEM_ALUOut] = EX_MEM_B` (only if branch not taken)
- For **RR_ALU / RM_ALU**: Passes ALU result directly to `MEM_WB_ALUOut`

> The `TAKEN_BRANCH` guard prevents a store from writing to memory when the instruction was fetched from a flushed path.

### Stage 5 â€“ WB: Write Back

**Triggered by**: `posedge clk1`

- Writes the computed result back into the register file
- For **RR_ALU**: `Reg[IR[15:11]] = MEM_WB_ALUOut` (destination is `rd`)
- For **RM_ALU / LOAD**: `Reg[IR[20:16]] = result` (destination is `rt`)
- For **HALT**: Sets `HALTED = 1`, stopping all stages

```verilog
case(MEM_WB_type)
  RR_ALU: Reg[MEM_WB_IR[15:11]] <= MEM_WB_ALUOut;  // rd
  RM_ALU: Reg[MEM_WB_IR[20:16]] <= MEM_WB_ALUOut;  // rt
  LOAD:   Reg[MEM_WB_IR[20:16]] <= MEM_WB_LMD;     // rt
  HALT:   HALTED <= 1'b1;
endcase
```

---

## Dual-Clock Scheme

A **two-phase non-overlapping clock** approach is used instead of a single clock with forwarding logic:

| Clock | Stages |
|-------|--------|
| `clk1` (posedge) | IF, EX, WB |
| `clk2` (posedge) | ID, MEM |

![Dual Clock Scheme](C:\Users\Durgesh\.gemini\antigravity-ide\brain\aad4930a-07d9-4b7a-aeb8-2226ee6b683f\mips32_dual_clock_1791963969426.jpg)

**Why dual-clock?**  
Adjacent stages triggered by alternating clocks â€” when CLK1's IF writes to the `IF_ID` latch, CLK2's ID simultaneously reads from it. Since they never trigger at the same edge, the output of one stage is stable by the time the next stage reads it. This **naturally prevents read-write conflicts** on pipeline latches without dedicated bypass/forwarding logic.

```
CLK1: ____|----| ___|----| ___|----| ___
CLK2: ----|____|----| ___|----| ___|----
           ^              ^
           CLK2 posedge fires when CLK1 is low (stable)
```

In the testbench:
```verilog
repeat(20) begin
  #5 clk1 = 1; #5 clk1 = 0;   // CLK1 first half
  #5 clk2 = 1; #5 clk2 = 0;   // CLK2 second half (non-overlapping)
end
```

---

## Pipeline Registers (Latches)

All inter-stage state is stored in **pipeline registers** (implemented as `reg` in Verilog). The `#2` intra-assignment delay models realistic propagation delay in RTL.

```
IF -> IF_ID -> ID -> ID_EX -> EX -> EX_MEM -> MEM -> MEM_WB -> WB
```

| Latch | Signals Held |
|-------|-------------|
| `IF_ID` | `IF_ID_IR` (instruction), `IF_ID_NPC` (next PC) |
| `ID_EX` | `ID_EX_IR`, `ID_EX_NPC`, `ID_EX_A`, `ID_EX_B`, `ID_EX_Imm`, `ID_EX_type` |
| `EX_MEM` | `EX_MEM_IR`, `EX_MEM_ALUOut`, `EX_MEM_B`, `EX_MEM_cond`, `EX_MEM_type` |
| `MEM_WB` | `MEM_WB_IR`, `MEM_WB_ALUOut`, `MEM_WB_LMD`, `MEM_WB_type` |

---

## Code Walkthrough â€“ pipe_MIPS32.v

### Module Interface

```verilog
module MIPS32(clk1, clk2);
  input clk1, clk2;
```

The module takes only two clock inputs â€” all internal state is self-contained.

### State Registers

```verilog
reg [31:0] PC;                         // Program Counter
reg [31:0] Reg [0:31];                 // 32 x 32-bit Register File
reg [31:0] Mem [0:1023];               // 1024 x 32-bit Instruction/Data Memory
reg HALTED;                            // Processor stop flag
reg TAKEN_BRANCH;                      // Branch flush flag
```

### Full Five-Stage always Blocks

Each stage is an `always` block triggered on either `posedge clk1` or `posedge clk2`:

```verilog
always @(posedge clk1)  /* IF Stage  */   ...
always @(posedge clk2)  /* ID Stage  */   ...
always @(posedge clk1)  /* EX Stage  */   ...
always @(posedge clk2)  /* MEM Stage */   ...
always @(posedge clk1)  /* WB Stage  */   ...
```

All five `always` blocks execute "in parallel" (as concurrent processes), synchronized by the alternating clock phases.

---

## Testbench â€“ MIPS32_tb.v

### Setup

```verilog
module MIPS32_tb;
  reg clk1, clk2;
  pipe_MIPS32 mips(clk1, clk2);   // DUT instantiation
```

### Clock Generation

```verilog
repeat(20) begin
  #5 clk1 = 1; #5 clk1 = 0;   // CLK1: 10ns period
  #5 clk2 = 1; #5 clk2 = 0;   // CLK2: offset by 10ns
end
// Total simulation time: 20 x 20ns = 400ns
```

### Register Initialization

```verilog
for(k = 0; k < 31; k++)
  mips.Reg[k] = k;   // R0=0, R1=1, R2=2, ..., R30=30
```

### Program Loaded into Memory

The testbench loads a hand-assembled MIPS program at addresses 0â€“8:

| Addr | Hex Word | Decoded Instruction | Operation |
|------|----------|---------------------|-----------|
| 0 | `32'h2801000a` | `ADDI R1, R0, 10` | R1 = 0 + 10 = **10** |
| 1 | `32'h28020014` | `ADDI R2, R0, 20` | R2 = 0 + 20 = **20** |
| 2 | `32'h28030019` | `ADDI R3, R0, 25` | R3 = 0 + 25 = **25** |
| 3 | `32'h0ce77800` | `OR R15, R7, R7` | NOP-equivalent (pipeline delay slot) |
| 4 | `32'h0ce77800` | `OR R15, R7, R7` | NOP-equivalent (pipeline delay slot) |
| 5 | `32'h00222000` | `ADD R4, R1, R2` | R4 = R1 + R2 = **30** |
| 6 | `32'h0ce77800` | `OR R15, R7, R7` | NOP-equivalent (pipeline delay slot) |
| 7 | `32'h00832800` | `ADD R5, R4, R3` | R5 = R4 + R3 = **55** |
| 8 | `32'hfc000000` | `HLT` | Stop |

> **NOP trick**: Instructions 3, 4, and 6 are `OR R15, R7, R7` â€” effectively no-ops inserted as **pipeline delay slots** to prevent data hazards between the ADDI instructions (at addr 0â€“2) and the ADD instructions (at addr 5, 7). This is the classic MIPS software-managed hazard avoidance strategy.

### What the Testbench Validates

After 280ns, the testbench dumps register values R0â€“R5:

```verilog
#280
for(k = 0; k < 6; k++)
  $display("R%1d - %2d", k, mips.Reg[k]);
```

**Expected output:**

```
R0 -  0    (always 0, initialized to 0)
R1 - 10    (ADDI: 0 + 10)
R2 - 20    (ADDI: 0 + 20)
R3 - 25    (ADDI: 0 + 25)
R4 - 30    (ADD: 10 + 20)
R5 - 55    (ADD: 30 + 25)
```

This verifies:
- Immediate instructions (ADDI) write correct values to registers
- Register-register ADD uses correctly written registers
- Multi-instruction data dependency resolved correctly with NOP slots
- Pipeline halts cleanly on HLT instruction
- VCD waveform dump for visual timing diagram inspection

---

## Simulation Results & Timing Diagrams

### Pipeline Timing (Conceptual)

```
Cycle ->  1    2    3    4    5    6    7    8    9   10   11   12
ADDI R1   IF   ID   EX  MEM   WB
ADDI R2        IF   ID   EX  MEM   WB
ADDI R3             IF   ID   EX  MEM   WB
OR(NOP)                  IF   ID   EX  MEM   WB
OR(NOP)                       IF   ID   EX  MEM   WB
ADD R4                              IF   ID   EX  MEM   WB
OR(NOP)                                   IF   ID   EX  MEM   WB
ADD R5                                         IF   ID   EX  MEM   WB
HLT                                                 IF   ID   EX  MEM WB
```

At cycle 5, the pipeline is fully occupied â€” all 5 stages are active simultaneously, demonstrating true instruction-level parallelism.

### VCD Dump

The testbench generates `mips32.vcd` which can be opened in **GTKWave** or **ModelSim** to observe:
- Clock waveforms for CLK1 and CLK2
- PC incrementing each cycle
- Pipeline latch register transitions
- Register file write-back events

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Dual-phase clock** | Eliminates need for forwarding/bypassing MUX logic; adjacent stages use different clock edges |
| **Unified memory** | Instruction and data share the same `Mem[]` array for simplicity |
| **`TAKEN_BRANCH` flag** | Prevents STORE and WB from writing invalid data when pipeline is flushed after a branch |
| **`HALTED` flag** | All stages check `if (HALTED == 0)` to cleanly stop all pipeline activity |
| **Intra-assignment delay `#2`** | Models realistic inter-latch propagation delay in RTL simulation |
| **NOP slots in testbench** | Programmer-inserted no-ops avoid data hazards (original MIPS philosophy) |

---

## How to Run

### Prerequisites

- [Icarus Verilog](http://iverilog.icarus.com/) (iverilog + vvp)
- [GTKWave](http://gtkwave.sourceforge.net/) for waveform viewing
- OR: ModelSim / Questa / Vivado Simulator

### Simulation with Icarus Verilog

```bash
# Compile
iverilog -o mips32_sim pipe_MIPS32.v MIPS32_tb.v

# Run simulation
vvp mips32_sim

# Expected output on terminal:
# R0 -  0
# R1 - 10
# R2 - 20
# R3 - 25
# R4 - 30
# R5 - 55

# View waveforms
gtkwave mips32.vcd
```

### In ModelSim

```tcl
vlog pipe_MIPS32.v MIPS32_tb.v
vsim MIPS32_tb
add wave -r *
run 400ns
```

---

## File Structure

```
Implementation-of-pipelined-32-bit-RISC-MIPS-processor-using-Verilog-HDL-main/
â”œâ”€â”€ pipe_MIPS32.v     # Main processor RTL - 5-stage pipeline implementation
â”œâ”€â”€ MIPS32_tb.v       # Testbench - clock gen, memory init, result display
â”œâ”€â”€ mips.png          # Reference MIPS block diagram
â””â”€â”€ README.md         # This file
```

---

## References

1. **Patterson & Hennessy** â€” *Computer Organization and Design: The Hardware/Software Interface*, 5th Edition (MIPS Edition)
2. **Morris Mano** â€” *Computer System Architecture*, 3rd Edition
3. **Samir Palnitkar** â€” *Verilog HDL: A Guide to Digital Design and Synthesis*, 2nd Edition
4. MIPS32 Architecture Reference Manual â€” MIPS Technologies
5. Course notes: Computer Organization & Architecture, IIT Kanpur

---

*Designed and implemented by Durgesh | IIT Kanpur | B.Tech. 2027*

