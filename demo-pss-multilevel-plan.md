# Demo: PSS Multi-Level Verification — Implementation Plan

*"One Scenario File. Every Stage. No Rewrites."*

---

## Overview

This demo is **Entry Point 2: Multi-Level Verification**. It targets verification
engineers and SoC integration leads. The core claim it demonstrates is:

> A PSS scenario authored once runs unchanged against a pure-Python behavioral
> model (sub-second) and a UVM/RTL simulation under Verilator, finding the same
> bugs at both levels with identical pass/fail semantics.

This demo is **fully standalone**. It does not depend on EP1 (greenfield
synthesis) or EP3 (brownfield acceleration). All tools are open-source.

---

## The Design Subject

The existing **OpenCores Wishbone DMA** (`examples/dma/verilog/wb_dma_top.v`),
configured for 4 channels. This is a real, non-trivial design that verifiers
immediately recognize as representative of IP they actually work on.

Using the existing RTL serves the demo narrative directly: the audience is
positioned as a verification team *receiving* the design, not as a design team
authoring it. The WB register sequence complexity (CSR, TXSZ, ADR0, ADR1, AM0,
AM1 per channel) is a feature, not a liability — PSS abstracts that complexity
into a single `dma_xfer_a` action, which is the point of the demo.

### wb_dma Interface Summary (4-channel configuration)

```
wb_dma_top  (ch_count=4, ch0_conf=4'h1, ch1_conf=4'h1,
              ch2_conf=4'h1, ch3_conf=4'h1, pri_sel=2'h0)

Wishbone Slave (CSR access):
  wb0s_data_i/o, wb0_addr_i, wb0_sel_i, wb0_we_i, wb0_cyc_i,
  wb0_stb_i, wb0_ack_o, wb0_err_o, wb0_rty_o

Wishbone Master (DMA data bus):
  wb0m_data_i/o, wb0_addr_o, wb0_sel_o, wb0_we_o, wb0_cyc_o,
  wb0_stb_o, wb0_ack_i, wb0_err_i, wb0_rty_i

Interrupts: inta_o (channel 0–3 OR'd), intb_o
Next-descriptor: nd0_i–nd3_i (tied low for non-linked-list transfers)
```

### wb_dma Register Map (relevant subset for 4-channel demo)

```
Word addr  | Register        | Notes
-----------|-----------------|---------------------------------------
0x00       | Global CSR      | [0]=disable all, [1]=pause_all
0x01       | INT_MASK_A      | per-channel interrupt mask
0x03       | INT_SRC_A       | per-channel interrupt source (W1C)
0x08       | CH0_CSR         | [0]=ch_en, [1]=IRL, [4]=BW mode, [28]=BUSY, [29]=ERR, [30]=DONE
0x09       | CH0_TXSZ        | [11:0]=transfer size (words), [27:24]=CH_TOC
0x0A       | CH0_ADR0        | source address
0x0C       | CH0_ADR1        | destination address
0x10–0x17  | CH1_*           | (same layout as CH0)
0x18–0x1F  | CH2_*
0x20–0x27  | CH3_*
```

The PSS adapter encapsulates this register sequence completely. The PSS
scenario never sees register addresses — it sees `dma_xfer_a` with channel,
src, dst, length fields.

---

## Demo Acts

### Act 1 — PSS Into the Existing UVM Testbench (8 min)

The engineer shows the UVM testbench for the wb_dma. It has a WB initiator
agent, a WB memory slave agent, a scoreboard, and the standard UVM test
hierarchy. Then they show the *two* changes needed to add PSS:

```systemverilog
// In dma_tb_top.sv — two imports added, nothing else changed
import "DPI-C" function void zuspec_pss_init(string scenario_lib);
import "DPI-C" function void zuspec_pss_run(string scenario_name);
```

```systemverilog
// New UVM test class — minimal boilerplate
class dma_pss_test extends uvm_test;
  task run_phase(uvm_phase phase);
    phase.raise_objection(this);
    zuspec_pss_init("./pss/dma_pkg.pss");
    zuspec_pss_run("s_dual_concurrent");
    phase.drop_objection(this);
  endtask
endclass
```

Run it:

