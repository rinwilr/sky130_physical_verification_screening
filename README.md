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

The **SkyWater SKY130** is the first fully open-source PDK, born from a collaboration between **Google** and **SkyWater Technology Foundry**. The "130" in SKY130 stands for the 130nm feature size — the minimum transistor length the process targets. However, **the actual minimum-length device in SKY130 is 150nm, not 130nm**, due to process-specific reasons.

A **PDK (Process Design Kit)** bridges the gap between the chip designer and the foundry. It bundles everything needed to design a chip for a particular process: design rules, device models, standard cell libraries, I/O cells, and primitive device definitions.

| Resource | Link |
|----------|------|
| Documentation | https://skywater-pdk--136.org.readthedocs.build |
| PDK Library & Files | https://github.com/google/skywater-pdk |
| Community (Slack) | https://join.skywater.tools |


#### L2 - Open-Source EDA Tools

The central piece that ties the toolchain together is **open_pdks** — it installs the SKY130 PDK and configures all supported EDA tools to work with it.

- open_pdks: http://opencircuitdesign.com/open_pdks
- GitHub: https://github.com/RTimothyEdwards/open_pdks

Add image here

**Installing SKY130 PDK via open_pdks:**
```bash
git clone https://github.com/RTimothyEdwards/open_pdks
cd open_pdks
configure --enable-sky130-pdk
make
sudo make install
```

**Primary tools supported by open_pdks:**

| Tool | Role | Link |
|------|------|------|
| **Magic** | Layout editor, DRC, parasitic extraction | http://opencircuitdesign.com/magic |
| **Netgen** | LVS netlist comparison | http://opencircuitdesign.com/netgen |
| **Xschem** | Schematic capture & netlist generation | https://github.com/StefanSchippers/xschem |
| **ngspice** | SPICE circuit simulation | http://ngspice.sourceforge.net |
| **KLayout** | GDS viewer, DRC scripting, XOR | https://www.klayout.de |
| **OpenLANE** | Automated RTL-to-GDSII flow | https://github.com/The-OpenROAD-Project/OpenLane |

**Additional tools:**

| Tool | Role | Link |
|------|------|------|
| **iverilog** | Verilog simulation | https://iverilog.icarus.com |
| **qflow** | Digital synthesis flow | http://opencircuitdesign.com/qflow |
| **IRSIM** | Switch-level simulation | http://opencircuitdesign.com/irsim |
| **xcircuit** | Schematic drawing | http://opencircuitdesign.com/xcircuit |

**SkyWater SKY130 Libraries:**

**1. Digital standard cells**
* sky130_fd_sc_hd
* sky130_fd_sc_hs
* sky130_fd_sc_lp
* sky130_fd_sc_hdll
* sky130_fd_sc_ms
* sky130_fd_sc_hvl
* sky130_fd_sc_ls
  
**2. Primitive devices/analog**
* sky130_fd_pr
  
**3. I/O cells**
* sky130_fd_io
  
**4. 3rd-party libraries**
* sky130_ml_xx_hd
* sky130_sram_macros

**1. Centralized PDK Filesystem Structure**

The standard reference installation for Open PDKs places the PDK libraries and technology files in a centralized system directory (`/usr/share/pdk/`). Below is the standard directory structure:

```text
/usr/share/pdk/
└── sky130A/
    ├── libs.tech/
    │   ├── magic/
    │   ├── openlane/
    │   ├── xschem/
    │   └── ngspice/
    └── libs.ref/
        ├── sky130_fd_sc_hd/
        │   ├── mag/
        │   ├── spice/
        │   ├── lib/
        │   ├── lef/
        │   ├── gds/
        │   └── verilog/
        ├── sky130_fd_io/
        └── sky130_fd_pr/
```

**SkyWater SKY130 Reference Libraries (`libs.ref`)**
* **`sky130_fd_sc_hd`**: High-Density (HD) Digital standard cells.
* **`sky130_fd_io`**: Input/Output (I/O) cell libraries for pad frames.
* **`sky130_fd_pr`**: Primitive devices (Transistors, Resistors, Capacitors) for analog simulation.

---

**2. Tool Configuration & Workspace Initialization**

To use EDA tools like **Magic VLSI** and **Xschem** locally inside your project workspace, you must initialize your local folders with configuration files mapped from the centralized PDK installation.

```text
/usr/share/pdk/sky130A/libs.tech/
├── magic/
│   └── sky130A.magicrc  ──────────>  [Copies/Links to] ──>  project_root/mag/.magicrc
└── xschem/
    └── xschemrc         ──────────>  [Copies/Links to] ──>  project_root/xschem/xschemrc
```
**3. Organizing Project Subcells**

When creating hierarchical chip designs containing multiple custom subcells or macro blocks, you can choose between two main repository layout strategies.

