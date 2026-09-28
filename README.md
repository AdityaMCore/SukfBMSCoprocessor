
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

Every battery pack in an electric vehicle (EV) or energy storage system (ESS) contains dozens to thousands of individual battery cells[cite: 2]. To ensure safety, prevent thermal runaway, and maximize battery lifespan, the Battery Management System (BMS) must constantly monitor the State-of-Charge (SOC)—essentially the "fuel gauge"—for **every single cell** in real time[cite: 2].

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