```
$ dfm run sim-run-dual
[uvm] Compiling wb_dma + UVM testbench under Verilator...
[uvm] UVM_INFO: Running dma_pss_test
[uvm] [ZUSPEC] Loaded: s_dual_concurrent
[uvm] [ZUSPEC] CH0: 256 transfers — PASS
[uvm] [ZUSPEC] CH1: 256 transfers — PASS
[uvm] [ZUSPEC] Fairness delta: 0 (threshold 4) — PASS
[uvm] UVM_INFO: TEST PASSED
[uvm] Runtime: 22.4 s
```

### Act 2 — Same Scenario at Algorithmic Level (0.3 s)

Switch target to the Python behavioral model without changing the scenario:

```
$ dfm run python-dual
[python] CH0: 256 transfers — PASS
[python] CH1: 256 transfers — PASS
[python] Fairness delta: 0 — PASS
[python] Runtime: 0.31 s
```

**The visual**: scenario file unchanged, runtime drops from 22s to 0.3s.

### Act 3 — Starvation Bug Found Fast at Algorithmic Level

Run the starvation scenario. It exposes a round-robin fairness violation when
CH0 and CH1 operate in burst mode:

```
$ dfm run python-stress
[python] VIOLATION: CH2 completed 0 transfers while CH0+CH1 saturating
[python] Root cause: round-robin arbiter does not bound wait time under burst load
[python] Runtime: 0.4 s

$ dfm run sim-run-stress
[uvm]   VIOLATION: CH2 completed 0 transfers (same root cause)
[uvm]   Runtime: 31.2 s
```

Same violation, same root cause, 75× faster to find at the algorithmic level.

### Act 4 — Cross-Block Contract (Bonus, ~3 min)

Two behavioral models (DMA + minimal interrupt controller stub) under a
PSS scenario that constrains the interaction. Shows a protocol bug in the
INTC edge-conversion logic that block-level tests can never find.

---

## Directory Layout

```
demo/pss/
├── README.md                        # Cold-clone setup instructions
│
├── pss/                             # PSS scenario files (NEW)
│   ├── dma_pkg.pss                  # DMA component + action definitions
│   ├── s_single_channel.pss         # Smoke test: one channel, 16 sequential xfers
│   ├── s_dual_concurrent.pss        # Two channels, fairness constraint
│   └── s_bandwidth_stress.pss       # Four channels, starvation trigger
│
├── model/                           # Python behavioral model (NEW)
│   ├── dma_model.py                 # Asyncio DMA simulation
│   └── adapter_python.py            # PSS platform adapter for Python target
│
├── tb/                              # UVM testbench (NEW)
│   ├── wb_if.sv                     # Wishbone interface (SystemVerilog interface)
│   ├── wb_seq_item.sv               # UVM sequence item for WB transactions
│   ├── wb_driver.sv                 # UVM driver — drives wb_if from seq_item
│   ├── wb_monitor.sv                # UVM monitor — samples wb_if
│   ├── wb_agent.sv                  # UVM agent (driver + monitor + sequencer)
│   ├── wb_mem_slave.sv              # WB memory slave (pure Verilog, word-addressed SRAM)
│   ├── dma_env.sv                   # UVM env (initiator agent + mem slave + scoreboard)
│   ├── dma_scoreboard.sv            # Checks completed transfer count per channel
│   ├── dma_pss_bridge.sv            # DPI adapter: zuspec_pss_init/run → WB sequences
│   ├── dma_pss_test.sv              # UVM test class (two lines + raise/drop objection)
│   └── dma_tb_top.sv                # TB top: clock/reset, DUT, wb_mem_slave, UVM kickoff
│
├── adapter_uvm/                     # RTL/UVM PSS adapter (NEW)
│   └── adapter_uvm.py               # PSS platform adapter: xfer_a → DPI WB reg writes
│
└── flow.yaml                        # dfm workflow: each scenario×target as a task (NEW)
```

---

## Component Inventory

### What is **REUSED** (zero new work)

| Artifact | Location | Role |
|----------|----------|------|
| `wb_dma_top.v` + all submodules | `packages/zuspec/examples/dma/verilog/` | Design under verification — **used as-is** |
| `uvm_pkg.sv` + UVM base library | `packages/uvm/src/` | UVM runtime for testbench |
| `zuspec-fe-pss` | `packages/zuspec-fe-pss/` | PSS parser + IR compilation |
| `zuspec-dataclasses` | `packages/zuspec-dataclasses/` | PSS Python runtime (ScenarioRunner, randomize, do, parallel) |
| `pyhdl-if` | `packages/pyhdl-if/` | Python↔SV bridge: spawns Python from SV and provides call-back API |
| `zuspec-be-hdlsim` | `packages/zuspec-be-hdlsim/` | Verilator compile + launch infrastructure |
| WB BFM reference | `packages/pyhdl-if/examples/call/dpi/call_sv_bfm/wb_init_bfm.sv` | Reference for WB driver implementation |