**Option 1: Top-Level Tool Architecture (Recommended for Flat Verification)**
This method separates files cleanly based on the specific tool interface. Layout files stay together, and schematic files stay together.

```text
project_root/
├── mag/
│   ├── subcell_name_1/
│   └── subcell_name_2/
└── xschem/
    ├── subcell_name_1/
    └── subcell_name_2/
```
**Option 2: Component-Centric Architecture (Recommended for Modular IPs)**
This method groups all design file types belonging to a specific sub-block into a single unified directory.

```text
project_root/
└── subcell_name/
    ├── mag/
    └── xschem/
```
#### L3 - Understanding SkyWater PDK - Layers

The SkyWater SKY130 process is described as a **hybrid 130nm-180nm standard CMOS fabrication process**. Most FET transistors have a minimum length of 150nm, while specially designed ones can go down to 130nm.

The process has **5 layers of aluminium metal** plus a layer of **titanium nitride (TiN)** which acts like a metal layer but has relatively high resistance — making it unsuitable for long routes. This layer, called **local interconnect (li1)**, is probably the most unintuitive layer in the whole process for anyone used to processes where the first routing layer above polysilicon is aluminium.

**The general rule for local interconnect:** use it only to connect neighboring devices, and otherwise contact up immediately to metal1. If you look at the SkyWater standard cells, local interconnect is used for power and ground rails — but because it should never be used for routing, the entire length of both li1 rails is strapped with metal1 above.

**Contacts to polysilicon** are also a bit unusual in this process — they require a **nitride poly cut (NPC)** layer around the contacts. Usually this is not an issue, but it is possible to place contacts in a way that does not satisfy design rules.

The metal layers have **progressive thicknesses** — local interconnect is the thinnest, metal5 is the thickest. This is a bare minimum of metal layers for this feature size. When running digital flows, the tools should reserve metal5 for power routing, and local interconnect is unusable for signal routing — which leaves only 4 real routing layers.

**Frontend vs. Backend layers:**

The metal layers are the **backend layers**. Everything below them is the **frontend**. They differ in their fabrication steps:
- **Frontend** — mainly diffusion and ion implantation processes. Includes oxide growth (insulation layers)
- **Backend** — metal deposition, usually by sputtering

**Frontend layers include:**
- **diff and tap** — Diffusion forms the active areas of transistors. Tap provides substrate/well contacts for body connections
- **Deep N-well (dnwell)** — Creates a barrier that isolates circuits from substrate noise
- **HVI (High Voltage Implant)** — Has more than one use; it forms the thick gate oxide needed for high-voltage (5V) devices
- **MiM cap layers** — Metal-insulator-Metal capacitor layers for analog circuits

**Other special layers:**
- **RDL (Redistribution Layer)** — Fabricated by a 3rd party. Used for wafer-level packaging (no wire bonding needed)

![SKY130 Layer Stack](images/Day1_SK1/sky130_layers.png)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/84ea4264-8a85-463e-93a9-c926f4f8c643" />

#### L4 - Understanding SkyWater PDK - Devices

**MOSFETs:**

| Device | Type | Description |
|--------|------|-------------|
| `sky130_fd_pr__nfet_01v8` | NMOS 1.8V | Standard threshold, general purpose |
| `sky130_fd_pr__pfet_01v8` | PMOS 1.8V | Standard threshold, general purpose |
| `sky130_fd_pr__nfet_01v8_lvt` | NMOS LVT | Low threshold, higher speed, more leakage |
| `sky130_fd_pr__pfet_01v8_hvt` | PMOS HVT | High threshold, lower leakage, slower |
| `sky130_fd_pr__nfet_g5v0d10v5` | NMOS 5V | Thick gate oxide, for I/O |
| `sky130_fd_pr__pfet_g5v0d10v5` | PMOS 5V | Thick gate oxide, for I/O |

**Passive Devices:** Poly resistors, high-resistance poly resistors, MiM capacitors, varactors, P-N junction diodes

**Bipolar Devices:** PNP and NPN BJTs available for bandgap references and analog circuits

All devices are **parameterized cells (pcells)** — the designer specifies W and L, and the layout is generated automatically. SPICE models use BSIM4 with corner models (tt, ff, ss, sf, fs) for process variation analysis.

