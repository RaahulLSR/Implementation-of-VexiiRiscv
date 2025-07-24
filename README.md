# VexiiRiscv Synthesis Guide

## 🧠 Introduction

This repository documents the complete process of configuring, generating, synthesizing, and verifying the **VexiiRiscv** CPU core for use in FPGA and ASIC flows. The goal is to provide a clean and educational walkthrough for students, engineers, and hobbyists working on advanced RISC-V CPU designs.

> ⚠️ **Note**: This repository is not about modifying the VexiiRiscv internals, but about **how to use the official VexiiRiscv repo** to configure a CPU instance of your choice and synthesize it for implementation on FPGA or ASIC targets. This includes any minimal changes required to the original source for successful instantiation and tool compatibility.

---

### 🔷 About VexiiRiscv

[VexiiRiscv](https://github.com/SpinalHDL/VexiiRiscv) is the second-generation RISC-V CPU generator developed using SpinalHDL. It is designed to span a wide spectrum of applications—from tiny microcontrollers to full-featured multi-core Linux-capable SoCs.

- ✅ Implements 32-bit and 64-bit RISC-V ISA (IMAFDCSU)
- 🔁 Supports both bare-metal and full Linux OS (Buildroot, Debian, etc.)
- 🧩 Configurable issue width (single or dual), late-ALU, branch prediction, MMU (SV32/SV39), PMP, cache features
- 🧠 Includes advanced features like BTB/RAS/GShare predictors, non-blocking caches, hardware/software prefetching
- 🔗 Supports TileLink, AXI4, Wishbone buses (depending on config)
- 🚀 Achieves high performance with features like out-of-order D-cache and instruction prefetching
- 🛠️ Designed with better scalability, clean frontend, and high-frequency design in mind
- 🆓 Open-source under the MIT License

---

### 📰 Learn More

Here are some great resources to understand and follow VexiiRiscv's evolution:

- 🔗 [VexiiRiscv Official Docs](https://spinalhdl.github.io/VexiiRiscv-RTD/master/VexiiRiscv/Introduction/index.html)
- 🎥 [Running Debian on VexiiRiscv](https://youtu.be/dR_jqS13D2c?t=112)
- 📄 [FSiC 2024 Talk](https://wiki.f-si.org/index.php?title=Moving_toward_VexiiRiscv)
- 📄 [ORConf 2024](https://fossi-foundation.org/orconf/2024#vexiiriscv--a-debian-demonstration)

---

## 🖥️ System Info

This entire process was carried out on:

> **Ubuntu 22.04 LTS** running inside **Oracle VirtualBox**

A clean VM setup helps ensure all toolchains work without cross-version issues.

---

## 2️⃣ Prerequisites and Initial Setup

The official VexiiRiscv repository allows you to set up the environment in two main ways:
1. Using a pre-built Docker image
2. **(Recommended)** Building all the tool flows and dependencies from scratch

> 💡 This guide follows the second approach for **full flexibility and transparency**.

---

### 🧱 Environment Requirements

To work with VexiiRiscv, the following tools and dependencies are needed:

- Java JDK 8
- SBT (Scala Build Tool)
- Verilator (optional – for simulation)
- RVLS / Spike + ELFIO (optional – for lock-step simulation and ELF loading)
- RISC-V GCC Toolchain (optional – for compiling RISC-V binaries)

> 🔒 **Tip**: To avoid version mismatches and package conflicts, it's best to set this up in a clean Linux environment (e.g., Ubuntu 20.04 or 22.04 on VirtualBox).

> 🚫 **Warning**: Never try to interrupt the below commands while running. It may take some time espeially during the first time you run it. If interrupted it may lead to partial and damaged install which will create trouble in later stages and very hard to debug.

---

### ⚙️ Installing Java JDK 8

```bash
sudo add-apt-repository -y ppa:openjdk-r/ppa
sudo apt-get update
sudo apt-get install openjdk-8-jdk -y
sudo update-alternatives --config java
sudo update-alternatives --config javac 
```
⚙️ Installing SBT (Scala Build Tool)
```bash

echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo apt-key add
sudo apt-get update
sudo apt-get install sbt
```

⚙️ Installing Verilator (optional for simulation) 
```bash

sudo apt-get install git make autoconf g++ flex bison help2man
git clone https://github.com/verilator/verilator
cd verilator
git checkout v4.216  # You can choose any stable version
autoconf
./configure
make
sudo make install
```

⚙️ RVLS / Spike + ELFIO (optional)
```bash

sudo apt-get install device-tree-compiler libboost-all-dev
git clone https://github.com/serge1/ELFIO.git
cd ELFIO
git checkout d251da09a07dff40af0b63b8f6c8ae71d2d1938d  # Compatible version
sudo cp -R elfio /usr/include
cd .. && rm -rf ELFIO
```

⚙️ Installing the RISC-V GCC Toolchain
```bash

git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
./configure --prefix=/opt/riscv --enable-multilib
make
sudo make install
echo 'export PATH=/opt/riscv/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
⚙️ Cloning git repo
```bash
git clone --recursive https://github.com/SpinalHDL/VexiiRiscv.git
cd VexiiRiscv

# (optional) Compile riscv-isa-sim (spike), used as a golden model during the sim to check the dut behaviour (lock-step)
cd ext/riscv-isa-sim
mkdir build
cd build
../configure --prefix=$RISCV --enable-commitlog  --without-boost --without-boost-asio --without-boost-regex
make -j$(nproc)
cd ../../..

# (optional) Compile RVLS, (need riscv-isa-sim (spike)
cd ext/rvls
make -j$(nproc)
cd ../..
```

✅ Verifying Your Setup
To verify that everything is working, try running:

```bash

sbt "runMain vexiiriscv.Generate"
```
If a .v file is generated in the generated/ folder, your environment is set up correctly.

---

<img width="940" height="287" alt="image" src="https://github.com/user-attachments/assets/f47ef5c6-8bd0-486a-8ae1-cf6275bbcfb4" />


---

## 3️⃣ Verilog Generation

Once your environment is properly set up, you can now generate the Verilog for a custom VexiiRiscv CPU configuration.

---

### 🔧 Basic Command

To generate the Verilog:

```bash
sbt "Test/runMain vexiiriscv.Generate"
This will generate a .v file inside a generated/ folder.
```

🧩 Exploring Configuration Options
You can explore all supported options using:

```bash
sbt "Test/runMain vexiiriscv.Generate --help" 
```

Some of the most useful options include:

| Parameter                | Description                                                                 |
|--------------------------|-----------------------------------------------------------------------------|
| `--xlen=32/64`           | Set RISC-V data width (default: 32-bit)                                     |
| `--with-rvm`             | Enable multiplication/division instructions                                 |
| `--with-rvc`             | Enable compressed instructions                                               |
| `--with-rva`             | Enable atomic instructions                                                   |
| `--with-rvf`             | Enable 32-bit floating point                                                 |
| `--with-rvd`             | Enable 64-bit floating point                                                 |
| `--with-supervisor`      | Enable privileged mode, MMU (SV32/SV39), PMP                                 |
| `--with-btb`             | Enable Branch Target Buffer prediction                                       |
| `--with-gshare`          | Enable GShare branch predictor (requires BTB)                                |
| `--with-ras`             | Enable Return Address Stack prediction (requires BTB)                        |
| `--with-jtag-tap`        | Enable JTAG debugging support                                                |
| `--allow-bypass-from=0`  | Enable integer result bypassing from stage 0 (improves performance)          |
| `--regfile-async`        | Use asynchronous read register file (may not be supported on all FPGAs)      |
| `--mmu-sync-read`        | Make MMU memories use synchronous reads (helps for FPGA timing)              |
| `--fetch-l1`             | Enable L1 instruction cache                                                  |
| `--fetch-l1-ways=2/4/...`| Set number of L1 I-cache ways (each way is 4KB)                              |
| `--lsu-l1`               | Enable L1 data cache                                                         |
| `--lsu-l1-ways=2/4/...`  | Set number of L1 D-cache ways                                                |
| `--report-model`         | Print the CPU's pipeline execution model in the terminal                    |


📊 Example Output of --report-model
Using the --report-model flag provides insights into how each instruction executes within the pipeline. For example:

```bash

Execute lane : lane0
- Layer : early0
  - instruction : Rvi_ADD
    - read  integer[RS1], stage 0
    - read  integer[RS2], stage 0
    - write integer[RD], stage 0
    - completion stage 0
```

This is useful for deeply understanding the internal architecture and debugging performance bottlenecks.

✅ Sample Command for a Rich Config CPU
```bash

sbt "Test/runMain vexiiriscv.Generate --xlen=32 --with-rvc --with-rvm --with-rva --with-supervisor --with-btb --with-gshare --with-ras --fetch-l1 --lsu-l1 --with-jtag-tap --report-model"
```
📷 Screenshot:
IMMAGE 2

🧪 Verilator Simulation Preview (Optional)
ℹ️ If your simulator uses X-propagation (not Verilator), enable: --with-boot-mem-init

To run a Verilator-based simulation:

```bash

sbt
compile
Test/runMain vexiiriscv.tester.TestBench --with-mul --with-div --load-elf ext/NaxSoftware/baremetal/dhrystone/build/rv32ima/dhrystone.elf --trace-all

```
This will generate:

test.fst: Open with GTKWave (signal waveform)

konata.log: Open with Konata viewer

spike.log: Spike (golden model) output

tracer.log: VexiiRiscv's simulated output

For a detailed simulation and firmware guide, refer to:  
📘 [VexiiRiscv Simulation Tutorial](https://spinalhdl.github.io/VexiiRiscv-RTD/master/VexiiRiscv/Tutorial/index.html)

## Testing VexiiRiscv Firmware with Blinky and OpenOCD

### Blinky C Code

For testing purposes, a simple blinky code was used:

```cpp
#include <system/soc.hpp>
#include <utils/bytes.hpp>

[[noreturn]]
int main() {
    using namespace std::chrono_literals;

    soc::init();

    soc::uart.write(utils::to_bytes("Starting LED blinky!\n"));

    while (true) {
        reinterpret_cast<volatile uint32_t>(0x10003000) = 0xAAAA;
        soc::uart.write(utils::to_bytes("LED: 0xAAAA\n"));
        soc::delay(1us);

        reinterpret_cast<volatile uint32_t>(0x10003000) = 0x5555;
        soc::uart.write(utils::to_bytes("LED: 0x5555\n"));
        soc::delay(1us);
    }
}
```

> 💡 For linker script and other files, refer to the [VexiiFirmware GitHub repo](https://github.com/VexiiRiscv/vexiifirmware).

---

### Connecting with OpenOCD to the Simulation

**OpenOCD** is used to connect your PC to a microcontroller for debugging or programming through JTAG.

You can simulate this JTAG connection between OpenOCD and VexiiRiscv using TCP:

1. **Install OpenOCD**

2. Run the VexiiRiscv simulation with additional flags:

```bash
sbt "Test/runMain vexiiriscv.tester.TestBench --no-probe --no-rvls-check --debug-privileged --debug-jtag-tap --jtag-remote"
```

> ⚠️ Remove `--trace-all` to avoid large log files and simulation slowdowns.

**Flag Descriptions:**

* `--no-probe`: Disables CPU inactivity watchdog.
* `--no-rvls-check`: Disables RVLS golden model checking.
* `--debug-privileged`: Enables debug interface and CSRs.
* `--debug-jtag-tap`: Adds JTAG logic.
* `--jtag-remote`: Enables TCP bridge for JTAG.

3. Launch OpenOCD:

```bash
cd src/main/tcl/openocd/
openocd -f vexiiriscv_sim.tcl
```

You should see:

```
Info : Listening on port 3333 for gdb connections
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
```

---

### Connecting via Telnet

For low-level debugging, telnet is often more effective:

```bash
telnet localhost 4444
```

**Example commands:**

```bash
mdw 0x80000000                        # Read 32-bit word
mww 0x80000000 0x04200513             # Write instruction to memory
reg pc 0x80000000                     # Set PC
step                                  # Execute one instruction
reg pc                                # Read PC
reg a0                                # Read register a0
```

---

### Hello World in C

**Folder Setup:**

```bash
cd VexiiRiscv
mkdir -p helloworld/src
cd helloworld
```

**main.c**

```c
#include <sim.h>

void main(){
    for(int i=0;i<10;i++) {
        char *str = "hello world";
        while(*str) sim_putchar(*str++);
    }
}
```

**Makefile**

```makefile
PROJ_NAME=helloworld
STANDALONE=../ext/NaxSoftware/baremetal
SRCS =  $(wildcard src/*.c) \
        $(wildcard src/*.cpp) \
        $(wildcard src/*.S) \
        ${STANDALONE}/common/start.S
include ../ext/NaxSoftware/baremetal/common/app.mk
```

**Compile:**

```bash
make
```

**Output:**

```
build/helloworld.elf
build/helloworld.map
```

**Fix Compilation Error:**
If you see:

```
Error: unrecognized opcode `csrc mstatus,x1', extension `zicsr' required
```

Add the following to line 1 of `start.S`:

```asm
.option arch, +zicsr
```

---

### Run Simulation with Compiled ELF

```bash
cd ..
sbt "Test/runMain vexiiriscv.tester.TestBench --with-rvm --allow-bypass-from=0 --load-elf helloworld/build/helloworld.elf --trace-all --no-probe --debug-privileged --no-rvls-check"
```

Expected Output:

```
hello world
hello world
...
hello world (10 times)
```

✅ Your test firmware runs correctly on the simulated VexiiRiscv core!

## MicroSoC Integration with VexiiRiscv

When building a usable CPU core, a bare CPU is not enough. We need peripherals and memory to build a system. For that, SoC (System-on-Chip) generators are used. VexiiRiscv supports two major SoC generators:

* [LiteX](https://github.com/enjoy-digital/litex)
* [MicroSoC](https://github.com/SpinalHDL/VexiiRiscv/tree/dev/src/main/scala/vexiiriscv/soc/micro)

This section focuses on **MicroSoC** due to its simplicity, flexibility, and tight integration with VexiiRiscv.

### About MicroSoC

MicroSoC is a lightweight SoC generator based on:

* VexiiRiscv CPU core
* TileLink interconnect

#### Goals:

* Provide a simple reference SoC design
* Deliver a small and light FPGA SoC
* Focus on high-frequency rather than high-IPC performance

### Key Files

* `MicroSoc.scala`: SoC top-level module
* `MicroSocGen.scala`: Generates Verilog from Scala
* `MicroSocSim.scala`: Verilator-based simulation testbench

These files are well-commented to help new users understand their structure and function.

### Default Generation

```sh
# Generate SoC Verilog with default config
sbt "runMain vexiiriscv.soc.micro.MicroSocGen"

# Example: SoC with 32KB RAM, RV32IMC, 50 MHz clock
sbt "runMain vexiiriscv.soc.micro.MicroSocGen --ram-bytes=32768 --with-rvm --with-rvc --system-frequency=50000000"

# List all available config options
sbt "runMain vexiiriscv.soc.micro.MicroSocGen --help"
```

### Simulating MicroSoC

```sh
# Run default simulation
sbt "runMain vexiiriscv.soc.micro.MicroSocSim"

# List simulation arguments
sbt "runMain vexiiriscv.soc.micro.MicroSocSim --help"
```

#### Key Simulation Parameters

| Command             | Description                                     |
| ------------------- | ----------------------------------------------- |
| `--load-elf <file>` | Load ELF binary into RAM/ROM/Flash              |
| `--trace-fst`       | Dump full waveforms (GTKWAVE-compatible `.fst`) |
| `--trace-konata`    | Generate instruction trace for Konata viewer    |

To enhance performance:

```sh
--allow-bypass-from=0 --with-rvm --with-btb --with-ras --with-gshare
```

While simulation runs, connect using:

```sh
openocd -f src/main/tcl/openocd/vexiiriscv_sim.tcl
```

---

### Compiling and Running C/C++ Firmware

Use the official CMake-based firmware repo:

```sh
git clone https://github.com/SpinalHDL/VexiiFirmware.git
cd VexiiFirmware
export VEXII_FIRMWARE=$PWD

cmake -S . -B build -DSOC=microsoc/default -DDEVICE=microsoc_sim
make -C build example-uart
```

* `SOC=microsoc/default` selects SoC config
* `DEVICE=microsoc_sim` targets simulation runtime

Run simulation with your firmware ELF:

```sh
cd $VEXIIRISCV
sbt "runMain vexiiriscv.soc.micro.MicroSocSim --load-elf $VEXII_FIRMWARE/build/app/uart/example-uart.elf --regfile-async --allow-bypass-from=0"
```

Optional: Add `--trace-fst --trace-konata` for waveform dumps.

Expected output:

```txt
[info] [Progress] Start MicroSocSim test simulation with seed 42
[info] WAITING FOR TCP JTAG CONNECTION
[info] Hello Vexii!
...
```

---

### Adding a Custom Peripheral

MicroSoC includes an example peripheral called `PeripheralDemo.scala`:
📎 [https://github.com/SpinalHDL/VexiiRiscv/blob/dev/src/main/scala/vexiiriscv/soc/micro/PeripheralDemo.scala](https://github.com/SpinalHDL/VexiiRiscv/blob/dev/src/main/scala/vexiiriscv/soc/micro/PeripheralDemo.scala)

#### Components Used:

* `PeripheralDemo`: Actual peripheral logic (LEDs, buttons, interrupt)
* `mapper`: Easy peripheral register creation
* `BufferCC`: Handles button metastability (2-stage FF chain)
* `PeripheralDemoFiber`: Integration logic (TileLink bus, MMIO, GPIO signals)

#### Integration Example:

```scala
val demo = new PeripheralDemoFiber(new PeripheralDemoParam(12,16))
demo.node at 0x10003000 of bus32
plic.mapUpInterrupt(3, demo.interrupt)
```

#### Enable Peripheral in Simulation:

```sh
sbt "runMain vexiiriscv.soc.micro.MicroSocSim --demo-peripheral leds=16,buttons=12"
```
<img width="940" height="578" alt="image" src="https://github.com/user-attachments/assets/61cb2242-d3b1-45f5-b776-5b90c54b3673" />

🖼️ You can view the LED/button peripheral in GTKWave via trace files.

# VexiiRiscv FPGA Import Guide

This document describes how to take the VexiiRiscv-generated RTL and program memory, modify them for FPGA compatibility, and simulate or synthesize using Vivado.

> ✅ **Host OS**: Ubuntu 22.04 LTS running on Oracle VirtualBox (on Windows host)

---

## 🧩 Step 1: SBT Generation and Output Files

After generating the SoC using the appropriate SBT command:

```bash
sbt "runMain vexriscv.demo.MuraxWithRamInit"
```

You will find new files in the VexiiRiscv output directory:

* `MicroSoc.v`
* `MicroSoc.v_toplevel_system_ram_thread_logic_mem_symbol<X>.bin`

Where `<X>` varies based on the internal RAM size (usually from 0 to N, each being one byte lane).

---

## 🛠 Step 2: Preparing for Vivado Import

To import this design into Vivado for FPGA simulation or synthesis, a few changes are needed in `MicroSoc.v`.

### 🔧 Modifications Required

1. **Force RAM synthesis as block memory**
   Add this line before every RAM `reg` declaration:

   ```verilog
   (* ram_style = "block" *)
   ```

2. **Switch from `readmemb` to `readmemh`**
   Change the memory initialization lines like this:

   ```verilog
   // Before
   <img width="940" height="508" alt="image" src="https://github.com/user-attachments/assets/f23eb1e1-e2fc-46c2-9b22-26444a10a9ff" />


   // After
   <img width="940" height="503" alt="image" src="https://github.com/user-attachments/assets/ad88bb1b-c985-46e0-8b72-5c056558c0ba" />

   ```

3. **Update the memory file names**
   The generated `.bin` files need to be converted to `.mem` format compatible with Vivado.

---

## 🔁 Step 3: Converting `.bin` to `.mem` for Vivado

Use the Python script below to convert the ASCII binary files to hex `.mem` format:

```python
# convert_bin_to_mem.py

input_file = "Vexriscv/MicroSoc.v_toplevel_system_ram_thread_logic_mem_symbol3.bin"
output_file = "Vexriscv/program3.mem"
binary_output_file = "output.bin"  # Optional

with open(input_file, "r") as fin:
    lines = fin.readlines()

bytes_list = []
for line in lines:
    bin_str = line.strip()
    if bin_str:
        if len(bin_str) != 8 or any(c not in '01' for c in bin_str):
            raise ValueError(f"Invalid binary string: {bin_str}")
        value = int(bin_str, 2)
        bytes_list.append(value)

with open(output_file, "w") as fout:
    for b in bytes_list:
        fout.write(f"{b:02X}\n")

with open(binary_output_file, "wb") as fbin:
    fbin.write(bytearray(bytes_list))

print(f"Converted {len(bytes_list)} binary lines to .mem and .bin formats.")
```

---

## ✅ Final Step: Vivado Simulation or Synthesis

* Import `MicroSoc.v` into Vivado.
* Add all the `.mem` files (`program0.mem`, `program1.mem`, etc.) as simulation sources.
* Proceed with simulation or bitstream generation.

---

## 📘 Additional Resources

For a detailed simulation and firmware guide, refer to:

📘 [VexiiRiscv Simulation Tutorial](VexiiRiscv-Simulation-Tutorial.md)

> ✅ You now have a VexiiRiscv-based SoC with initialized RAM ready for FPGA execution.



---

### Tool Info

- **Tool Version**: Vivado v2024.1 (win64) Build 5076996  
- **Date**: Thu Jul 10 15:10:40 2025  
- **Host**: GRAVITY (64-bit major release, build 9200)  
- **Command**: `report_utilization -file MicroSoc_utilization_placed.rpt -pb MicroSoc_utilization_placed.pb`  
- **Design**: `MicroSoc`  
- **Device**: `xc7a35tcpg236-1`  
- **Speed File**: `-1`  
- **Design State**: `Fully Placed`

---

## Post-Placement Resource Utilization

This report confirms the proper utilization of **Block RAM** (BRAM), LUTs, Registers, DSPs, IOs, and other critical FPGA resources. You can analyze these reports at each implementation stage (synthesis, placement, routing) to verify memory and logic usage.

### Key Highlights

- **LUTs Used**: 2290 of 20800 (≈11.01%)
- **Slice Registers**: 2148 of 41600 (≈5.16%)
- **Block RAM Tiles**: 9 of 50 (≈18.00%)
  - RAMB36/FIFO: 8
  - RAMB18: 2
- **DSPs Used**: 4 of 90 (≈4.44%)
- **Bonded IO Blocks**: 32 of 106 (≈30.19%)
- **Clocking (BUFGCTRL)**: 1 of 32 (≈3.13%)

---

## Next Steps for FPGA Programming

Once the `.xdc` file is **properly matched** to the design ports (clocks, LEDs, switches, etc.), the core is ready for deployment onto the FPGA.

### Simulation Note

- **Behavioral Simulation** in Vivado will **visually show** the results, including memory contents.
- **Post-Synthesis Simulation**, however, will **not reflect the `.mem` contents**, since it doesn’t account for memory initialization.
- Despite that, the **`.mem` is embedded in the bitstream**, so **you do not need to manually program the memory** on FPGA.

---

## Clock Frequency Considerations

If your C program (running on the softcore) blinks an LED in every loop (or every few loops), the blink might not be visible due to the **high clock frequency**.

### Recommended Strategy:

- **Do not reduce the clock frequency globally**, as it is a poor workaround.
- Instead, **increase the delay loop count** inside the C code to make the blink visible.
- Be cautious: this will also **increase simulation runtime**, especially in behavioral sim, so find a **balance** based on your test goals.

---

## Final Output

If all the above aspects are handled correctly—utilization, constraint matching, simulation expectations, and clock behavior—you'll achieve a **functionally correct FPGA implementation** of your custom softcore system, visible both in simulation and on real hardware.

---
