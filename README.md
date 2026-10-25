# 4-bit Array Multiplier: RTL to GDSII

| Team member | Roll number |
|---|---|
| Vedant Zope | 2024102003 |
| Kavya Pandey | 2024102001 |

## Repository structure

Required submission structure from project document Section 5 and the provided screenshot:

```text
<team-repository>/
|-- README.md
|-- rtl/
|   |-- full_adder.v
|   `-- mult_array.v
|-- tb/
|   `-- tb_mult.v
|-- synth/
|   |-- synth.tcl
|   |-- mult_array_netlist.v       
|   `-- mult_array_stat.rpt        
|-- sta/
|   |-- sta.tcl
|   `-- mult_array_timing.rpt     
|-- pnr/
|   |-- config.json
|   |-- metrics.json             
|   `-- mult_array.gds          
`-- report/
    `-- report.pdf
```


## Commands in document order

The sequence below follows project document Sections 4 and 6–9. It records the prescribed procedure; a complete original execution history was not supplied. Use the repository root (the folder containing this README) for Parts A–C, and `pnr/` for Part D.

### 1. Environment checks

The document specifies the IIC-OSIC-TOOLS Docker environment with SKY130A. Follow its Quick launch instructions, open `http://localhost`, and use the container terminal. Host `~/eda/designs` maps to `/foss/designs`; keep the repository there. The document provides no exact container-launch command.

Inside the container, check the tools and selected PDK:

```bash
iverilog -V | head -1
yosys -V
sta -version
echo $PDK_ROOT $PDK
```

If SKY130A is not selected, run the document's conditional command:

```bash
sak-pdk sky130A
```

### 2. Part A: RTL simulation

Working directory: repository root. Compile, simulate, then inspect the waveform:

```bash
iverilog -g2012 -o sim tb/tb_mult.v rtl/full_adder.v rtl/mult_array.v
vvp sim
gtkwave mult.vcd
```

For the required error-detection check, temporarily change the sum expression in `rtl/full_adder.v` to `a ^ b` and repeat the compile and simulation commands. Then restore `a ^ b ^ cin` and repeat those commands before synthesis. The document gives no separate command for editing or restoring the file.

**Discrepancy:** `tb/tb_mult.v` uses `#5` for each half-cycle without an explicit timescale, so the document's required 10 ns clock period is not established by that file alone.

### 3. Part B: synthesis and gate-level simulation

Working directory: repository root. Run the synthesis script:

```bash
yosys -c synth/synth.tcl
```

The commands inside `synth/synth.tcl` run in this order and generate the two files under `synth/`:

```tcl
set lib $::env(PDK_ROOT)/sky130A/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
yosys read_verilog rtl/full_adder.v rtl/mult_array.v
yosys synth -top mult_array -flatten
yosys dfflibmap -liberty $lib
yosys abc -liberty $lib
yosys opt_clean -purge
yosys tee -o synth/mult_array_stat.rpt stat -liberty $lib
yosys write_verilog -noattr synth/mult_array_netlist.v
```

Next, compile the generated netlist with the PDK models and simulate it:

```bash
CELLS=$PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog
iverilog -g2012 -DFUNCTIONAL -DUNIT_DELAY=#1 -o gls \
  tb/tb_mult.v $CELLS/primitives.v $CELLS/sky130_fd_sc_hd.v synth/mult_array_netlist.v
vvp gls
```

### 4. Part C: static timing analysis

Working directory: repository root. Section 8 requires a first run with a 10 ns clock period. The required script sequence for that run is:

```tcl
set lib $::env(PDK_ROOT)/sky130A/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_liberty $lib
read_verilog synth/mult_array_netlist.v
link_design mult_array
create_clock -name clk -period 10 [get_ports clk]
set_input_delay 1.0 -clock clk [delete_from_list [all_inputs] [get_ports clk]]
set_output_delay 1.0 -clock clk [all_outputs]
report_worst_slack -max
report_checks -path_delay max -digits 3 > sta/mult_array_timing.rpt
```

**Discrepancy:** the saved `sta/sta.tcl` uses `-period 2.983`. Set its period to 10 before the first required run:

```bash
sta sta/sta.tcl
```

Use the reported worst setup slack to calculate `Tmin = 10 − WNS`. Set the clock period in `sta/sta.tcl` to `Tmin` rounded to 0.01 ns, then run again:

```bash
sta sta/sta.tcl
```

The original 10 ns run log is unavailable; the saved script does not establish that both required runs were performed.

### 5. Part D: physical design and layout viewing

Start in the repository root. Set `CLOCK_PERIOD` in `pnr/config.json` to `1.2 × Tmin`, rounded up to the next 0.5 ns, as Section 9 requires. Then enter `pnr/` and run the flow:

```bash
cd pnr
librelane config.json
```

The document permits `python3 -m librelane config.json` as an alternative invocation from the same directory. If timing fails, increase `CLOCK_PERIOD` by 0.5 ns and rerun.

Required attempt record: the supplied run notes list clock periods of **4.0, 4.5, 5.0, 5.5, 6.0, and 6.5 ns**, in that order. Individual run commands and logs are unavailable; the saved configuration contains the last period.

Copy `final/metrics.json` and the GDS from the successful run under `pnr/runs/<date-time>/` to `pnr/metrics.json` and `pnr/mult_array.gds`. The original run directory, exact GDS source filename, and copy commands are unavailable, so no copy command is assumed. Do not commit `pnr/runs/`.

Still inside `pnr/`, open the required GDS file for the whole-die and zoomed layout views:

```bash
klayout mult_array.gds
```

This resolves the document's `klayout <file>.gds` placeholder to the required submission filename.

The results, screenshots, discussion, and answers are in [report/report.pdf](report/report.pdf).
