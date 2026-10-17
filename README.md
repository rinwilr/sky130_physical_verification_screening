# Physical Verification using SkyWater SKY130 PDK

A documentation of my learnings from the **Physical Verification** course using the SkyWater 130nm open-source PDK, as part of the screening for the **Women in VLSI** program conducted by Marvell Technology, SFAL, ESSCI, and VSD.

This repository covers the fundamentals of physical verification — DRC (Design Rule Checking) and LVS (Layout vs. Schematic) — using open-source EDA tools like **Magic**, **Netgen**, **Xschem**, and **ngspice**, all built around the **SKY130** process node.

---

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

The **SkyWater SKY130** is the first foundry-provided, fully open-source Process Design Kit (PDK), released through a collaboration between **Google** and **SkyWater Technology**. The "130" in SKY130 refers to the **130nm feature size** — this is the minimum transistor length the process was originally designed for. However, there is an important caveat: **for obscure process reasons, the actual minimum-length device in SKY130 is 150nm, not 130nm.**

The PDK is a public repository, making real silicon fabrication accessible to hobbyists, researchers, and small startups for the first time.

**Key Resources:**

| Resource | Link |
|----------|------|
| PDK Documentation | https://skywater-pdk--136.org.readthedocs.build |
| PDK Library & Files | https://github.com/google/skywater-pdk |
| Community (Slack) | https://join.skywater.tools |

**What is a PDK?**

A PDK (Process Design Kit) is a collection of files that models a semiconductor fabrication process for EDA tools. It serves as the interface between the chip designer and the foundry. A PDK contains:

- **Process design rules** — the geometric constraints (minimum widths, spacings, enclosures) that the layout must satisfy for successful manufacturing. These are enforced through DRC.
- **Device models** — SPICE-level transistor models (such as BSIM3/BSIM4 parameter files) that simulators use to predict circuit behavior.
- **Digital standard cell libraries** — pre-designed, pre-characterized logic gates (AND, OR, flip-flops, etc.) with layout, timing models (.lib), abstract views (.lef), and netlists.
- **I/O libraries** — pad cells for connecting the chip to the outside world.
- **Primitive device libraries** — parameterized cells (pcells) for transistors, resistors, capacitors, and diodes.

The SkyWater 130nm node is a hybrid technology featuring:
- 1 local interconnect layer (li1)
- 5 metal layers (met1 through met5)
- Poly and diffusion layers for transistor construction
- High-resistance poly resistors
- MiM (Metal-insulator-Metal) capacitors
- PNP and NPN BJT devices

The PDK is hosted on GitHub and maintained by the open-source community. All design rules, device models, and cell libraries are publicly available.

> **Key takeaway:** SKY130 democratizes chip design. Before this, accessing a foundry PDK required NDAs and expensive licensing. Now anyone can design a chip from RTL to GDSII using entirely free and open tools.

![SkyWater PDK Overview](images/Day1_SK1/skywater_pdk_overview.png)

---

#### L2 - Open-Source EDA Tools

For physical verification and the broader RTL-to-GDSII flow, we use a suite of open-source tools. Each tool has a specific role:

| Tool | Purpose | Key Capability |
|------|---------|----------------|
| **Magic** | Layout editor & DRC engine | Interactive layout editing, DRC checking, parasitic extraction, GDS read/write |
| **Netgen** | LVS engine | Compares schematic and layout netlists for functional equivalence |
| **Xschem** | Schematic capture | Draws circuit schematics, generates SPICE netlists |
| **ngspice** | Circuit simulator | Runs SPICE simulations for analog/mixed-signal verification |
| **OpenLANE** | Automated RTL-to-GDSII flow | Integrates synthesis, PnR, DRC, LVS into one automated pipeline |
| **KLayout** | Alternative layout viewer | GDS viewing, DRC scripting, XOR comparison |