> **Note on DMA RTL location**: The Verilog sources are currently inside the `packages/zuspec/`
> subtree (fetched by ivpm), but **must be copied locally** into the demo tree (e.g.
> `demo/pss/rtl/`). The `packages/zuspec/` content is a transient dependency that may be
> removed or altered; the demo must not depend on it directly. The `dma_c.pss` in
> `../zuspec-pss/` is a *different*, higher-abstraction DMA PSS model — do not confuse the two.

---

### What is **IMPLEMENTED** (new work)

#### 1. PSS Scenario Files — `demo/pss/pss/`

Four files. These are the central artifact of the demo — they do not change
between targets.

**`dma_pkg.pss`** — component definition and abstract action. The `exec body`
block is left empty here; it is filled in by each platform adapter at compile
time or via `extend`.

```pss
component dma_env_c {
    action xfer_a {
        rand bit[2]  channel;          // DMA channel 0–3
        rand bit[32] src;              // source word address
        rand bit[32] dst;              // destination word address
        rand bit[12] length;           // transfer length in 32-bit words

        constraint {
            channel < 4;
            length >= 1; length <= 256;
            src >= 32'h0001_0000; src < 32'h0002_0000;
            dst >= 32'h0002_0000; dst < 32'h0003_0000;
            // src and dst non-overlapping guaranteed by address range split
        }
    }
}
```

**`s_single_channel.pss`** — 16 sequential transfers on CH0. Used as the
smoke test. Expected: all 16 pass, destination memory matches source.

**`s_dual_concurrent.pss`** — CH0 and CH1 in a PSS `parallel` block, each
completing 256 transfers. Fairness constraint embedded in the scenario:

```pss
action s_dual_concurrent {
    activity {
        parallel {
            replicate(256) { do xfer_a with { channel == 0; }; }
            replicate(256) { do xfer_a with { channel == 1; }; }
        }
    }
    // Neither channel completes more than 4 ahead of the other
    constraint { abs(ch0_done - ch1_done) <= 4; }
}
```

**`s_bandwidth_stress.pss`** — All four channels, each requesting 256
transfers. CH0 and CH1 constrained to burst mode (`length == 64`); CH2 and
CH3 constrained to single-word mode (`length == 1`). No fairness constraint
is asserted — the scenario deliberately allows the starvation to manifest so
the runner can measure and report it.

---

#### 2. Python Behavioral Model — `demo/pss/model/`

**`dma_model.py`** — pure-Python asyncio simulation. No C, no simulator.
Models the round-robin arbiter as a shared `asyncio.Lock` with a per-channel
grant counter. A transfer is a coroutine that acquires the lock, counts
`length` memory-bus beats, releases the lock, and records completion.

```python
class DMAChannel:
    async def transfer(self, src: int, dst: int, length: int) -> None:
        """Simulate one DMA transfer. Acquires bus once per word."""
        for _ in range(length):
            async with self.arbiter.bus_lock:
                self.memory[dst] = self.memory.get(src, 0)
                src += 1; dst += 1
        self.completed += 1

class DMAModel:
    channels: list[DMAChannel]
    memory:   dict[int, int]   # flat word-addressed memory
```

Each class has an explicit docstring noting which Verilog module it mirrors.
This is the "readable model" talking point.

**`adapter_python.py`** — PSS platform adapter. Implements the `xfer_a`
exec body by calling `DMAModel.channels[channel].transfer(src, dst, length)`.
Uses `zuspec.dataclasses.ScenarioRunner` for scenario orchestration.

---

#### 3. UVM Testbench — `demo/pss/tb/`

A complete, self-contained UVM testbench. All files are new; no existing
testbench infrastructure is copied or modified.

**`wb_if.sv`** — SystemVerilog interface for the Wishbone slave port. Carries
clk, rst, and all WB signals. Used as the connection point between the DUT,
the UVM driver, the monitor, and the PSS bridge.

