## FFT Architectures

The AMD/Xilinx FFT IP Core provides four architecture options, allowing designers to trade off between throughput, latency, and hardware resource utilization.

### 1. Pipelined Streaming I/O

The Pipelined Streaming architecture is designed for maximum throughput. Data is continuously streamed into and out of the FFT core without requiring the complete frame to be stored before processing.

#### Features

- Highest throughput
- Continuous data streaming
- Suitable for real-time applications
- Uses Decimation-In-Frequency (DIF)
- Higher resource utilization

#### Applications

- OFDM systems
- Wireless communication
- High-speed DSP applications

---

### 2. Radix-4 Burst I/O

The Radix-4 Burst architecture stores data internally and processes it in bursts using Radix-4 butterflies.

#### Features

- Fewer computation stages
- Better performance than Radix-2 Burst
- Moderate resource utilization
- Uses Decimation-In-Time (DIT)

#### Number of Stages


log_4(N)


---

### 3. Radix-2 Burst I/O

The Radix-2 Burst architecture processes data using Radix-2 butterflies.

#### Features

- Simpler implementation
- Lower hardware complexity
- Uses Decimation-In-Time (DIT)
- Suitable for medium-performance designs

#### Number of Stages

log_2(N)


---

### 4. Radix-2 Lite Burst I/O

The Radix-2 Lite Burst architecture is optimized for minimum FPGA resource utilization.

#### Features

- Lowest resource consumption
- Simplified hardware implementation
- Uses Decimation-In-Time (DIT)
- Lowest throughput among all architectures

#### Best For

- Resource-constrained FPGA designs
- Educational and low-cost implementations

---

## Architecture Comparison

| Architecture | Algorithm | Throughput | Resource Usage | Best Use Case |
|-------------|-----------|------------|----------------|---------------|
| Pipelined Streaming I/O | DIF | Highest | Highest | Real-time DSP |
| Radix-4 Burst I/O | DIT | High | Medium-High | Balanced performance |
| Radix-2 Burst I/O | DIT | Medium | Medium | General-purpose FFT |
| Radix-2 Lite Burst I/O | DIT | Low | Lowest | Resource-constrained designs |

---

## Architecture Selection Guide

### Choose Pipelined Streaming I/O if:
- Maximum throughput is required.
- Continuous streaming data is available.
- FPGA resources are not a major constraint.

### Choose Radix-4 Burst I/O if:
- High performance is required.
- A balanced resource-performance tradeoff is desired.

### Choose Radix-2 Burst I/O if:
- Simplicity is preferred.
- Moderate performance is sufficient.

### Choose Radix-2 Lite Burst I/O if:
- FPGA resources are limited.
- Throughput requirements are low.