**Magic** deserves special mention — originally written by John Ousterhout at UC Berkeley in the 1980s, it has been continuously maintained by **Tim Edwards** at Open Circuit Design. It natively understands the SKY130 technology files and can perform:
- Interactive DRC with detailed error reporting
- Parasitic extraction (capacitance and resistance)
- GDS/LEF/DEF import and export
- Netlist generation for LVS

**Netgen** is a structural netlist comparator. It takes two SPICE netlists — one from the schematic (via Xschem) and one from the layout (via Magic extraction) — and verifies that they represent the same circuit topology. It can handle hierarchical designs, flatten them when needed, and report mismatches down to the individual net or device level.

![Open-Source EDA Ecosystem](images/Day1_SK1/opensource_eda_tools.png)

---

#### L3 - Understanding SkyWater PDK - Layers

The SKY130 process uses a specific set of physical layers, each identified by a name and GDS layer/datatype number. Understanding these layers is essential for layout design and DRC.

**Backend Metal Stack:**

```
         ┌─────────────────────────────┐
         │         met5 (thick)        │  ← Top redistribution layer
         ├─────────────────────────────┤
         │           via4              │
         ├─────────────────────────────┤
         │           met4              │
         ├─────────────────────────────┤
         │           via3              │
         ├─────────────────────────────┤
         │           met3              │
         ├─────────────────────────────┤
         │           via2              │
         ├─────────────────────────────┤
         │           met2              │
         ├─────────────────────────────┤
         │           via1              │
         ├─────────────────────────────┤
         │           met1              │
         ├─────────────────────────────┤
         │          mcon               │
         ├─────────────────────────────┤
         │     li1 (local interconnect)│  ← Lowest routing layer
         ├─────────────────────────────┤
         │          licon              │
         ├─────────────────────────────┤
         │    nsdm/psdm (implants)     │
         ├─────────────────────────────┤
         │    diff / poly (frontend)   │  ← Transistor level
         └─────────────────────────────┘
```

**Key layer details:**

- **diff** — Active diffusion area where transistors are formed. N-type (ndiff) and P-type (pdiff) are distinguished by implant masks.
- **poly** — Polysilicon gate layer. Where it crosses diffusion, a transistor channel is formed.
- **li1 (local interconnect)** — A thin routing layer **made of TiN (Titanium Nitride)** sitting between the transistors and metal1. It has high sheet resistance and is meant only for very short, device-level connections. It is *not* a true metal layer and should be avoided for long routing.
- **met1 through met5** — The five copper metal routing layers. met1 is the thinnest; met5 is the thickest and used for power distribution and the **RDL (Redistribution Layer)** which is handled separately.
- **mcon** — Contact connecting li1 to met1.
- **via1 to via4** — Vias connecting adjacent metal layers.
- **nwell / dnwell** — N-well is used for PMOS transistors. **Deep N-well (dnwell)** provides noise isolation between sensitive analog circuits and the noisy digital substrate.
- **NPC (Nitride Poly Cut)** — An automatically generated layer. Magic creates it from poly and contact positions — the designer does not draw it manually.

Each layer has specific design rules governing minimum width, minimum spacing, minimum enclosure (how much a via must overlap the metal around it), and minimum area.

![SKY130 Layer Stack](images/Day1_SK1/sky130_layers.png)

---

#### L4 - Understanding SkyWater PDK - Devices

The SKY130 PDK provides several device types, each modeled and characterized for circuit simulation:

**MOSFET Transistors:**

| Device | Type | Vth | Use Case |
|--------|------|-----|----------|
| `sky130_fd_pr__nfet_01v8` | NMOS | Standard Vt | General digital/analog |
| `sky130_fd_pr__pfet_01v8` | PMOS | Standard Vt | General digital/analog |
| `sky130_fd_pr__nfet_01v8_lvt` | NMOS | Low Vt | High-speed, more leakage |
| `sky130_fd_pr__pfet_01v8_hvt` | PMOS | High Vt | Low-leakage, slower |
| `sky130_fd_pr__nfet_g5v0d10v5` | NMOS 5V | Thick oxide | I/O, high voltage |
| `sky130_fd_pr__pfet_g5v0d10v5` | PMOS 5V | Thick oxide | I/O, high voltage |