```systemverilog
interface wb_if (input logic clk, rst);
    logic [31:0] addr, wdata, rdata;
    logic [3:0]  sel;
    logic        we, cyc, stb, ack, err;
endinterface
```

**`wb_seq_item.sv`** — UVM sequence item carrying one WB transaction
(addr, data, we). Used by the driver and monitor.

**`wb_driver.sv`** — UVM driver. On each `get_next_item`, drives the WB
initiator signals (addr, wdata, sel, we, cyc, stb), waits for `ack`, then
calls `item_done`. Based structurally on `wb_init_bfm.sv` from `pyhdl-if`
examples but wrapped as a UVM component.

**`wb_monitor.sv`** — UVM monitor. Samples WB bus on `ack`, broadcasts a
cloned `wb_seq_item` on the analysis port. Used by the scoreboard.

**`wb_agent.sv`** — UVM agent. Instantiates driver + monitor + sequencer.

**`wb_mem_slave.sv`** — Standalone Verilog module (not a UVM component).
Implements a Wishbone slave responding to the DMA master port. Contains a
`reg [31:0] mem [0:MEM_WORDS-1]` array pre-initialised with a deterministic
pattern. This is the memory the DMA reads from and writes to.

```verilog
module wb_mem_slave #(parameter MEM_WORDS = 65536) (
    input         clk, rst,
    input  [31:0] wb_addr_i,
    input  [31:0] wb_data_i,
    output [31:0] wb_data_o,
    input  [3:0]  wb_sel_i,
    input         wb_we_i, wb_cyc_i, wb_stb_i,
    output        wb_ack_o
);
    integer i; initial for (i=0; i<MEM_WORDS; i=i+1) mem[i] = i;
```

**`dma_scoreboard.sv`** — UVM scoreboard. Listens to the WB monitor's
analysis port. Tracks writes to `INT_SRC_A` (interrupt source register) to
count per-channel transfer completions. Reports mismatch if expected
completion counts are not reached. Also reports per-channel completion deltas
to enable the starvation story.

**`dma_pss_bridge.sv`** — The critical integration piece. Exports two DPI
functions (`zuspec_pss_init`, `zuspec_pss_run`) callable from `run_phase`.
On the inbound side (Python → SV), exposes two DPI tasks for register access
(`dpi_wb_write`, `dpi_wb_read`) that create `wb_seq_item` instances and send
them via the WB agent's sequencer. The bridge holds a handle to the
`wb_sequencer` populated by the env.

> **Integration mechanism — pyhdl-if, not raw DPI**:
> The Python PSS runtime is launched *by* the SV testbench via `pyhdl-if`'s
> spawning infrastructure (not via hand-rolled `export "DPI-C"` tasks).
> pyhdl-if handles all DPI plumbing internally; the Python adapter calls back
> into SV (e.g., `wb_write`/`wb_read`) using pyhdl-if's registered-task API.
> The explicit DPI function signatures below are illustrative of the interface
> contract, not necessarily the literal SV source.

```systemverilog
// pyhdl-if spawns Python; Python calls these back via the pyhdl-if API
task dpi_wb_write(input int unsigned addr, input int unsigned data);
task dpi_wb_read (input int unsigned addr, output int unsigned data);
```

The PSS adapter (`adapter_uvm.py`) calls `wb_write` / `wb_read` through
`pyhdl-if`, which routes the call back into the UVM WB agent's sequencer.
This preserves the UVM sequence layer — the scoreboard sees every transaction.

> **Open question**: Exact pyhdl-if API for registering call-back tasks and
> spawning Python from SV `run_phase` — consult
> `packages/pyhdl-if/examples/call/dpi/call_sv_bfm/` as the reference pattern.

**`dma_pss_test.sv`** — One UVM test class. The entire body is the two-line
PSS invocation shown in Act 1.

**`dma_tb_top.sv`** — Testbench top. Instantiates `wb_dma_top` (4-channel
configuration), `wb_mem_slave`, and the WB interfaces. Generates clock and
reset. Calls `run_test()` to start UVM.

---

#### 4. UVM PSS Adapter — `demo/pss/adapter_uvm/`

**`adapter_uvm.py`** — PSS platform adapter for the UVM/Verilator target.
The `xfer_a` exec body maps to this sequence of register writes via DPI:

