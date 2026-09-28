
# Time-Interleaved Spherical UKF Coprocessor for Multi-Cell Battery SOC Estimation

### Colored-Noise Modeling and Hardware-in-the-Loop Validation

FPGA-based hardware acceleration of multi-cell Lithium-ion battery State-of-Charge (SOC) estimation using a Spherical Simplex Unscented Kalman Filter (SUKF), time-interleaved hardware architecture, colored-noise modeling, and Hardware-in-the-Loop (HIL) validation.

**Author:** Aditya Mittal  
**Program:** B.Tech Electrical & Electronics Engineering  
**Platform:** Xilinx Zynq-7000 / XC7Z020  
**Tools:** MATLAB/Simulink, Vitis HLS, Vivado, Python, PYNQ[cite: 1]  

---

## Overview

Accurate State-of-Charge (SOC) estimation is essential for the safety, efficiency, and longevity of multi-cell lithium-ion battery systems used in electric vehicles and energy storage systems.

This project investigates a hardware-efficient implementation of a Spherical Simplex Unscented Kalman Filter (SUKF) for multi-cell battery SOC estimation.

The architecture combines:
- 2-RC equivalent-circuit battery modeling
- Spherical Simplex Unscented Kalman Filter
- AR(1) colored-noise modeling
- Time-interleaved multi-cell processing
- Innovation-gated scheduling
- Fixed-point arithmetic
- FPGA hardware acceleration
- Hardware-in-the-Loop validation

---
## Why This Project?

Every battery pack in an electric vehicle (EV) or energy storage system (ESS) contains dozens to thousands of individual battery cells. To ensure safety, prevent thermal runaway, and maximize battery lifespan, the Battery Management System (BMS) must constantly monitor the State-of-Charge (SOC)—essentially the "fuel gauge"—for **every single cell** in real time[cite: 2].

1. **Direct Measurement Is Impossible:** You cannot directly measure SOC with a sensor; it must be estimated mathematically using voltage, current, and temperature readings[cite: 2].
2. **Standard Filters Are Flawed:** Basic algorithms like Extended Kalman Filters (EKF) fail during sudden acceleration because batteries exhibit heavy electrochemical non-linearities[cite: 1, 2].
3. **Hardware & Cost Bottlenecks:** Advanced algorithms like the Unscented Kalman Filter (UKF) handle non-linearities well, but running a dedicated UKF circuit for every cell consumes far too much FPGA/ASIC hardware chip area and power[cite: 2].
4. **Real-World Noise Isn't Ideal:** Standard filters assume sensor noise is purely random ("white noise")[cite: 2]. Real EV environments suffer from temporally correlated ("colored") noise caused by thermal gradients, sensor drift, and electromagnetic interference (EMI) from motors[cite: 2].

This project resolves these bottlenecks by combining **algorithmic efficiency** with **hardware multiplexing**[cite: 1, 2].

---

## What Is This Project? (In Plain English)

Think of a battery pack with dozens of cells like a busy grocery store:

- **The Naïve Way (Parallel Hardware Replication):** Building a dedicated, expensive cash register (UKF filter circuit) for every single customer (battery cell)[cite: 2]. It works fast, but it wastes massive amounts of hardware space and power[cite: 2].
- **The Proposed Way (Time-Interleaved Coprocessor):** Building **one single, high-speed cash register** (shared SUKF datapath) that services customers one by one in rapid sequence[cite: 1, 2]. As each cell steps up, the system loads its current data from a fast memory cache (Context SRAM), updates its SOC estimate, saves the updated state back to memory, and moves to the next cell in microseconds[cite: 1, 2].

To make this single hardware register even faster and smarter:
- **Spherical Simplex Reduction:** We cut the mathematical steps required per cell by ~50% ($n+2$ sample points instead of $2n+1$)[cite: 2].
- **Smart Scheduling:** If a cell's voltage hasn't changed (e.g., sitting idle), the scheduler skips its math update for that turn, freeing compute bandwidth for active cells[cite: 1, 2].
- **Colored-Noise Defense:** We teach the filter to recognize and strip out correlated sensor drift and motor noise rather than being tricked by it[cite: 1, 2].

---

## What Am I Doing?

1. **Algorithm Development (MATLAB/Simulink):** Building a 2-RC equivalent circuit battery model augmented with an AR(1) state model to capture colored noise[cite: 1, 2]. Testing EKF, standard UKF, and Spherical Simplex UKF (SUKF) side-by-side[cite: 2].
2. **Hardware Coprocessor Design (Vitis HLS & C++):** Translating the SUKF mathematical execution pipeline into fixed-point hardware (`ap_fixed` arithmetic) optimized with unrolling and pipelining directives[cite: 1, 2].
3. **Multi-Cell System Architecture (Vivado & Verilog):** Designing the time-interleaving context-switching logic, on-chip Context SRAM registers, and an innovation-gated scheduler[cite: 1, 2].
4. **Real-Time HIL Validation (PYNQ & Zynq-7000 FPGA):** Streaming real, recorded EV driving dynamic currents (UDDS and US06 drive cycles) into the physical FPGA via AXI-DMA and logging real-time SOC estimates back to the host[cite: 1, 2].




## System Architecture