![SKY130 Devices](images/Day1_SK1/sky130_devices.png)
<img width="681" height="502" alt="image" src="https://github.com/user-attachments/assets/a92c60af-7c0c-47eb-b2f2-219a7bbc4eae" />
<img width="642" height="537" alt="image" src="https://github.com/user-attachments/assets/76414375-c560-4b27-b31c-6b877c88bbb1" />
<img width="691" height="397" alt="image" src="https://github.com/user-attachments/assets/5ddc06e1-bc4c-47df-a98b-1f88c3ea0177" />
<img width="688" height="532" alt="image" src="https://github.com/user-attachments/assets/a30ec908-1d6c-43dd-8e70-20327e093aaf" />
<img width="720" height="480" alt="image" src="https://github.com/user-attachments/assets/8fafc59b-fcd2-4939-822d-cf5909a490a6" />
<img width="565" height="507" alt="image" src="https://github.com/user-attachments/assets/50c0e399-a785-4763-b762-f3bf3b9dda49" />
<img width="673" height="493" alt="image" src="https://github.com/user-attachments/assets/efa84707-5716-4dc2-84e1-4dc895d5e173" />

#### L5 - Understanding SkyWater PDK Libraries

**SKY130 Libraries:**

1. **Digital standard cells:**
   - `sky130_fd_sc_hd` — High-density (most commonly used)
   - `sky130_fd_sc_hdll` — High-density, low-leakage
   - `sky130_fd_sc_hs` — High-speed
   - `sky130_fd_sc_ms` — Medium-speed
   - `sky130_fd_sc_ls` — Low-speed
   - `sky130_fd_sc_lp` — Low-power
   - `sky130_fd_sc_hvl` — High-voltage
   - <img width="682" height="510" alt="image" src="https://github.com/user-attachments/assets/3889b039-b51e-4d41-b0a2-3d08c7df90a0" />
   - <img width="707" height="562" alt="image" src="https://github.com/user-attachments/assets/312b7b91-cecb-4316-9f0a-2b2c81782347" />

2. **Primitive devices / analog:** `sky130_fd_pr`
<img width="1188" height="575" alt="image" src="https://github.com/user-attachments/assets/3d5e7eba-1dc8-42f4-a989-d421fcd4c92d" />

3. **I/O cells:** `sky130_fd_io`
<img width="1201" height="582" alt="image" src="https://github.com/user-attachments/assets/3d82080b-5ef1-44ed-8f17-9f4dfd5c4353" />

4. **3rd-party libraries:** `sky130_ml_xx_hd`, `sky130_sram_macros`
<img width="1197" height="571" alt="image" src="https://github.com/user-attachments/assets/01691ddb-1d0f-4ad8-b9f0-db914e8f5978" />

**PDK Installed Filesystem Structure:**

```
/usr/share/pdk/
└── sky130A/
    ├── libs.tech/          ← Technology files for each tool
    │   ├── magic/
    │   ├── openlane/
    │   └── xschem/
    └── libs.ref/           ← Reference libraries
        ├── sky130_fd_sc_hd/
        ├── sky130_fd_pr/
        └── sky130_fd_io/
```

**Project Directory Structure:**

```
project_root/
├── xschem/         ← Schematics (copy xschemrc here)
├── mag/            ← Layouts (copy .magicrc here)
├── spice/          ← SPICE netlists
├── openlane/       ← OpenLANE config
└── verilog/        ← RTL source files
```

Config files are copied from the PDK installation into the project:
- `sky130A/libs.tech/magic/sky130A.magicrc` → `mag/.magicrc`
- `sky130A/libs.tech/xschem/xschemrc` → `xschem/xschemrc`

**For projects with multiple subcells**, two organization styles:

```
# Style 1: Subcells inside tool dirs     # Style 2: Tool dirs inside subcells
project_root/                             project_root/
├── mag/                                  ├── subcell_A/
│   ├── subcell_A/                        │   ├── mag/
│   └── subcell_B/                        │   └── xschem/
└── xschem/                               └── subcell_B/
    ├── subcell_A/                            ├── mag/
    └── subcell_B/                            └── xschem/
```

![SKY130 Libraries](images/Day1_SK1/sky130_libraries.png)

#### L6 - Open-Source Tools and Flows

The complete open-source RTL-to-GDSII flow passes through these stages:

1. **Synthesis** (Yosys/ABC) → Gate-level netlist
2. **Floorplan + Placement** (OpenROAD) → Physical placement of cells
3. **CTS** (TritonCTS) → Clock tree synthesis
4. **Routing** (FastRoute + TritonRoute) → Metal connections
5. **Signoff** → **DRC** (Magic) + **LVS** (Netgen)
6. **Tapeout** → Final GDSII file sent to the foundry

**DRC** checks that every shape in the layout satisfies the foundry's manufacturing rules. **LVS** confirms the layout implements exactly the circuit described in the schematic. No foundry accepts a design without both passing clean.

**OpenLANE** automates this entire pipeline end to end. For custom analog design, each tool is used individually — which is the focus of this course.

![RTL to GDSII Flow](images/Day1_SK1/rtl_to_gdsii_flow.png)

---

### SK2 - Tool Installation and Basic DRC/LVS Design Flow

#### L1 - Check Tool Installations

The tools must be installed in the correct order. **open_pdks is installed last** because it configures technology files based on where each tool is installed.

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