```python
async def exec_xfer(action):
    ch = action.channel
    base = 0x20 + ch * 0x20   # word address; multiply by 4 for byte addr

    await wb_write(base + 0x02, action.src)           # CH_ADR0 = source
    await wb_write(base + 0x04, action.dst)           # CH_ADR1 = destination
    await wb_write(base + 0x01,                       # CH_TXSZ
        (action.length & 0xFFF) | (0b0001 << 24))     # size + single-block mode
    await wb_write(base + 0x00,                       # CH_CSR: enable + go
        (1 << 0) | (1 << 2))                          # CH_EN + CH_IRL

    # Poll CH_CSR[30] (DONE bit) or wait for INTA interrupt
    while True:
        csr = await wb_read(base + 0x00)
        if csr & (1 << 30): break
        await sim_delay(10)   # yield 10 clock edges
```

This adapter is the most implementation-intensive piece, but it is also the
most direct illustration of what PSS abstracts away. The scenario author
never sees any of this code.

---

#### 5. Demo Runner — `demo/pss/flow.yaml` (dfm workflow)

> **Orchestration uses `dfm`** (the `dv-flow-mgr` workflow tool, installed at
> `packages/python/bin/dfm`). Each demo target and scenario is captured as a
> task in `flow.yaml`. Select simulator with `-D sim=<vlt|vcs|xcm|mti>`;
> default is `vlt` (Verilator).

```bash
dfm run sim-run-single            # run s_single_channel on default (vlt) sim
dfm run sim-run-single -D sim=vcs # run with VCS
dfm run python-single             # run s_single_channel on Python model (no sim)
dfm run all                       # run all scenarios × all targets
```

The `flow.yaml` structure follows the pattern established in
`packages/pyhdl-if/examples/uvm/component_proxy_smoke/flow.yaml`:

```yaml
# yaml-language-server: $schema=https://dv-flow.github.io/flow.dv.schema.json
package:
  name: zuspec-example-pss-dma
  with:
    sim:
      type: str
      value: "vlt"          # override: -D sim=vcs|xcm|mti|vlt

  tasks:
  # ── Shared sources ──────────────────────────────────────────────────────
  - name: pythonpath
    uses: std.SetEnv
    with:
      prepend_path:
        PYTHONPATH: ${{ rootdir }}

  - name: rtl-src
    uses: std.FileSet
    with:
      type: verilogSource
      include: ["packages/zuspec/examples/dma/verilog/*.v"]

  - name: tb-src
    uses: std.FileSet
    with:
      type: systemVerilogSource
      include: ["tb/*.sv"]

  # ── UVM simulation image (simulator-agnostic via ${{ sim }}) ────────────
  - name: sim-img
    uses: "hdlsim.${{ sim }}.SimImage"
    needs:
    - rtl-src
    - tb-src
    - "hdlsim.${{ sim }}.SimLibUVM"   # UVM pre-compiled library
    - pyhdl-if.UVMPkg                  # pyhdl-if UVM integration
    - pyhdl-if.DpiLib                  # pyhdl-if DPI C library
    with:
      top: [dma_tb_top]

  # ── Per-scenario UVM simulation runs ────────────────────────────────────
  - root: sim-run-single
    uses: "hdlsim.${{ sim }}.SimRun"
    needs: [pythonpath, sim-img]
    with:
      plusargs: ["UVM_TESTNAME=dma_pss_test", "pss_scenario=s_single_channel"]

  - root: sim-run-dual
    uses: "hdlsim.${{ sim }}.SimRun"
    needs: [pythonpath, sim-img]
    with:
      plusargs: ["UVM_TESTNAME=dma_pss_test", "pss_scenario=s_dual_concurrent"]

  - root: sim-run-stress
    uses: "hdlsim.${{ sim }}.SimRun"
    needs: [pythonpath, sim-img]
    with:
      plusargs: ["UVM_TESTNAME=dma_pss_test", "pss_scenario=s_bandwidth_stress"]

  # ── Python-only (no simulator) runs ─────────────────────────────────────
  - root: python-single
    needs: [pythonpath]
    shell: bash
    run: python model/run_scenario.py s_single_channel

  - root: python-dual
    needs: [pythonpath]
    shell: bash
    run: python model/run_scenario.py s_dual_concurrent

  - root: python-stress
    needs: [pythonpath]
    shell: bash
    run: python model/run_scenario.py s_bandwidth_stress

  # ── Aggregate target ─────────────────────────────────────────────────────
  - root: all
    shell: bash
    needs:
    - python-single
    - python-dual
    - python-stress
    - sim-run-single
    - sim-run-dual
    - sim-run-stress
    run: echo "All scenarios passed on all targets"
```

