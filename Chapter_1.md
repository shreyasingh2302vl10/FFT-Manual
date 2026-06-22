# Introduction – FFT IP Core

AMD/Xilinx ka **FFT (Fast Fourier Transform) IP Core** Cooley–Tukey FFT algorithm implement karta hai jo **Discrete Fourier Transform (DFT)** ko efficiently calculate karta hai.

## Main Features

- Forward FFT aur Inverse FFT dono support karta hai.
- Transform size `N = 2^m`, jahan `m = 3` se `16` tak hai.
- Fixed-point aur Floating-point dono support.
- Data width: 8–34 bits.
- Runtime par FFT size configure kiya ja sakta hai.
- Natural order ya Bit-reversed order output.
- Cyclic Prefix insertion support.
- Multiple architectures available:
  - Pipelined Streaming I/O
  - Radix-4 Burst I/O
  - Radix-2 Burst I/O
  - Radix-2 Lite Burst I/O

## Arithmetic Modes

1. Unscaled Fixed Point
2. Scaled Fixed Point
3. Block Floating Point
4. Native Floating Point

## Supported Devices

- Versal SoC
- UltraScale+
- UltraScale
- Zynq-7000
- 7-Series FPGA

## Interface

- Uses **AXI4-Stream Interface** for communication.

## One-line Viva Answer

> The FFT LogiCORE IP is a configurable AMD/Xilinx FPGA IP that efficiently computes forward and inverse DFTs using the Cooley-Tukey FFT algorithm with support for multiple architectures, data formats, and AXI4-Stream interfaces.

---

**Note:** This chapter mainly introduces the FFT IP Core and its features. The actual working, architectures, and implementation details begin in **Chapter 2 (Overview)**.