**Passive Devices:**

- **Resistors:** Diffusion resistors, poly resistors (`sky130_fd_pr__res_generic_po`), and high-resistance poly resistors (`sky130_fd_pr__res_high_po`)
- **Capacitors:** MiM capacitors (`sky130_fd_pr__cap_mim_m3_1`, `sky130_fd_pr__cap_mim_m3_2`) and varactors
- **Diodes:** P-N junction diodes for ESD protection and antenna rule fixes

**Bipolar Transistors:**

- PNP (`sky130_fd_pr__pnp_05v5_W3p40L3p40`) and NPN BJTs are available for bandgap references and analog circuits.

Each device is a **parameterized cell (pcell)** — the designer specifies width (W) and length (L), and the layout generator produces the geometry automatically. The SPICE models are BSIM4 for MOSFETs and include corner models (tt, ff, ss, sf, fs) for process variation analysis.

![SKY130 Device Types](images/Day1_SK1/sky130_devices.png)

---

#### L5 - Understanding SkyWater PDK Libraries

The SKY130 PDK organizes its content into a directory hierarchy under the installation path (typically `/usr/local/share/pdk/sky130A/`):

```
sky130A/
├── libs.tech/          ← Technology files for each tool
│   ├── magic/
│   │   ├── sky130A.magicrc      ← Magic startup config
│   │   └── sky130A.tech         ← Magic technology file (layer defs + DRC rules)
│   ├── xschem/
│   │   └── xschemrc             ← Xschem startup config with SKY130 libs
│   ├── ngspice/
│   │   └── spinit                ← ngspice initialization
│   ├── netgen/
│   │   └── sky130A_setup.tcl     ← Netgen device matching rules
│   └── openlane/                 ← OpenLANE configuration
│
├── libs.ref/           ← Reference libraries (standard cells, I/O, primitives)
│   ├── sky130_fd_sc_hd/          ← High-density digital standard cells
│   │   ├── gds/                  ← GDSII layout files
│   │   ├── lef/                  ← Abstract layout (for place & route)
│   │   ├── lib/                  ← Timing/power characterization
│   │   ├── spice/                ← Transistor-level netlists
│   │   ├── mag/                  ← Magic layout files
│   │   ├── verilog/              ← Behavioral models
│   │   └── cdl/                  ← Circuit Description Language netlists
│   ├── sky130_fd_sc_hs/          ← High-speed standard cells
│   ├── sky130_fd_sc_ms/          ← Medium-speed standard cells
│   ├── sky130_fd_sc_ls/          ← Low-speed standard cells
│   ├── sky130_fd_sc_lp/          ← Low-power standard cells
│   ├── sky130_fd_io/             ← I/O pad cells
│   └── sky130_fd_pr/             ← Primitive devices (transistors, R, C)
```

**Library naming convention:** `sky130_fd_sc_hd`

- `sky130` — process name
- `fd` — foundry (SkyWater)
- `sc` — standard cell (or `pr` for primitives, `io` for I/O)
- `hd` — variant (high-density)

The **high-density (hd)** library is the most commonly used for digital design. It contains hundreds of cells — from basic inverters and NAND gates to complex flip-flops and multiplexers — each available in multiple drive strengths (e.g., `sky130_fd_sc_hd__inv_1`, `sky130_fd_sc_hd__inv_2`, `sky130_fd_sc_hd__inv_4`).

Each cell has views in multiple formats so that different EDA tools can consume them:
- **GDS** — the full physical layout, used by Magic and KLayout
- **LEF** — abstract view with pin locations and blockage, used by placement/routing tools
- **LIB** — timing and power models in Liberty format, used by synthesis and STA tools
- **SPICE** — transistor-level netlist, used by Netgen for LVS and ngspice for simulation

![SKY130 Library Hierarchy](images/Day1_SK1/sky130_libraries.png)

---