**Key dfm patterns used** (confirmed via `dfm show task`):
- `"hdlsim.${{ sim }}.SimImage"` — simulator-agnostic compile/elab; supports
  `vlt`, `vcs`, `xcm`, `mti` (all extend `hdlsim.SimImage`)
- `"hdlsim.${{ sim }}.SimLibUVM"` — pre-compiled UVM library per simulator
- `pyhdl-if.UVMPkg` — pyhdl-if UVM integration SV package (needs `pyhdl-if.SvPkg`)
- `pyhdl-if.DpiLib` — pyhdl-if DPI C library (passed to `SimRun` via `needs`)
- `sim-img` is shared; each scenario run is a separate `SimRun` task pointing
  at the same compiled image

---

## Testbench Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│  dma_tb_top.sv                                          │
│                                                         │
│  ┌─────────────────────┐    ┌──────────────────────┐   │
│  │  wb_dma_top.v        │    │  wb_mem_slave.v       │   │
│  │  (4-channel DMA)     │◄──►│  (WB memory slave)   │   │
│  │                      │WBm │                      │   │
│  └──────────┬───────────┘    └──────────────────────┘   │
│             │ WBs (CSR)                                  │
│  ┌──────────▼───────────────────────────────────────┐   │
│  │  dma_env.sv  (UVM environment)                   │   │
│  │  ┌──────────────┐  ┌──────────────────────────┐  │   │
│  │  │  wb_agent.sv │  │  dma_scoreboard.sv        │  │   │
│  │  │  (initiator) │  │  (tracks completions)     │  │   │
│  │  └──────┬───────┘  └──────────────────────────┘  │   │
│  │         │ sequences                               │   │
│  │  ┌──────▼───────────────────────────────────┐    │   │
│  │  │  dma_pss_bridge.sv                        │    │   │
│  │  │  - exports: zuspec_pss_init/run (DPI)     │    │   │
│  │  │  - exports: dpi_wb_write/read (DPI)       │    │   │
│  │  └──────────────────┬────────────────────────┘    │   │
│  └─────────────────────│────────────────────────────-┘   │
└────────────────────────│─────────────────────────────────┘
                         │ DPI (pyhdl-if)
           ┌─────────────▼────────────────────┐
           │  Python PSS Runtime               │
           │  - zuspec-fe-pss (PSS parser)     │
           │  - zuspec-dataclasses (runner)    │
           │  - adapter_uvm.py (exec bodies)   │
           │  - dma_pkg.pss + scenario files   │
           └──────────────────────────────────┘
