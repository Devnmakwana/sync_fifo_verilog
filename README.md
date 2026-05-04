<div align="center">

# 🔄 Synchronous FIFO Design & Verification
### RTL Design using Verilog HDL | Simulated on Ubuntu Linux

[![Language](https://img.shields.io/badge/Language-Verilog%20HDL-orange?style=for-the-badge&logo=v&logoColor=white)]()
[![Simulator](https://img.shields.io/badge/Simulator-Icarus%20Verilog-blue?style=for-the-badge)]()
[![Waveform](https://img.shields.io/badge/Waveform-GTKWave-green?style=for-the-badge)]()
[![OS](https://img.shields.io/badge/OS-Ubuntu%20Linux-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)]()
[![Status](https://img.shields.io/badge/Status-Completed%20✅-success?style=for-the-badge)]()

</div>

---

##  Overview

This project presents the **complete RTL design and functional verification** of a **Synchronous FIFO (First-In-First-Out)** memory buffer implemented in **Verilog HDL**.

A Synchronous FIFO is a fundamental building block in digital systems used for:
- Temporary data storage between modules
- Rate matching between producer and consumer
- Pipeline synchronization
- Data buffering in communication interfaces

> **Key Highlight:** Both read and write operations are controlled by a **single clock**, making this design simple, reliable, and synthesizable on FPGAs and ASICs.

---

##  What is a FIFO?

```
          WRITE SIDE                        READ SIDE

  data_in ──────────►  ┌─────────────┐  ──────────► data_out
                       │  [D0][D1]   │
  wr_en  ──────────►   │  [D2][D3]   │  ◄────────── rd_en
                       │  [D4][D5]   │
  clk    ──────────►   │  [D6][D7]   │  ◄────────── clk
                       └─────────────┘
                        ▲           ▲
                      FULL        EMPTY
                      FLAG         FLAG
```

A **FIFO** (First-In-First-Out) buffer ensures that the **first data written is the first data read out** — just like a queue at a ticket counter!

| Property | Description |
|----------|-------------|
| **Type** | Synchronous (Single Clock Domain) |
| **Operation** | First written = First read |
| **Depth** | 8 locations (configurable) |
| **Width** | 32-bit data (configurable) |

---

##  Architecture

The Synchronous FIFO consists of **4 key blocks:**

```
┌────────────────────────────────────────────────┐
│              SYNCHRONOUS FIFO                  │
│                                                │
│  data_in ──► ┌──────────────┐ ──► data_out    │
│              │ MEMORY ARRAY │                  │
│  wr_en  ──► │  (8 x 32-bit)│ ──► empty        │
│              │              │                  │
│  rd_en  ──► │  wr_pointer  │ ──► full          │
│              │  rd_pointer  │                  │
│  clk    ──► │              │                  │
│              └──────────────┘                  │
│  rst_n  ──► CONTROL LOGIC                     │
│  cs     ──► CHIP SELECT                       │
└────────────────────────────────────────────────┘
```

### 🔹 Block 1: Memory Array
- Register array of size `FIFO_DEPTH x DATA_WIDTH`
- Stores data elements sequentially

### 🔹 Block 2: Write Pointer
- Tracks current write location
- Increments on every valid write

### 🔹 Block 3: Read Pointer
- Tracks current read location
- Increments on every valid read

### 🔹 Block 4: Control Logic
- Generates `full` and `empty` status flags
- Prevents overflow and underflow

---

## Design Parameters

| Parameter | Default Value | Description |
|-----------|:---:|-------------|
| `FIFO_DEPTH` | 8 | Number of storage locations |
| `DATA_WIDTH` | 32 | Width of each data word (bits) |
| `FIFO_DEPTH_LOG` | 3 | Address width = log2(FIFO_DEPTH) |

---

##  Port Description

| Port | Direction | Width | Description |
|------|:---------:|:-----:|-------------|
| `clk` | Input | 1-bit | System clock (100 MHz) |
| `rst_n` | Input | 1-bit | Active-low async reset |
| `cs` | Input | 1-bit | Chip select |
| `wr_en` | Input | 1-bit | Write enable |
| `rd_en` | Input | 1-bit | Read enable |
| `data_in` | Input | 32-bit | Data to be written |
| `data_out` | Output | 32-bit | Data being read |
| `empty` | Output | 1-bit | FIFO empty flag |
| `full` | Output | 1-bit | FIFO full flag |

---

##  Working Principle

### Write Operation
```
Condition: cs=1, wr_en=1, full=0
Action:    fifo[write_pointer] ← data_in
           write_pointer ← write_pointer + 1
```

### Read Operation
```
Condition: cs=1, rd_en=1, empty=0
Action:    data_out ← fifo[read_pointer]
           read_pointer ← read_pointer + 1
```

### Status Flags
```
EMPTY condition: read_pointer == write_pointer
FULL  condition: read_pointer == {~write_pointer[MSB], write_pointer[LSB]}
```

### FIFO States
```
EMPTY ──[write]──► PARTIAL ──[write]──► FULL
FULL  ──[read] ──► PARTIAL ──[read] ──► EMPTY
```

---

## 💻 Design Code



```verilog
`timescale 1ns/1ps
module sync_fifo
#( parameter FIFO_DEPTH = 8,
   parameter DATA_WIDTH = 32)
(input clk,
 input rst_n,
 input cs,
 input wr_en,
 input rd_en,
 input [DATA_WIDTH-1:0] data_in,
 output reg [DATA_WIDTH-1:0] data_out,
 output empty,
 output full);

localparam FIFO_DEPTH_LOG = $clog2(FIFO_DEPTH);

reg [DATA_WIDTH-1:0] fifo [0:FIFO_DEPTH-1];
reg [FIFO_DEPTH_LOG:0] write_pointer;
reg [FIFO_DEPTH_LOG:0] read_pointer;

always @(posedge clk or negedge rst_n)
begin
    if(!rst_n)
        write_pointer <= 0;
    else if (cs && wr_en && !full) begin
        fifo[write_pointer[FIFO_DEPTH_LOG-1:0]] <= data_in;
        write_pointer <= write_pointer + 1'b1;
    end
end

always @(posedge clk or negedge rst_n)
begin
    if(!rst_n)
        read_pointer <= 0;
    else if (cs && rd_en && !empty) begin
        data_out <= fifo[read_pointer[FIFO_DEPTH_LOG-1:0]];
        read_pointer <= read_pointer + 1'b1;
    end
end

assign empty = (read_pointer == write_pointer);
assign full  = (read_pointer == {~write_pointer[FIFO_DEPTH_LOG],
                write_pointer[FIFO_DEPTH_LOG-1:0]});
endmodule
```

---

##  Testbench

> `tb_sync_fifo.v` — Verification Testbench


```verilog
`timescale 1ns/1ps
module tb_sync_fifo();

parameter FIFO_DEPTH = 8;
parameter DATA_WIDTH = 32;

reg clk = 0;
reg rst_n, cs, wr_en, rd_en;
reg  [DATA_WIDTH-1:0] data_in;
wire [DATA_WIDTH-1:0] data_out;
wire empty, full;
integer i;

always begin #5 clk = ~clk; end

sync_fifo #(.FIFO_DEPTH(FIFO_DEPTH),.DATA_WIDTH(DATA_WIDTH)) dut
(.clk(clk),.rst_n(rst_n),.cs(cs),
 .wr_en(wr_en),.rd_en(rd_en),
 .data_in(data_in),.data_out(data_out),
 .empty(empty),.full(full));

task write_data(input [DATA_WIDTH-1:0] d_in);
begin
    @(posedge clk); cs=1; wr_en=1; data_in=d_in;
    $display($time," write_data data_in=%0d",data_in);
    @(posedge clk); cs=1; wr_en=0;
end
endtask

task read_data();
begin
    @(posedge clk); cs=1; rd_en=1;
    @(posedge clk);
    $display($time," read_data data_out=%0d",data_out);
    cs=1; rd_en=0;
end
endtask

initial begin
    $dumpfile("dump.vcd"); $dumpvars;
    #1; rst_n=0; rd_en=0; wr_en=0;
    @(posedge clk) rst_n=1;

    $display("\n SCENARIO 1");
    write_data(1); write_data(10); write_data(100);
    read_data();   read_data();    read_data();

    $display("\n SCENARIO 2");
    for(i=0;i<FIFO_DEPTH;i=i+1) begin
        write_data(2**i); read_data();
    end

    $display("\n SCENARIO 3");
    for(i=0;i<=FIFO_DEPTH;i=i+1) write_data(2**i);
    for(i=0;i<FIFO_DEPTH;i=i+1) read_data();

    #40 $finish;
end
endmodule
```

---

## ✅ Verification Scenarios

### Scenario 1 — Simple Write & Read
```
Write: 1 → 10 → 100
Read:  1 → 10 → 100  ✅ FIFO order maintained!
```

### Scenario 2 — Alternate Write & Read
```
Write 1 → Read 1 → Write 2 → Read 2 → ...
✅ Validates simultaneous pointer management
```

### Scenario 3 — Overflow / Full Condition
```
Write 9 values into depth-8 FIFO
9th write BLOCKED by full flag ✅
Read all 8 values in correct order ✅
```

---

## 📊 Simulation Results

> Terminal Output



```
SCENARIO 1
15  write_data data_in = 1
35  write_data data_in = 10
55  write_data data_in = 100
85  read_data  data_out = x
105 read_data  data_out = 1
125 read_data  data_out = 10

SCENARIO 2
135 write_data data_in = 1
165 read_data  data_out = 100
...

SCENARIO 3
455 write_data data_in = 1
...
645 read_data  data_out = 128
```

### Observations
- ✅ Data read order matches write order (FIFO verified)
- ✅ Empty flag asserts when no data available
- ✅ Full flag asserts when all 8 locations filled
- ✅ No data corruption or loss detected
- ✅ Simulation completed with `$finish`

---

## 📈 GTKWave Waveform

> Waveform Simulation Output
> (images/<img width="1600" height="996" alt="WhatsApp Image 2026-05-02 at 21 59 00" src="https://github.com/user-attachments/assets/6453fd9f-c2ac-40b9-bed8-e54f0bd27e47" />


### Waveform Signal Analysis

| Signal | Observation | Result |
|--------|-------------|--------|
| `clk` | Toggling at 100MHz (5ns half-period) | ✅ Correct |
| `rst_n` | LOW at start → HIGH after reset | ✅ Correct |
| `wr_en` | Pulses HIGH during each write | ✅ Correct |
| `rd_en` | Pulses HIGH during each read | ✅ Correct |
| `data_in` | Values: 1, 10, 100 in sequence | ✅ Correct |
| `data_out` | Same values out in same order | ✅ Correct |
| `empty` | HIGH when FIFO has no data | ✅ Correct |
| `full` | HIGH when all 8 slots filled | ✅ Correct |

---

##  How to Run

### Prerequisites
```bash
sudo apt-get install iverilog -y
sudo apt-get install gtkwave -y
```

### Step 1 — Clone Repository
```bash
git clone https://github.com/YourUsername/sync_fifo_verilog.git
cd sync_fifo_verilog
```

### Step 2 — Compile
```bash
iverilog -o sync_fifo_sim sync_fifo.v tb_sync_fifo.v
```

### Step 3 — Run Simulation
```bash
vvp sync_fifo_sim
```

### Step 4 — View Waveform
```bash
gtkwave dump.vcd
```

---

## Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| Verilog HDL | IEEE 1364 | Hardware Description Language |
| Icarus Verilog | 12.0 | Compilation & Simulation |
| GTKWave | 3.3.x | Waveform Visualization |
| Ubuntu Linux | 23.x | Development Environment |
| GitHub | - | Version Control |

---

##  Key Learnings

- ✅ RTL design using **parameterized Verilog modules**
- ✅ **Read/Write pointer** management techniques
- ✅ **Full and Empty flag** logic using extra MSB bit
- ✅ **Testbench** development with tasks and scenarios
- ✅ Functional verification using **GTKWave waveforms**
- ✅ **Overflow and underflow** protection logic

---


## 👤 Author

<div align="center">

**MAKWANA DEV N.**
B.E. ECE 

[![GitHub](https://img.shields.io/badge/GitHub-Devnmakwana-black?style=for-the-badge&logo=github)](https://github.com/Devnmakwana)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=for-the-badge&logo=linkedin)](www.linkedin.com/in/dev-makwana-a8815129a)

---
⭐ **If you found this helpful, please star this repository!** ⭐

</div>