#### L6 - Open-Source Tools and Flows

The complete open-source RTL-to-GDSII flow looks like this:

```
  RTL (Verilog)
       │
       ▼
  ┌──────────┐
  │  Yosys   │  Synthesis → Gate-level netlist
  └────┬─────┘
       │
       ▼
  ┌──────────────┐
  │  OpenROAD    │  Floorplanning → Placement → CTS → Routing
  └────┬─────────┘
       │
       ▼
  ┌──────────┐
  │  Magic   │  GDS generation, DRC, Parasitic Extraction
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  Netgen  │  LVS (Layout vs. Schematic)
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  GDSII   │  Tapeout-ready file
  └──────────┘
```

**Physical Verification fits in the signoff stage.** Before a design goes to the foundry for fabrication, two checks must pass:

1. **DRC (Design Rule Check)** — Ensures every geometric shape in the layout satisfies the foundry's manufacturing rules. A DRC violation means the chip may not fabricate correctly.

2. **LVS (Layout vs. Schematic)** — Ensures the layout implements exactly the circuit the designer intended. The layout netlist (extracted from Magic) must structurally match the schematic netlist (from Xschem).

These are not optional — no foundry will accept a design without clean DRC and LVS results.

**OpenLANE** wraps all these tools into one automated pipeline. For a fully custom analog design, however, you use each tool individually — which is what this course focuses on.

![RTL to GDSII Flow](images/Day1_SK1/rtl_to_gdsii_flow.png)

---

### SK2 - Tool Installation and Basic DRC/LVS Design Flow

#### L1 - Check Tool Installations

All tools are installed on the lab environment. The installation order matters — **open_pdks must be installed last** because it configures the technology files for each tool based on their install locations.

**Installation steps (for reference):**

**Magic (Layout Editor):**
```bash
git clone git://opencircuitdesign.com/magic
cd magic
./configure
make
sudo make install
```

**Netgen (LVS Tool):**
```bash
git clone git://opencircuitdesign.com/netgen
cd netgen
./configure
make
sudo make install
```

**Xschem (Schematic Capture):**
```bash
git clone https://github.com/StefanSchippers/xschem.git xschem_git
cd xschem_git
./configure
make
sudo make install
```

**ngspice (SPICE Simulator):**
```bash
# Download tarball from ngspice.sourceforge.io
tar -zxvf ngspice-37.tar.gz
cd ngspice-37
mkdir release
cd release
../configure --with-x --with-readline=yes --disable-debug
make
sudo make install
```

Install xterm for viewing ngspice plots:
```bash
sudo apt-get install xterm
```

**open_pdks (Technology Files):**
```bash
git clone git://opencircuitdesign.com/open_pdks
cd open_pdks
./configure --enable-sky130-pdk
make
sudo make install
```

**Verifying installations:**

After installation, we set up a working directory and copy the necessary configuration files:

```bash
mkdir inverter_lab
cd inverter_lab
mkdir mag xschem netgen

# Copy Xschem config
cd xschem
cp /usr/local/share/pdk/sky130A/libs.tech/xschem/xschemrc .
cp /usr/local/share/pdk/sky130A/libs.tech/ngspice/spinit .spiceinit

# Copy Magic config
cd ../mag
cp /usr/local/share/pdk/sky130A/libs.tech/magic/sky130A.magicrc .magicrc

# Copy Netgen config
cd ../netgen
cp /usr/local/share/pdk/sky130A/libs.tech/netgen/sky130A_setup.tcl .
```

Then verify each tool launches correctly:

```bash
# Launch Magic with SKY130 tech
magic -d XR &

# Launch Xschem
xschem &

# Launch Netgen
netgen &

# Launch ngspice
ngspice
```

Each tool should open its GUI without errors, confirming that the PDK paths are correctly configured.

![Tool Installation Verification - Magic](images/Day1_SK2/magic_launch.png)

![Tool Installation Verification - Xschem](images/Day1_SK2/xschem_launch.png)