```

---

## The Starvation Bug

The `wb_dma`'s built-in arbitration operates in burst mode: once a channel is
granted the bus, it holds it for the entire transfer. With 4-channel round-robin
and CH0/CH1 each requesting 64-word transfers while CH2/CH3 request 1-word
transfers:

- Effective round-robin grant sequence: CH0(64 beats) → CH1(64 beats) →
  CH2(1 beat) → CH3(1 beat) → repeat
- CH2 and CH3 each receive only 1/130 of total bus bandwidth
- Under 256-transfer load, CH2/CH3 complete their 256 transfers only after
  CH0/CH1 complete ~16,500 transfers

**Python model**: surfaces in 0.4 s. The `asyncio.Lock` holding pattern
makes the starvation visible directly in coroutine scheduling.

**UVM/RTL**: surfaces in ~31 s. The scoreboard detects the disparity in
completion counts after the simulation runs.

**The fix** (in the Python model's `DMAModel.arbiter`, and verbally described
for the RTL): add a `MAX_BURST_BEATS` credit counter so no channel holds the
bus for more than N consecutive beats. Two lines in each model. Formally
demonstrable: re-run `s_bandwidth_stress` after the fix, confirm all channels
complete within a bounded ratio of each other.

---

## Toolchain Requirements

| Tool | Version | Install | Purpose |
|------|---------|---------|---------|
| Python | 3.11+ | system | PSS runtime, adapters, runner |
| Verilator | 5.x | ivpm → `packages/verilator/` | RTL simulation (locally installed, **not** system apt) |
| GCC / Clang | any modern | system | Verilator C++ compilation |
| zuspec-fe-pss | current | `packages/python/` venv (ivpm) | PSS parsing |
| zuspec-dataclasses | current | `packages/python/` venv (ivpm) | PSS Python runtime |
| pyhdl-if | current | `packages/python/` venv (ivpm) | Python↔SV bridge |
| zuspec-be-hdlsim | current | `packages/python/` venv (ivpm) | Verilator integration |
| UVM base library | included | `packages/uvm/src/` (ivpm) | UVM runtime (Apache-2.0) |
| dfm (dv-flow-mgr) | current | `packages/python/bin/dfm` (ivpm) | Workflow orchestration |

> **Setup**: This project uses `direnv` to manage the environment — run `direnv allow`
> once after cloning to activate it. The `.envrc` adds `packages/python/bin` and
> `packages/verilator/bin` to `PATH`, so `dfm`, `python`, and `verilator` are all
> available without further activation. Run `ivpm update` first to populate `packages/`.

No commercial EDA tools. No QEMU. The entire demo runs from a fresh clone.

---

## Implementation Sequence

Each step produces something independently runnable before the next begins.

1. **Verify wb_dma compiles standalone** under Verilator with a trivial
   directed testbench (clock/reset, tie DMA master to a simple ack-always
   memory model, write one register via the WB slave port). Smoke-test the
   existing RTL before building anything on top of it.

2. **Write Python behavioral model** (`dma_model.py`). Unit-test with a
   plain Python script (no PSS). Verify: single-channel transfer completes,
   data in memory matches expected, starvation appears under burst load.

3. **Write PSS package** (`dma_pkg.pss`). Verify it parses with
   `zuspec-fe-pss`. Write `adapter_python.py`. Run `s_single_channel`
   against the Python model.

4. **Write `s_dual_concurrent.pss`**. Run against Python model. Verify
   fairness constraint passes.

5. **Write `s_bandwidth_stress.pss`**. Run against Python model. Confirm
   starvation is detected and reported. Apply the burst-credit fix. Confirm
   the scenario passes after the fix.

6. **Write UVM testbench** (`wb_if.sv` through `dma_tb_top.sv`). Compile
   under Verilator with the UVM library. Run a directed UVM test (no PSS yet)
   to confirm the WB agent drives the DMA correctly and the scoreboard sees
   completions.

7. **Write `dma_pss_bridge.sv` and `adapter_uvm.py`**. Wire DPI exports.
   Run `s_single_channel` end-to-end under Verilator via PSS.

8. **Run all three scenarios against both targets**. Verify identical
   pass/fail results. Time both runs. Fix any latency or ordering issues in
   the adapter.

9. **Write `flow.yaml`** with all scenario × target tasks. Write `README.md`. Validate
   cold-clone from-scratch execution with `dfm run all`.

---

## Success Criteria

- `dfm run python-single` → PASS
- `dfm run sim-run-single` → PASS  (default sim=vlt; also passes with `-D sim=vcs|xcm|mti`)
- `dfm run python-dual` → PASS
- `dfm run sim-run-dual` → PASS
- `dfm run python-stress` → VIOLATION reported
- `dfm run sim-run-stress` → same VIOLATION
- `dfm run all` → all tasks complete, total runtime < 3 min on vlt
- The three `.pss` scenario files are bit-for-bit identical across both target runs
- Fresh-clone setup documented and tested: under 10 min from `git clone` to first PASS

> **Note**: `dfm` is the `dv-flow-mgr` workflow tool. Task names match the
> `flow.yaml` root tasks. Simulator is selected via `-D sim=vlt|vcs|xcm|mti`
> (default: `vlt`). The single shared `sim-img` task compiles once; each
> `sim-run-*` task reuses it.

---

## Out of Scope

- QEMU device plugin target — requires QEMU build infrastructure
- Formal/bounded model checking — the starvation story is told through
  deterministic scenario execution, not formal proof (which belongs in EP1)
- Cross-block integration contract (EP2 Act 4) — implementable as a short bonus
  act using a trivial interrupt controller Python stub (~60 lines); excluded
  from the required plan but noted here for future extension
- Coverage database and dashboard (EP2 Act 5) — stretch goal
