# Physical Verification using SKY130

Documentation of my work from the **Physical Verification** course using the SkyWater 130nm open-source PDK, as part of the screening for the **Women in VLSI** program by Marvell Technology, SFAL, ESSCI, and VSD.

## Table of Contents

- [Day 1 - Introduction to SkyWater SKY130](#day-1---introduction-to-skywater-sky130)
  - [SK1 - Introduction to SkyWater PDKs and Open-Source EDA Tools](#sk1---introduction-to-skywater-pdks-and-open-source-eda-tools)
    - [L1 - Introduction to SkyWater PDK](#l1---introduction-to-skywater-pdk)
    - [L2 - Open-Source EDA Tools](#l2---open-source-eda-tools)
    - [L3 - Understanding SkyWater PDK - Layers](#l3---understanding-skywater-pdk---layers)
    - [L4 - Understanding SkyWater PDK - Devices](#l4---understanding-skywater-pdk---devices)
    - [L5 - Understanding SkyWater PDK Libraries](#l5---understanding-skywater-pdk-libraries)
    - [L6 - Open-Source Tools and Flows](#l6---open-source-tools-and-flows)
  - [SK2 - Tool Installation and Basic DRC/LVS Design Flow](#sk2---tool-installation-and-basic-drclvs-design-flow)
    - [L1 - Check Tool Installations](#l1---check-tool-installations)
    - [L2 - Creating SKY130 Device Layout in Magic](#l2---creating-sky130-device-layout-in-magic)
    - [L3 - Creating Simple Schematic in Xschem](#l3---creating-simple-schematic-in-xschem)
    - [L4 - Creating Symbol and Exporting Schematic in Xschem](#l4---creating-symbol-and-exporting-schematic-in-xschem)
    - [L5 - Importing Schematic to Layout and Inverter Layout Steps](#l5---importing-schematic-to-layout-and-inverter-layout-steps)
    - [L6 - Final DRC/LVS Checks](#l6---final-drclvs-checks)
- [References](#references)
- [Acknowledgements](#acknowledgements)

---

## Day 1 - Introduction to SkyWater SKY130

### SK1 - Introduction to SkyWater PDKs and Open-Source EDA Tools

#### L1 - Introduction to SkyWater PDK

The **SkyWater SKY130** is the first fully open-source PDK, a collaboration between **Google** and **SkyWater Technology Foundry**. The "130" refers to the 130nm feature size (minimum transistor length). However, **the actual minimum-length device in SKY130 is 150nm, not 130nm** — this is due to obscure process reasons.

A **PDK (Process Design Kit)** is the interface between the chip designer and the foundry. It contains process design rules, device models, digital standard cell libraries, I/O libraries, and primitive device libraries.

| Resource | Link |
|----------|------|
| Documentation | https://skywater-pdk--136.org.readthedocs.build |
| PDK Library & Files | https://github.com/google/skywater-pdk |
| Community (Slack) | https://join.skywater.tools |

![SkyWater PDK](images/Day1_SK1/skywater_pdk_overview.png)

#### L2 - Open-Source EDA Tools

The following open-source tools are compatible with the SKY130 PDK:

- **Magic** — Layout editor, DRC engine, and parasitic extraction. Written by John Ousterhout at UC Berkeley, maintained by Tim Edwards.
- **Netgen** — LVS (Layout vs. Schematic) comparator. Compares two SPICE netlists for structural equivalence.
- **Xschem** — Schematic capture tool. Generates SPICE netlists for simulation and LVS.
- **ngspice** — Open-source SPICE simulator for analog/mixed-signal verification.
- **OpenLANE** — Automated RTL-to-GDSII flow integrating Yosys, OpenROAD, Magic, and Netgen.
- **KLayout** — Alternative GDS viewer with DRC scripting and XOR comparison.

![Open-Source EDA Tools](images/Day1_SK1/opensource_eda_tools.png)

#### L3 - Understanding SkyWater PDK - Layers

The SKY130 process has the following layer stack:

- **diff** — Active diffusion area where transistors are formed
- **poly** — Polysilicon gate layer. Where poly crosses diffusion, a transistor channel is formed
- **li1 (Local Interconnect)** — Made of **TiN (Titanium Nitride)**. Used only for very short, device-level connections due to high sheet resistance
- **met1 to met5** — Five metal routing layers. met1 is thinnest, met5 is thickest (used for power distribution)
- **mcon** — Contact connecting li1 to met1
- **via1 to via4** — Vias connecting adjacent metal layers
- **licon** — Contact connecting diffusion/poly to li1
- **nwell / dnwell** — N-well for PMOS. **Deep N-well** provides noise isolation between analog and digital circuits
- **NPC (Nitride Poly Cut)** — An automatically generated layer. Magic generates it from poly and contact positions; the designer does not draw it manually
- **RDL (Redistribution Layer)** — Handled separately on the top metal

![SKY130 Layer Stack](images/Day1_SK1/sky130_layers.png)

#### L4 - Understanding SkyWater PDK - Devices

**MOSFETs:**

| Device | Type | Description |
|--------|------|-------------|
| `sky130_fd_pr__nfet_01v8` | NMOS 1.8V | Standard threshold, general purpose |
| `sky130_fd_pr__pfet_01v8` | PMOS 1.8V | Standard threshold, general purpose |
| `sky130_fd_pr__nfet_01v8_lvt` | NMOS LVT | Low threshold, high speed |
| `sky130_fd_pr__pfet_01v8_hvt` | PMOS HVT | High threshold, low leakage |
| `sky130_fd_pr__nfet_g5v0d10v5` | NMOS 5V | Thick oxide, I/O |
| `sky130_fd_pr__pfet_g5v0d10v5` | PMOS 5V | Thick oxide, I/O |

**Passive Devices:** Poly resistors, high-res poly resistors, MiM capacitors, varactors, P-N junction diodes.

**Bipolar Devices:** PNP and NPN BJTs for bandgap references and analog circuits.

All devices are **parameterized cells (pcells)** — the designer sets W and L, and the layout is generated automatically. SPICE models are BSIM4 with corner models (tt, ff, ss, sf, fs).

![SKY130 Devices](images/Day1_SK1/sky130_devices.png)

#### L5 - Understanding SkyWater PDK Libraries

The PDK organizes libraries under `/usr/local/share/pdk/sky130A/`:

- **libs.tech/** — Technology files for each tool (Magic, Xschem, ngspice, Netgen, OpenLANE)
- **libs.ref/** — Reference libraries:
  - `sky130_fd_sc_hd` — High-density standard cells (most commonly used)
  - `sky130_fd_sc_hs` — High-speed standard cells
  - `sky130_fd_sc_ms` — Medium-speed standard cells
  - `sky130_fd_sc_ls` — Low-speed standard cells
  - `sky130_fd_sc_lp` — Low-power standard cells
  - `sky130_fd_io` — I/O pad cells
  - `sky130_fd_pr` — Primitive devices (transistors, R, C)

**Naming convention:** `sky130` (process) + `fd` (foundry) + `sc` (standard cell) + `hd` (high-density)

Each cell has multiple views: **GDS** (physical layout), **LEF** (abstract for P&R), **LIB** (timing models), **SPICE** (transistor-level netlist).

![SKY130 Libraries](images/Day1_SK1/sky130_libraries.png)

#### L6 - Open-Source Tools and Flows

Physical verification is part of the **signoff stage** in the RTL-to-GDSII flow:

1. **Synthesis** (Yosys/ABC) → Gate-level netlist
2. **Floorplan + Placement** (OpenROAD) → Physical placement of cells
3. **CTS** (TritonCTS) → Clock tree synthesis
4. **Routing** (FastRoute + TritonRoute) → Metal connections
5. **Signoff** → **DRC** (Magic) + **LVS** (Netgen)
6. **Tapeout** → Final GDSII file

**DRC** ensures the layout meets all foundry manufacturing rules. **LVS** ensures the layout matches the intended schematic. No foundry accepts a design without clean DRC and LVS.

**OpenLANE** automates this entire pipeline. For custom analog design, each tool is used individually — which is what this course focuses on.

![RTL to GDSII Flow](images/Day1_SK1/rtl_to_gdsii_flow.png)

---

### SK2 - Tool Installation and Basic DRC/LVS Design Flow

#### L1 - Check Tool Installations

The tools are installed in the following order. **open_pdks must be installed last** as it configures technology files based on tool install locations.

**Magic:**
```bash
git clone git://opencircuitdesign.com/magic
cd magic
./configure
make
sudo make install
```

**Netgen:**
```bash
git clone git://opencircuitdesign.com/netgen
cd netgen
./configure
make
sudo make install
```

**Xschem:**
```bash
git clone https://github.com/StefanSchippers/xschem.git xschem_git
cd xschem_git
./configure
make
sudo make install
```

**ngspice:**
```bash
tar -zxvf ngspice-37.tar.gz
cd ngspice-37
mkdir release
cd release
../configure --with-x --with-readline=yes --disable-debug
make
sudo make install
```

**open_pdks:**
```bash
git clone git://opencircuitdesign.com/open_pdks
cd open_pdks
./configure --enable-sky130-pdk
make
sudo make install
```

Setting up the working directory and copying config files:

```bash
mkdir inverter_lab && cd inverter_lab
mkdir mag xschem netgen

cd xschem
cp /usr/local/share/pdk/sky130A/libs.tech/xschem/xschemrc .
cp /usr/local/share/pdk/sky130A/libs.tech/ngspice/spinit .spiceinit

cd ../mag
cp /usr/local/share/pdk/sky130A/libs.tech/magic/sky130A.magicrc .magicrc

cd ../netgen
cp /usr/local/share/pdk/sky130A/libs.tech/netgen/sky130A_setup.tcl .
```

Verifying each tool launches correctly:

![Magic Launch](images/Day1_SK2/magic_launch.png)

![Xschem Launch](images/Day1_SK2/xschem_launch.png)

![Netgen Launch](images/Day1_SK2/netgen_launch.png)

![ngspice Launch](images/Day1_SK2/ngspice_launch.png)

---

#### L2 - Creating SKY130 Device Layout in Magic

Launched Magic from `mag/` directory with `magic -d XR` and explored the SKY130 standard cell layout by loading an existing inverter cell from the PDK:

```tcl
% gds read /usr/local/share/pdk/sky130A/libs.ref/sky130_fd_sc_hd/gds/sky130_fd_sc_hd.gds
% load sky130_fd_sc_hd__inv_1
```

We can observe the PMOS (inside N-well) and NMOS transistors, power rails (VPWR, VGND) on met1, and local interconnect (li1) used for internal routing.

![Standard Cell in Magic](images/Day1_SK2/magic_std_cell_layout.png)

![Device Layout in Magic](images/Day1_SK2/magic_device_layout.png)

---

#### L3 - Creating Simple Schematic in Xschem

Launched Xschem from the `xschem/` directory and built a CMOS inverter schematic:

- Placed `sky130_fd_pr__nfet_01v8` (W=1, L=0.15) and `sky130_fd_pr__pfet_01v8` (W=2, L=0.15)
- Connected gates together as input **A**
- Connected drains together as output **Y**
- PMOS source → VDD, NMOS source → GND
- Body connections: PMOS body → VDD, NMOS body → GND

![Inverter Schematic](images/Day1_SK2/xschem_inverter_schematic.png)

---

#### L4 - Creating Symbol and Exporting Schematic in Xschem

Created an inverter symbol from the schematic using **Symbol → Make symbol from schematic**.

Built a testbench `inverter_tb.sch`:
- Placed the inverter symbol
- Added VDD source (1.8V) and input pulse source
- Added simulation commands:

```spice
.lib /usr/local/share/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.tran 0.01n 10n
.save all
```

![Inverter Symbol](images/Day1_SK2/xschem_inverter_symbol.png)

![Inverter Testbench](images/Day1_SK2/xschem_inverter_testbench.png)

---

**Pre-Layout Simulation (ngspice):**

Ran the transient simulation from the testbench. The inverter behaves as expected — output Y is the complement of input A.

![Pre-Layout Simulation Result](images/Day1_SK2/ngspice_inverter_simulation.png)

---

#### L5 - Importing Schematic to Layout and Inverter Layout Steps

Launched Magic from `mag/` directory with `magic -d XR` and built the inverter layout:

- Placed NMOS and PMOS devices matching the schematic dimensions
- Routed using local interconnect (li1) and metal1
- Added power rails (VPWR, VGND) on met1
- Added substrate and well taps for body connections
- Added port labels: A, Y, VPWR, VGND

![Inverter Layout](images/Day1_SK2/magic_inverter_layout.png)

![Inverter Layout with Labels](images/Day1_SK2/magic_inverter_layout_labels.png)

---

#### L6 - Final DRC/LVS Checks

**DRC Check:**

Ran DRC in Magic:

```tcl
% drc check
% drc listall why
```

All DRC violations were resolved. Layout is DRC clean.

![DRC Clean](images/Day1_SK2/magic_drc_clean.png)

**Layout Netlist Extraction:**

Extracted the SPICE netlist from the layout for LVS:

```tcl
% extract do local
% extract all
% ext2spice lvs
% ext2spice
```

![Extraction Output](images/Day1_SK2/magic_extraction.png)

**LVS with Netgen:**

Compared the schematic netlist (from Xschem) with the layout netlist (from Magic):

```bash
netgen -batch lvs "../mag/inverter.spice inverter" "../xschem/inverter.spice inverter"
```

Result: **Circuits match uniquely.**

![LVS Result - Match](images/Day1_SK2/netgen_lvs_match.png)

**Post-Layout Simulation:**

After confirming LVS clean, ran a post-layout simulation using the extracted netlist to verify that layout parasitics do not degrade circuit performance.

![Post-Layout Simulation](images/Day1_SK2/postlayout_simulation.png)

---

## References

- [SkyWater SKY130 PDK](https://github.com/google/skywater-pdk)
- [Open_PDKs](https://github.com/RTimothyEdwards/open_pdks)
- [Magic VLSI](http://opencircuitdesign.com/magic/)
- [Netgen LVS](http://opencircuitdesign.com/netgen/)
- [Xschem](https://xschem.sourceforge.io/stefan/index.html)
- [ngspice](http://ngspice.sourceforge.net/)
- [VSD Physical Verification](https://www.vlsisystemdesign.com/)

## Acknowledgements

- **Tim Edwards** — Maintainer of Magic, Netgen, and open_pdks
- **Kunal Ghosh** — Co-founder, VSD
- **Marvell Technology, SFAL, ESSCI** — Women in VLSI initiative
- **SkyWater Technology & Google** — Open-source PDK program