**Setting up the working directory:**

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

**Verifying installations:**

![Magic Launch](images/Day1_SK2/magic_launch.png)

![Xschem Launch](images/Day1_SK2/xschem_launch.png)

![Netgen Launch](images/Day1_SK2/netgen_launch.png)

![ngspice Launch](images/Day1_SK2/ngspice_launch.png)

---

#### L2 - Creating SKY130 Device Layout in Magic

Launched Magic with `magic -d XR` from the `mag/` directory and explored an existing standard cell:

```tcl
% gds read /usr/local/share/pdk/sky130A/libs.ref/sky130_fd_sc_hd/gds/sky130_fd_sc_hd.gds
% load sky130_fd_sc_hd__inv_1
```

The loaded inverter cell shows the PMOS (inside N-well) and NMOS transistors, power rails (VPWR, VGND) on met1, and li1 used for internal routing.

![Standard Cell in Magic](images/Day1_SK2/magic_std_cell_layout.png)

![Device Layout](images/Day1_SK2/magic_device_layout.png)

---

#### L3 - Creating Simple Schematic in Xschem

Launched Xschem and built the CMOS inverter schematic by placing components from the open_pdk library:

- `sky130_fd_pr__nfet_01v8` (W=1, L=0.15) and `sky130_fd_pr__pfet_01v8` (W=2, L=0.15)
- Gates tied together → input **A**
- Drains tied together → output **Y**
- PMOS source → VDD, NMOS source → GND
- Body connections: PMOS body → VDD, NMOS body → GND

Device properties set here in the schematic will automatically reflect in the layout.

![Inverter Schematic](images/Day1_SK2/xschem_inverter_schematic.png)

---

#### L4 - Creating Symbol and Exporting Schematic in Xschem

Converted the schematic to a reusable symbol using **Symbol → Make symbol from schematic**. This symbol can now be placed in a testbench.

![Inverter Symbol](images/Day1_SK2/xschem_inverter_symbol.png)

Built a testbench (`inverter_tb.sch`) with the inverter symbol, VDD supply (1.8V), input pulse source, and simulation commands:

```spice
.lib /usr/local/share/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.tran 0.01n 10n
.save all
```

![Inverter Testbench](images/Day1_SK2/xschem_inverter_testbench.png)

**Pre-Layout Simulation (ngspice):**

Simulated the testbench in ngspice. The output Y correctly inverts input A.

![Pre-Layout Simulation](images/Day1_SK2/ngspice_inverter_simulation.png)

---

#### L5 - Importing Schematic to Layout and Inverter Layout Steps

Exported the schematic netlist and imported it into Magic via **File → Import SPICE**. Then completed the layout:

- Placed NMOS and PMOS devices matching the schematic W/L
- Routed with local interconnect (li1) and metal1
- Added power rails (VPWR, VGND) on met1
- Added substrate and well taps
- Labeled all ports: A, Y, VPWR, VGND

![Inverter Layout](images/Day1_SK2/magic_inverter_layout.png)

![Layout with Labels](images/Day1_SK2/magic_inverter_layout_labels.png)

---

#### L6 - Final DRC/LVS Checks

**DRC:**

```tcl
% drc check
% drc listall why
```

All violations resolved — layout is DRC clean.

![DRC Clean](images/Day1_SK2/magic_drc_clean.png)

**Netlist Extraction:**

```tcl
% extract do local
% extract all
% ext2spice lvs
% ext2spice
```

![Extraction](images/Day1_SK2/magic_extraction.png)

**LVS:**

```bash
netgen -batch lvs "../mag/inverter.spice inverter" "../xschem/inverter.spice inverter"
```

Result: **Circuits match uniquely.**

![LVS Match](images/Day1_SK2/netgen_lvs_match.png)

**Post-Layout Simulation:**

Ran simulation using the extracted netlist (with parasitics) to confirm the layout does not degrade performance.

![Post-Layout Simulation](images/Day1_SK2/postlayout_simulation.png)

---

## References

- [SkyWater SKY130 PDK](https://github.com/google/skywater-pdk)
- [Open_PDKs](https://github.com/RTimothyEdwards/open_pdks)
- [Magic VLSI](http://opencircuitdesign.com/magic/)
- [Netgen LVS](http://opencircuitdesign.com/netgen/)
- [Xschem](https://xschem.sourceforge.io/stefan/index.html)
- [ngspice](http://ngspice.sourceforge.net/)
- [VSD - VLSI System Design](https://www.vlsisystemdesign.com/)

## Acknowledgements

- **Tim Edwards** — Maintainer of Magic, Netgen, and open_pdks
- **Kunal Ghosh** — Co-founder, VSD
- **Marvell Technology, SFAL, ESSCI** — Women in VLSI initiative
- **SkyWater Technology & Google** — Open-source PDK program
