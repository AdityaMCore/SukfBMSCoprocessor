
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