![Tool Installation Verification - Netgen](images/Day1_SK2/netgen_launch.png)

![Tool Installation Verification - ngspice](images/Day1_SK2/ngspice_launch.png)

---

#### L2 - Creating SKY130 Device Layout in Magic

Magic is launched from the `mag/` directory where the `.magicrc` file resides, so it automatically loads the SKY130 technology:

```bash
cd mag
magic -d XR
```

The `-d XR` flag enables the Cairo-based X11 rendering for a better visual experience.

**Exploring a standard cell layout:**

To understand how SKY130 layouts look, we can load an existing standard cell from the PDK:

```tcl
% gds read /usr/local/share/pdk/sky130A/libs.ref/sky130_fd_sc_hd/gds/sky130_fd_sc_hd.gds
% load sky130_fd_sc_hd__inv_1
```

This loads the layout of a minimum-strength inverter from the high-density standard cell library. We can observe:
- The **PMOS** transistor in the upper half (inside the N-well)
- The **NMOS** transistor in the lower half
- **Power rails** (VPWR at top, VGND at bottom) running horizontally on met1
- The **input (A)** and **output (Y)** pins
- **Local interconnect (li1)** used for internal routing

To inspect a specific device, we can draw a basic NMOS transistor layout:

1. Select the `ndiffusion` layer and paint a rectangle for the active area
2. Paint `polysilicon` crossing over it to form the gate
3. Add `nsdm` (N+ source/drain implant) covering the diffusion
4. Connect the source, drain, and gate using `li1` and `licon` contacts

Magic enforces DRC rules in real-time — violations appear as white stippled patterns on the layout. The DRC error messages can be viewed with:

```tcl
% drc why
```

![Standard Cell Layout in Magic](images/Day1_SK2/magic_std_cell_layout.png)

![Device Layout in Magic](images/Day1_SK2/magic_device_layout.png)

---

#### L3 - Creating Simple Schematic in Xschem

Xschem is launched from the `xschem/` directory:

```bash
cd xschem
xschem
```

**Building an inverter schematic step by step:**

1. **Insert PMOS transistor:** Press `Ins` key or go to menu → place component → navigate to the SKY130 PDK library at `xschem/sky130_fd_pr/` → select `nfet_01v8.sym` and `pfet_01v8.sym`

2. **Set device parameters:** Select the NMOS device → press `q` to edit properties → set:
   - `W = 1` (width in um)
   - `L = 0.15` (length in um)
   - `nf = 1` (number of fingers)

   For the PMOS:
   - `W = 2` (typically 2x NMOS for balanced rise/fall)
   - `L = 0.15`
   - `nf = 1`

3. **Add power supply pins:** Insert `vdd.sym` and `gnd.sym` from the generic library

4. **Add I/O pins:** Insert `ipin.sym` for the input (name it `A`) and `opin.sym` for the output (name it `Y`)

5. **Wire the circuit:**
   - Connect PMOS source to VDD
   - Connect NMOS source to GND
   - Connect both gates together → this is the input `A`
   - Connect both drains together → this is the output `Y`
   - Connect PMOS body (nwell) to VDD
   - Connect NMOS body (psub) to GND

6. **Save** the schematic as `inverter.sch`

The completed inverter schematic should show the complementary PMOS/NMOS pair with proper power, ground, and signal connections.

![Inverter Schematic in Xschem](images/Day1_SK2/xschem_inverter_schematic.png)

---

#### L4 - Creating Symbol and Exporting Schematic in Xschem

**Creating a symbol for the inverter:**

1. From the schematic window, go to **Symbol → Make symbol from schematic**
2. Xschem generates `inverter.sym` automatically with the input pin `A` and output pin `Y`
3. The symbol can be customized — we can edit the graphic to look like a standard inverter triangle with a bubble

**Creating a testbench:**

Now we create a new schematic `inverter_tb.sch` to simulate the inverter:

1. Open a new schematic: **File → New schematic**
2. Place the inverter symbol we just created
3. Add stimulus and simulation components:
   - **Voltage source for VDD:** `vsource.sym` with `value = 1.8` (SKY130 nominal supply)
   - **Pulse source for input:** `vsource.sym` with pulse parameters:
     ```
     value = "pulse(0 1.8 0 10p 10p 1n 2n)"
     ```
     This creates a 500 MHz pulse with 10ps rise/fall times
   - **GND symbol** for ground reference

4. **Add simulation commands** using a `code_shown.sym` component:
   ```spice
   .lib /usr/local/share/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
   .tran 0.01n 10n
   .save all
   ```
   This includes the SKY130 SPICE models (typical-typical corner), runs a transient simulation for 10ns, and saves all node voltages.

5. **Generate the netlist:** Click the Netlist button or press the netlist hotkey. This produces `inverter_tb.spice`.

**Exporting the schematic netlist:**

For LVS, we also need a standalone netlist of just the inverter (not the testbench). In the inverter schematic:
- Go to **Simulation → LVS netlist: Top level is a .subckt** (enable this)
- Generate the netlist → this produces `inverter.spice` with proper `.subckt` / `.ends` wrapper

![Inverter Symbol](images/Day1_SK2/xschem_inverter_symbol.png)

![Inverter Testbench](images/Day1_SK2/xschem_inverter_testbench.png)

---

**Running the simulation in ngspice:**

From the testbench schematic in Xschem, click **Simulate** (or press the simulate hotkey). This launches ngspice with the testbench netlist.

In the ngspice terminal:

```
ngspice 1 -> plot v(A) v(Y)
```

This plots the input and output waveforms overlaid. We should observe:
- When input `A` goes HIGH (1.8V), output `Y` goes LOW (0V) — NMOS on, PMOS off
- When input `A` goes LOW (0V), output `Y` goes HIGH (1.8V) — PMOS on, NMOS off
- Sharp transitions with slight propagation delay and some overshoot/undershoot

This confirms the inverter functions correctly at the transistor level before we proceed to layout.

![ngspice Simulation Waveform](images/Day1_SK2/ngspice_inverter_simulation.png)

---

#### L5 - Importing Schematic to Layout and Inverter Layout Steps

**Importing the schematic netlist into Magic:**

In Magic (from the `mag/` directory), we import the schematic netlist to get a starting point for the layout:

```tcl
% magic -d XR
```

From the Magic console:
1. **File → Import SPICE** and select the `inverter.spice` file from the `xschem/` directory
2. Magic reads the netlist and places the devices (PMOS and PFET) as unrouted instances

Alternatively, we can build the layout from scratch, using the schematic as a reference.

**Manual inverter layout construction:**

1. **Place the NMOS transistor:**
   - Paint `ndiffusion` for the active area
   - Paint `polysilicon` crossing the diffusion (forms the gate)
   - The width and length should match the schematic (W=1um, L=0.15um)

2. **Place the PMOS transistor:**
   - Paint `nwell` first (PMOS lives inside N-well)
   - Paint `pdiffusion` inside the N-well
   - Paint `polysilicon` crossing the pdiffusion
   - Width = 2um, Length = 0.15um (matching schematic)

3. **Add contacts and routing:**
   - Place `licon` (local interconnect contact) on source/drain diffusions
   - Route using `li1` (local interconnect) for short connections
   - Place `mcon` to connect li1 to met1 where needed
   - Create power rails on `met1`: VPWR at top, VGND at bottom

4. **Connect the gates:**
   - Extend the poly from NMOS gate to PMOS gate (or route through li1)
   - This shared gate connection is the input `A`

5. **Connect the drains:**
   - Route NMOS drain to PMOS drain using li1/met1
   - This common drain node is the output `Y`

6. **Add substrate/well taps:**
   - Place `psubstratepdiff` (p+ tap) connected to VGND for NMOS body
   - Place `nsubstratendiff` (n+ tap inside N-well) connected to VPWR for PMOS body

