# Structural Wallace Tree Multiplier

![Verilog](https://img.shields.io/badge/Verilog-Structural%20RTL-blue)
![FPGA](https://img.shields.io/badge/FPGA-Xilinx%20Vivado-red)
![Digital Design](https://img.shields.io/badge/Digital%20Design-Multiplier%20Architecture-orange)
![Timing Analysis](https://img.shields.io/badge/Timing%20Analysis-WNS%20%2B%20WHS-green)
![Combinational Logic](https://img.shields.io/badge/Combinational%20Logic-HA%20%2B%20FA-lightgrey)

## Overview

This project implements an 8-bit unsigned structural Wallace tree multiplier in Verilog. The multiplier is built from manually generated partial products, half adders, and full adders instead of using the behavioral `*` operator.

The design was created to study low-level multiplier architecture, structural RTL design, FPGA timing behavior, and the performance difference between a single-cycle combinational multiplier and a multi-cycle shift-add multiplier.

## Project Highlights

- Designed an 8x8 unsigned Wallace tree multiplier using structural Verilog
- Generated 64 partial products using bitwise AND logic
- Reduced the partial product matrix using half adders and full adders
- Produced a 16-bit multiplication result through a single combinational datapath
- Added a registered timing wrapper to create a valid FPGA timing path
- Verified that the implemented design meets setup, hold, and pulse-width timing in Vivado
- Compared the single-cycle Wallace tree architecture against an 8-cycle shift-add multiplier

## Architecture

The multiplier accepts two 8-bit unsigned operands and produces a 16-bit product:

```verilog
module Wallace_Tree_Multiplier(
    input  [7:0]  A,
    input  [7:0]  B,
    output [15:0] sum
);
```

The design computes:

```text
sum = A * B
```

Instead of inferring a multiplier with the Verilog `*` operator, the design is implemented structurally.

## Partial Product Generation

Each partial product is generated with a bitwise AND operation.

Example:

```verilog
assign row1b0 = A[0] & B[0];
assign row1b1 = A[0] & B[1];
assign row2b0 = A[1] & B[0];
```

For an 8x8 multiplication, this creates 64 partial product bits arranged into weighted columns.

## Half Adder

The half adder is used when two one-bit values need to be added without a carry-in.

```verilog
module HA(
    input  A,
    input  B,
    output Sum,
    output Cout
);

assign Sum  = A ^ B;
assign Cout = A & B;

endmodule
```

## Full Adder

The full adder is used to add three one-bit values: two input bits and one carry-in.

```verilog
module FA(
    input  A,
    input  B,
    input  Cin,
    output Cout,
    output Sum
);

wire Temp;

assign Temp = A ^ B;
assign Sum  = Temp ^ Cin;
assign Cout = ((Cin & Temp) | (A & B));

endmodule
```

## Wallace Tree Reduction

The multiplier reduces each weighted column of the partial product matrix using structural `HA` and `FA` instances.

The least significant product bit is assigned directly:

```verilog
assign sum[0] = row1b0;
```

Column 1 is reduced with a half adder:

```verilog
HA col_1(
    .A(row1b1),
    .B(row2b0),
    .Cout(col_1_carryout1),
    .Sum(sum[1])
);
```

Larger columns are reduced with multiple full adders and half adders. The final carry becomes the most significant product bit:

```verilog
assign sum[15] = col_14_carryout1;
```

## Timing Wrapper

A registered wrapper was added so Vivado can analyze the multiplier as a synchronous FPGA datapath.

```verilog
module Critical_Path_Test(
    input  wire       clk,
    input  wire       rst,
    output reg [15:0] sum
);
```

The wrapper registers the multiplier inputs and output:

```verilog
always @(posedge clk) begin
    if (rst) begin
        sum <= 16'd0;
        A   <= 8'd0;
        B   <= 8'd0;
    end else begin
        A   <= 8'b11111111;
        B   <= 8'b11111111;
        sum <= wallace_out;
    end
end
```

This creates a timing path from input registers, through the combinational Wallace tree multiplier, and into the output register.

The multiplier instance is preserved with a `DONT_TOUCH` attribute:

```verilog
(* DONT_TOUCH = "true" *)
Wallace_Tree_Multiplier test(
    .A(A),
    .B(B),
    .sum(wallace_out)
);
```

This prevents Vivado from optimizing away the structural multiplier during timing analysis.

## Timing Results

Vivado static timing analysis reported that all user-specified timing constraints were met.

| Timing Metric | Result |
|---|---:|
| Worst Negative Slack, WNS | 0.813 ns |
| Total Negative Slack, TNS | 0.000 ns |
| Setup Failing Endpoints | 0 |
| Worst Hold Slack, WHS | 0.425 ns |
| Total Hold Slack, THS | 0.000 ns |
| Hold Failing Endpoints | 0 |
| Worst Pulse Width Slack, WPWS | 4.500 ns |
| Total Pulse Width Negative Slack, TPWS | 0.000 ns |
| Pulse Width Failing Endpoints | 0 |

The positive setup slack confirms that the multiplier met the target clock period. The positive hold slack confirms that the design also met hold timing. With zero failing endpoints across setup, hold, and pulse-width checks, the implemented design successfully closed timing.

Assuming a 10 ns target clock period, the reported WNS of 0.813 ns corresponds to an approximate register-to-register datapath delay of:

```text
10.000 ns - 0.813 ns = 9.187 ns
```

This is approximately:

```text
1 / 9.187 ns = 108.85 MHz
```

## Implementation Summary

| Metric | Result |
|---|---|
| Multiplier Type | 8x8 unsigned structural Wallace tree |
| Input Width | 8 bits x 8 bits |
| Output Width | 16 bits |
| Implementation Style | Structural Verilog |
| Main Building Blocks | Partial products, half adders, full adders |
| Datapath Style | Single-cycle combinational multiplier |
| Timing Path | Register -> Wallace multiplier -> Register |
| LUT Usage | Approximately 91 LUTs |
| Architecture Comparison | 1-cycle Wallace tree vs. 8-cycle shift-add multiplier |
| Throughput Comparison | 8x higher throughput than the 8-cycle shift-add design |
| Timing Status | All user-specified timing constraints met |

## Verification

The multiplier can be verified by comparing the structural output against the expected behavioral multiplication result:

```verilog
expected = A * B;
```

Representative test cases:

| A | B | Expected Product |
|---:|---:|---:|
| 0 | 0 | 0 |
| 1 | 1 | 1 |
| 2 | 3 | 6 |
| 15 | 15 | 225 |
| 255 | 1 | 255 |
| 255 | 255 | 65025 |

The maximum unsigned 8-bit multiplication case is:

```text
255 x 255 = 65025
```

which fits within the 16-bit output range.

## File Structure

```text
structural-wallace-tree-multiplier/
├── rtl/
│   ├── Wallace_Tree_Multiplier.v
│   ├── FA.v
│   ├── HA.v
│   └── Critical_Path_Test.v
├── docs/
│   ├── timing_summary.png
│   ├── utilization_report.png
│   └── schematic.png
└── README.md
```

## How to Run in Vivado XSim

Example compile flow:

```bash
xvlog rtl/HA.v rtl/FA.v rtl/Wallace_Tree_Multiplier.v rtl/Critical_Path_Test.v
xelab work.Critical_Path_Test -s wallace_timing_sim
xsim wallace_timing_sim -gui
```

For implementation and timing analysis, run synthesis and implementation in Vivado, then inspect:

```text
Report Timing Summary
Report Utilization
Report Schematic
```

## Skills Demonstrated

- Structural Verilog RTL design
- Combinational datapath design
- Binary multiplier architecture
- Manual partial product generation
- Half-adder and full-adder construction
- FPGA static timing analysis
- Critical path analysis
- Resource and throughput comparison
- Vivado synthesis and implementation workflow

## What I Learned

This project showed how multiplication is implemented below the behavioral `*` operator. Building the multiplier structurally made the partial product matrix, adder reduction network, carry propagation, and timing cost visible at the RTL/datapath level.

The timing wrapper also showed why synchronous register boundaries are important for meaningful FPGA timing analysis. By registering the multiplier inputs and output, Vivado can analyze the critical path through the combinational multiplier and report setup and hold timing for a realistic FPGA datapath.

## Summary

This completed project demonstrates an 8-bit unsigned structural Wallace tree multiplier implemented from half adders, full adders, and manually generated partial products. The design successfully met setup, hold, and pulse-width timing in Vivado, showing a complete structural RTL design flow from datapath construction through FPGA timing analysis.