```text
                  ┌───────────────────────┐
                  │  MATLAB / Simulink    │
                  │  Drive-Cycle Data     │
                  └───────────┬───────────┘
                              │
                              │ AXI-DMA
                              ▼
                  ┌───────────────────────┐
                  │ Innovation-Gated      │
                  │ Scheduler             │
                  └───────────┬───────────┘
                              │
                              ▼
        ┌────────────────────────────────────────┐
        │       Time-Interleaved SUKF Datapath   │
        │                                        │
        │  ┌──────────────────────────────────┐  │
        │  │ Spherical Simplex Sigma Points   │  │
        │  └────────────────┬─────────────────┘  │
        │                   ↓                    │
        │  ┌──────────────────────────────────┐  │
        │  │ Nonlinear 2-RC State Propagation │  │
        │  └────────────────┬─────────────────┘  │
        │                   ↓                    │
        │  ┌──────────────────────────────────┐  │
        │  │ Colored-Noise / Cholesky Update  │  │
        │  └────────────────┬─────────────────┘  │
        │                   ↓                    │
        │  ┌──────────────────────────────────┐  │
        │  │ Measurement Correction           │  │
        │  └──────────────────────────────────┘  │
        └────────────────────┬───────────────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │ Context SRAM          │
                  │ Per-cell state        │
                  └───────────┬───────────┘
                              │
                              ▼
                       Cell SOC Output


```
### Core Architecture Concept

A conventional multi-cell implementation replicates a complete filter for every battery cell. This project instead uses a **single SUKF datapath shared across multiple channels**. Individual cell states and covariance matrices are stored in Context SRAM and swapped as each channel is serviced.

### Conventional (Parallel Replication)

```text

Cell 1 ──> UKF
Cell 2 ──> UKF
Cell 3 ──> UKF
Cell 4 ──> UKF
```

### Proposed (Time-Interleaved Datapath)

```text
Cell 1 ─┐
Cell 2 ─┤
Cell 3 ─┼──> Shared SUKF Datapath ──► Context SRAM
Cell 4 ─┘
```

---

# Spherical Simplex UKF (SUKF)

For an `n`-state system:

* **Standard UKF:** `2n + 1` sigma points
* **Spherical Simplex UKF:** `n + 2` sigma points

For the 3-state battery model (`n = 3`) used in this project:

* **Standard UKF:** 7 sigma points
* **SUKF:** 5 sigma points

This reduction lowers the computational load per iteration, facilitating real-time multi-cell multiplexing on hardware.

---

# Colored-Noise Modeling

Real battery sensor measurements exhibit temporally correlated noise caused by sensor drift, thermal gradients, and EMI coupling.

The project augments the state-space formulation with an **AR(1) process model**, allowing the SUKF to evaluate SOC tracking under correlated noise conditions.

---

# Innovation-Gated Scheduling

The scheduler inspects incoming cell voltage residuals. Channels with small innovation values bypass full covariance matrix updates, prioritizing execution bandwidth for dynamically active cells.

---

# Hardware Implementation

| Component / Feature    | Details                                            |
| ---------------------- | -------------------------------------------------- |
| **Target Device**      | Xilinx Zynq-7000 (`XC7Z020` / PYNQ-Z1 or ZedBoard) |
| **Synthesis Tools**    | Vitis HLS, Vivado Design Suite                     |
| **Languages**          | C++, Verilog HDL, Python (PYNQ driver)             |
| **Interconnect**       | AXI4-Lite, AXI-DMA, AXI4-Stream                    |
| **Datapath Precision** | Fixed-point arithmetic (`ap_fixed<X,Y>`)           |

---

# Hardware-in-the-Loop (HIL) Validation

Recorded battery current and voltage dynamic drive cycles are streamed from host memory to the FPGA via AXI-DMA. The coprocessor outputs SOC predictions, which are streamed back to compare against MATLAB/Simulink ground truth.

```text
MATLAB / Dataset ──► Host Memory ──► AXI-DMA ──► [ Zynq-7000 FPGA ] ──► SOC Output ──► Validation
```

### Evaluated Drive Cycles

* **UDDS:** Urban Dynamometer Driving Schedule
* **US06:** High-Acceleration Supplemental FTP

---

# Results

> [!NOTE]
> Results will be added following completion of setup and experimental runs:
>
> * MATLAB/Simulink reference model validation
> * Vitis HLS synthesis & timing closure
> * Fixed-point quantization analysis
> * Multi-cell scaling & resource utilization measurements
> * HIL test execution

### Resource Utilization (Target: XC7Z020)

| Metric                     | Utilized | Available | Utilization % |
| -------------------------- | -------: | --------: | ------------: |
| **LUT**                    |       -- |    53,200 |            -- |
| **FF**                     |       -- |   106,400 |            -- |
| **DSP48E**                 |       -- |       220 |            -- |
| **BRAM**                   |       -- |       140 |            -- |
| **Max Frequency (`Fmax`)** |   -- MHz |        -- |            -- |

---

# Roadmap

* [x] Repository setup and structural specification
* [ ] 2-RC battery model parameterization
* [ ] Reference EKF and Standard UKF implementation
* [ ] SUKF algorithm implementation
* [ ] AR(1) colored-noise state augmentation
* [ ] MATLAB simulation & validation
* [ ] Vitis HLS SUKF kernel development
* [ ] Fixed-point precision optimization (`ap_fixed`)
* [ ] Context SRAM & multi-cell interleaving logic
* [ ] Innovation-gated scheduler implementation
* [ ] Vivado block design & synthesis
* [ ] PYNQ AXI-DMA driver setup
* [ ] Real-time HIL drive-cycle testing (UDDS / US06)
* [ ] Data collection & manuscript reporting