7. **Add labels:**
   ```tcl
   % label A
   % label Y
   % label VPWR
   % label VGND
   ```

Magic continuously runs DRC in the background. Fix any violations (shown as white hatched regions) by adjusting spacing, widths, or adding missing overlaps.

![Inverter Layout in Magic - Complete](images/Day1_SK2/magic_inverter_layout.png)

![Inverter Layout - Zoomed with Labels](images/Day1_SK2/magic_inverter_layout_labels.png)

---

#### L6 - Final DRC/LVS Checks

**Running DRC:**

Magic performs DRC interactively, but we can force a full check:

```tcl
% drc check
```

To see all DRC errors:
```tcl
% drc listall why
```

To navigate through errors one by one:
```tcl
% drc find
```

The DRC should return **0 errors** for a clean layout. Common issues to fix:
- Minimum width violations on narrow wires
- Spacing violations between adjacent layers
- Missing overlap/enclosure of vias
- Missing well taps

![DRC Clean Check](images/Day1_SK2/magic_drc_clean.png)

**Extracting the layout netlist:**

Once DRC is clean, we extract the netlist from the layout for LVS:

```tcl
% extract do local
% extract all
% ext2spice lvs
% ext2spice
```

This generates `inverter.spice` in the `mag/` directory. The commands do the following:
- `extract do local` — write extraction files to the current directory
- `extract all` — extract the circuit from the layout (creates `.ext` files)
- `ext2spice lvs` — configure the extractor for LVS mode (no parasitic caps)
- `ext2spice` — convert the `.ext` files to a SPICE netlist

Clean up intermediate files:
```bash
rm *.ext
```

**Running LVS with Netgen:**

Now we compare the schematic netlist (from Xschem) with the layout netlist (from Magic):

```bash
cd ../netgen
netgen -batch lvs "../mag/inverter.spice inverter" "../xschem/inverter.spice inverter"
```

The command syntax is:
```
netgen -batch lvs "<file1> <subcircuit1>" "<file2> <subcircuit2>"
```

Netgen reads both netlists, matches their topology (nets and devices), and reports the result. The output should state:

```
Final result: Circuits match uniquely.
```

This means:
- The same number and type of devices exist in both netlists
- The connectivity (which device pin connects to which net) is identical
- The pin names match between schematic and layout

If there is a mismatch, Netgen writes a detailed report to `comp.out` showing:
- Which nets don't match
- Which devices don't match
- Pin-by-pin comparison

Common LVS errors and their causes:
| Error | Typical Cause |
|-------|---------------|
| Net mismatch | Missing or extra connection in layout |
| Device mismatch | Wrong transistor W/L, or extra parasitic device extracted |
| Pin mismatch | Label missing or placed on wrong net in layout |
| Missing device | Component not placed or not properly connected |

![LVS Result - Circuits Match](images/Day1_SK2/netgen_lvs_match.png)

---

## References

- [SkyWater SKY130 PDK Documentation](https://skywater-pdk.readthedocs.io/en/main/)
- [SkyWater PDK GitHub Repository](https://github.com/google/skywater-pdk)
- [Magic VLSI Layout Tool](http://opencircuitdesign.com/magic/)
- [Netgen LVS Tool](http://opencircuitdesign.com/netgen/)
- [Xschem Schematic Editor](https://xschem.sourceforge.io/stefan/index.html)
- [ngspice Circuit Simulator](http://ngspice.sourceforge.net/)
- [OpenLANE Flow](https://github.com/The-OpenROAD-Project/OpenLane)
- [VSD - VLSI System Design](https://www.vlsisystemdesign.com/)

## Acknowledgements

- **Tim Edwards** — for maintaining Magic, Netgen, and the open_pdks infrastructure
- **Kunal Ghosh** — Co-founder, VSD, for the Physical Verification course content
- **Marvell Technology, SFAL, ESSCI** — for the Women in VLSI initiative
- **SkyWater Technology** — for open-sourcing the 130nm PDK
- **Google** — for sponsoring the open-source PDK program
