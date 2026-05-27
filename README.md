# VSD-FPGA-FABRIC-DESIGN-AND-ARCHITECTURE
# FPGA — Fabric, Design and Architecture

This repository documents the work carried out during the **FPGA — Fabric, Design and Architecture** workshop. The workflow is built around open-source EDA tools wherever possible, covering the full FPGA design cycle from RTL to bitstream.

---

## Table of Contents

- [What is an FPGA?](#what-is-an-fpga)
- [FPGA vs ASIC — A Quick Comparison](#fpga-vs-asic--a-quick-comparison)
- [Day 1 — FPGA Fundamentals and the Vivado Toolchain](#day-1--fpga-fundamentals-and-the-vivado-toolchain)
  - [Internal Architecture of an FPGA](#internal-architecture-of-an-fpga)
  - [Inside the Configurable Logic Block](#inside-the-configurable-logic-block)
  - [Basys 3 Development Board](#basys-3-development-board)
  - [4-bit Counter Design in Vivado](#4-bit-counter-design-in-vivado)
  - [Simulation and Elaboration](#simulation-and-elaboration)
  - [Synthesis](#synthesis)
  - [Implementation](#implementation)
  - [Design Constraints](#design-constraints)
  - [Bitstream Generation](#bitstream-generation)
  - [Timing, Power and Area Reports](#timing-power-and-area-reports)
  - [Virtual I/O (VIO) Core](#virtual-io-vio-core)
- [Day 2 — OpenFPGA, VPR and the VTR Framework](#day-2--openfpga-vpr-and-the-vtr-framework)
  - [OpenFPGA Overview](#openfpga-overview)
  - [VPR — Versatile Place and Route](#vpr--versatile-place-and-route)
  - [VTR — Verilog to Routing](#vtr--verilog-to-routing)
  - [Running the VTR Flow](#running-the-vtr-flow)
  - [Post-Synthesis Simulation](#post-synthesis-simulation)
  - [Timing Analysis in VTR](#timing-analysis-in-vtr)
  - [Power Analysis in VTR](#power-analysis-in-vtr)
- [Day 3 — RISC-V Core Implementation on Vivado](#day-3--risc-v-core-implementation-on-vivado)
  - [RTL to Synthesis](#rtl-to-synthesis)
  - [Synthesis to Bitstream](#synthesis-to-bitstream)
- [Day 4 — SOFA FPGA Fabric Introduction](#day-4--sofa-fpga-fabric-introduction)
  - [Counter Area on SOFA](#counter-area-on-sofa)
  - [Counter Timing on SOFA](#counter-timing-on-sofa)
  - [Counter Post-Implementation on SOFA](#counter-post-implementation-on-sofa)
  - [Counter Power on SOFA](#counter-power-on-sofa)
- [Day 5 — Mapping the RISC-V Core onto SOFA Fabric](#day-5--mapping-the-risc-v-core-onto-sofa-fabric)
  - [RVMYTH Timing on SOFA](#rvmyth-timing-on-sofa)
  - [RVMYTH Utilization on SOFA](#rvmyth-utilization-on-sofa)
  - [RVMYTH Post-Implementation on SOFA](#rvmyth-post-implementation-on-sofa)
- [References](#references)
- [Acknowledgements](#acknowledgements)

---

## What is an FPGA?

An **FPGA (Field Programmable Gate Array)** is a semiconductor device built around an array of programmable logic elements interconnected through a configurable routing fabric. Unlike fixed-function chips, FPGAs can be reprogrammed after manufacturing, making them ideal for rapid prototyping and iterative hardware development.

---

## FPGA vs ASIC — A Quick Comparison

| Feature | FPGA | ASIC |
|---|---|---|
| Full Form | Field Programmable Gate Array | Application-Specific Integrated Circuit |
| Design Output | RTL → Bitstream | RTL → Physical Layout |
| Flexibility | Fully reprogrammable | Permanently fabricated |
| Power Efficiency | Lower — higher overhead for the same function | Higher — purpose-built, minimal overhead |
| Primary Use Case | Prototyping, validation, low-volume production | High-volume end products post-validation |

---

## Day 1 — FPGA Fundamentals and the Vivado Toolchain

### Internal Architecture of an FPGA

An FPGA's internal structure is composed of four primary building blocks:

- **Configurable Logic Blocks (CLBs)** — implement logic functions
- **Programmable Interconnects** — route signals between blocks
- **I/O Cells** — interface the device with the outside world
- **Block RAM / Memory** — dedicated on-chip storage

![FPGA Architecture](images/day1/fpga_architecture.png)

---

### Inside the Configurable Logic Block

The CLB is the fundamental unit of logic in an FPGA. Each CLB contains:

- **Look-Up Table (LUT)** — stores truth tables to implement any combinational logic function
- **Carry and Control Logic** — handles arithmetic chains efficiently
- **Flip-Flops / Latches** — stores state for sequential logic

![CLB Structure](images/day1/clb_structure.png)

---

### Basys 3 Development Board

The target hardware for this workshop is the **Basys 3 Artix-7 FPGA** board by Digilent. Key on-board elements include:

![Basys3 Board](images/day1/basys3_board.png)

| No. | Description | No. | Description |
|-----|-------------|-----|-------------|
| 01 | Power Good LED | 09 | Reset Button |
| 02 | I/O Header | 10 | Configuration Jumper |
| 03 | I/O Header | 11 | USB-UART Interface |
| 04 | 4-digit 7-segment Display | 12 | VGA Connector |
| 05 | Slide Switches (16×) | 13 | USB Port |
| 06 | User LEDs | 14 | External Power Jack |
| 07 | Push Buttons | 15 | Power Select Switch |
| 08 | Programming Done LED | 16 | JTAG Jumper |

---

### 4-bit Counter Design in Vivado

A **4-bit synchronous up-counter** with clock division logic is used as the reference design throughout Day 1. The source clock runs at 100 MHz and is divided down to produce a human-observable counting rate.

```verilog
`timescale 1ns / 1ps
// 4-bit up-counter with clock division
// Source clock: 100 MHz

module counter_clk_div(
    input  clk,
    input  rst,
    output reg [3:0] counter_out
);

    reg        div_clk;
    reg [25:0] delay_count;

    // Clock division block
    always @(posedge clk) begin
        if (rst) begin
            delay_count <= 26'd0;
            div_clk     <= 1'b0;
        end else begin
            if (delay_count == 26'd212) begin
                delay_count <= 26'd0;
                div_clk     <= ~div_clk;
            end else begin
                delay_count <= delay_count + 1;
            end
        end
    end

    // 4-bit counter block
    always @(posedge div_clk) begin
        if (rst)
            counter_out <= 4'b0000;
        else
            counter_out <= counter_out + 1;
    end

endmodule
```

---

### Simulation and Elaboration

**Behavioural simulation** verifies the functional correctness of the counter before synthesis. The waveform confirms proper counting and reset behaviour.

![Counter Simulation Waveform](images/day1/counter_simulation.png)

**Elaboration** resolves the design hierarchy — binding module instances, evaluating parameters, establishing net connectivity, and constructing the pre-synthesis model. The resulting schematic provides a gate-level view of the design prior to technology mapping.

![Elaborated Schematic](images/day1/elaboration_schematic.png)

I/O planning assigns RTL ports to physical FPGA pins.

![I/O Planning](images/day1/io_planning.png)

---

### Synthesis

Synthesis translates the RTL description into a technology-mapped gate-level netlist, optimised against a set of user-defined constraints.

![Synthesis Schematic](images/day1/synthesis_schematic.png)

---

### Design Constraints

Constraints communicate design requirements to the tool — including timing budgets, I/O pin assignments, clock definitions, and input/output delay specifications. They are typically supplied as an XDC file in Vivado.

---

### Bitstream Generation

A bitstream is a binary configuration file that programs the FPGA's internal logic fabric and routing resources. Vivado generates this file as the final step of the implementation flow and it is downloaded directly onto the device.

---

### Timing, Power and Area Reports

After implementation, Vivado produces detailed post-route reports.

**Timing Summary**

![Timing Summary](images/day1/timing_summary.png)

**Device Utilization**

![Device Utilization](images/day1/device_utilization.png)

**Power Analysis**

![Power Analysis](images/day1/power_analysis.png)

---

### Virtual I/O (VIO) Core

The **Virtual I/O (VIO)** IP core enables real-time monitoring and driving of internal FPGA signals without physical I/O pins. Both the number of ports and their widths are parameterizable, making it useful for in-system debug and verification.

---

## Day 2 — OpenFPGA, VPR and the VTR Framework

### OpenFPGA Overview

**OpenFPGA** is the first open-source FPGA IP generator supporting highly customizable, homogeneous FPGA fabrics. It covers the complete EDA chain — from Verilog to bitstream — along with automated self-testing infrastructure. The framework significantly compresses the FPGA development cycle, targeting researchers and chip designers who need agile, open-source flows.

Key capabilities:
- Automated fabric generation from architectural descriptions
- Full Verilog-to-bitstream support
- Integrated self-testing and verification
- Compatible with custom and standard FPGA architectures

![OpenFPGA Framework](images/day2/openfpga_framework.png)

---

### VPR — Versatile Place and Route

**VPR** is an open-source academic CAD tool that handles the packing, placement, and routing stages of the FPGA back-end flow. It accepts an architectural XML description and a technology-mapped circuit (BLIF format), and produces a fully placed-and-routed design along with performance metrics such as critical-path delay and resource utilization.

```bash
$VTR_ROOT/vpr/vpr \
  $VTR_ROOT/vtr_flow/arch/timing/EArch.xml \
  <path-to-blif-file> \
  --route_chan_width 100 \
  --disp on
```

The VPR flow executes four sequential stages:

1. **Packing** — merges logic primitives into complex blocks
2. **Placement** — assigns complex blocks to FPGA grid locations
3. **Routing** — determines signal paths between placed blocks
4. **Analysis** — extracts timing, area, and power metrics from the final implementation

![VPR Flow](images/day2/vpr_flow.png)

---

### VTR — Verilog to Routing

**VTR (Verilog to Routing)** is a complete open-source CAD framework that takes a Verilog RTL description and an FPGA architecture specification, and runs the full compilation flow through to a mapped, placed, and routed result.

```bash
$VTR_ROOT/vtr_flow/scripts/run_vtr_flow.py \
  $VTR_ROOT/doc/src/<verilog-file-path> \
  $VTR_ROOT/vtr_flow/arch/timing/EArch.xml \
  -temp_dir . \
  --route_chan_width 100
```

---

### Running the VTR Flow

The complete VTR flow consists of three main stages:

| Stage | Tool | Function |
|-------|------|----------|
| Elaboration & Synthesis | ODIN II | Converts Verilog RTL to a technology-independent netlist |
| Logic Optimisation & Tech Mapping | ABC | Optimises and maps to LUTs |
| Pack, Place, Route & Timing | VPR | Completes the back-end FPGA flow |

Results from the flow:

**Critical Paths**

![Critical Paths](images/day2/critical_paths.png)

**Net Statistics**

![Nets](images/day2/nets.png)

**Logical Connections**

![Logical Connections](images/day2/logical_connections.png)

**Routing Utilization**

![Routing Utilization](images/day2/routing_utilization.png)

---

### Post-Synthesis Simulation

Post-synthesis simulation in the VTR flow is equivalent to post-implementation simulation in a commercial flow — it verifies that the synthesised netlist matches the original RTL behaviour. The post-synthesis netlist is generated by enabling the following VPR flag:

```bash
--gen_post_synthesis_netlist on
```

The netlist is then simulated in Vivado to confirm functional correctness.

![Post-Synthesis Netlist](images/day2/post_synth_netlist.png)
![Post-Synthesis Simulation](images/day2/post_synth_simulation.png)

---

### Timing Analysis in VTR

Timing analysis requires a constraint file in SDC format, passed to VPR via:

```bash
--sdc_file <path-to-sdc-file>
```

**Setup Timing Report**

![Setup Timing](images/day2/setup_timing.png)

**Hold Timing Report**

![Hold Timing](images/day2/hold_timing.png)

---

### Power Analysis in VTR

VTR includes a built-in power estimation engine. Power analysis is activated using:

```bash
-power -cmos_tech $VTR_ROOT/vtr_flow/tech/PTM_45nm/45nm.xml
```

![VTR Power Report](images/day2/vtr_power.png)

---

## Day 3 — RISC-V Core Implementation on Vivado

The processor used in this section is **RVMYTH**, a 4-stage pipelined RISC-V core. The design is originally written in **TL-Verilog** (a transaction-level HDL abstraction) and compiled down to standard Verilog for the FPGA flow. A complete RTL-to-bitstream implementation is carried out on the Basys 3 board.

### RTL to Synthesis

The RVMYTH RTL is structured around the following key modules:

- Instruction Memory
- Data Memory
- ALU
- I/O Interface

The instruction memory is loaded with a simple test program that computes the sum of integers from 1 to 9. The accumulating result is observable on the simulation waveforms.

![RVMYTH Simulation](images/day3/rvmyth_simulation.png)

Pin mapping is performed in the elaboration stage, assigning FPGA physical I/Os to the core's ports.

![RVMYTH Elaboration](images/day3/rvmyth_elaboration.png)

The design is synthesised targeting the Basys 3 Artix-7 device. Post-synthesis schematic and applied constraints are shown below.

![RVMYTH Synthesis Schematic](images/day3/rvmyth_synth_schematic.png)
![RVMYTH Constraints](images/day3/rvmyth_constraints.png)

---

### Synthesis to Bitstream

During implementation, the synthesised netlist is translated into FPGA-native primitives — LUTs, MUXes, flip-flops — and mapped into CLBs. These are then placed and routed to meet timing constraints.

**Implemented Design Fragment**

![RVMYTH Implemented Design](images/day3/rvmyth_implementation.png)

**Timing Summary**

![RVMYTH Timing](images/day3/rvmyth_timing.png)

**Device Utilization Summary**

![RVMYTH Utilization](images/day3/rvmyth_utilization.png)

**Power Report**

![RVMYTH Power](images/day3/rvmyth_power.png)

---

## Day 4 — SOFA FPGA Fabric Introduction

**SOFA (Skywater Open-Source FPGAs)** is a family of open-source FPGA IP cores fabricated using the **Skywater 130nm open PDK** and generated through the **OpenFPGA** framework.

The specific fabric used in this workshop is **FPGA1212_QLSOFA_HD_PNR**, with the following specifications:

| Parameter | Value |
|-----------|-------|
| Max Operating Frequency | 50 MHz |
| LUT Count | 1152 |
| Flip-Flop Count | 2304 |
| Soft Adders | 1152 |

---

### Counter Area on SOFA

Resource utilization of the 4-bit counter mapped onto the SOFA fabric.

![SOFA Counter Area](images/day4/sofa_counter_area.png)

---

### Counter Timing on SOFA

Setup and hold timing analysis for the counter on SOFA.

![SOFA Counter Setup Timing](images/day4/sofa_counter_setup.png)
![SOFA Counter Hold Timing](images/day4/sofa_counter_hold.png)

---

### Counter Post-Implementation on SOFA

Functional verification of the counter after implementation on the SOFA fabric.

![SOFA Counter Post-Implementation](images/day4/sofa_counter_postimpl.png)

---

### Counter Power on SOFA

Power dissipation breakdown for the counter running on SOFA fabric.

![SOFA Counter Power](images/day4/sofa_counter_power.png)

---

## Day 5 — Mapping the RISC-V Core onto SOFA Fabric

The **RVMYTH** RISC-V core is now targeted to the custom SOFA FPGA fabric. The complete OpenFPGA + VTR flow is executed, generating implementation logs and analysis reports for the full processor design.

---

### RVMYTH Timing on SOFA

**Setup Timing Report**

![SOFA RVMYTH Setup Timing](images/day5/sofa_rvmyth_setup.png)

**Hold Timing Report**

![SOFA RVMYTH Hold Timing](images/day5/sofa_rvmyth_hold.png)

---

### RVMYTH Utilization on SOFA

Resource utilization of the RVMYTH core on the SOFA FPGA fabric.

![SOFA RVMYTH Utilization 1](images/day5/sofa_rvmyth_util_1.png)
![SOFA RVMYTH Utilization 2](images/day5/sofa_rvmyth_util_2.png)

---

### RVMYTH Post-Implementation on SOFA

Post-implementation simulation confirms correct functional operation of the RVMYTH core on the SOFA fabric.

![SOFA RVMYTH Post-Implementation](images/day5/sofa_rvmyth_postimpl.png)

---

## References

- [VLSI System Design](https://www.vlsisystemdesign.com/ip/)
- [RISC-V Core Reference](https://github.com/shivanishah269/risc-v-core)
- [RVMYTH — 4-Stage RISC-V Core](https://github.com/ShonTaware/RISC-V_Core_4_Stage)
- [SOFA — Skywater Open-Source FPGAs](https://github.com/lnis-uofu/SOFA)
- [OpenFPGA Documentation](https://openfpga.readthedocs.io/en/master/)
- [VPR Documentation](https://docs.verilogtorouting.org/en/latest/vpr/)
- [VTR Documentation](https://docs.verilogtorouting.org/en/latest/vtr/)

---

## Acknowledgements

Special thanks to the workshop instructors and the open-source EDA community — particularly the teams behind OpenFPGA, VTR/VPR, and the Skywater PDK — for making accessible, high-quality FPGA tooling available to students and researchers worldwide.
