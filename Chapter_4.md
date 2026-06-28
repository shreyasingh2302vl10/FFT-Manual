# Designing with the Core

## 1. Clocking (`aclk`)
Main clock source that synchronously controls all internal states, inputs, and outputs.

### Clock Enable (`aclken`)
* **Operations:**
  * **LOW:** Pauses the core (freezes current state).
  * **HIGH:** Resumes normal core processing.
* **Design Note:** Using `aclken` can reduce the core's maximum operating frequency ($f_{max}$).
## 2. Resets

### Synchronous Clear (`aresetn`)
An active-LOW synchronous reset signal that re-initializes the entire core.

* **Priority:** Overrides `aclken`. The core resets even if `aclken` is LOW.
* **Timing:** Requires a minimum active pulse of **2 clock cycles**.
* **Operations on Reset (`aresetn = 0`):**
  * Halts all active loads, transforms, and unloads.
  * Clears all internal counters and state variables.
  * Restores default/power-on configuration values.

#### Initial / Reset Values (Table 3 Summary)

| Signal | Initial / Reset Value | Condition / Notes |
| :--- | :--- | :--- |
| **NFFT** | Maximum point size ($N$) | Set via Vivado IDE |
| **FWD_INV** | `1` (Forward FFT) | Default direction |
| **SCALE_SCH** | $1/N$ | Overall scaling schedule |
| | `[10 10... 10]` | Radix-4 Burst / Pipelined (N is a power of 4) |
| | `[01 10... 10]` | Radix-4 Burst / Pipelined (N is NOT a power of 4) |
| | `[01 01... 01]` | Radix-2 Burst / Radix-2 Lite Burst |
###  Numerical Example of SCALE_SCH

For an FFT point size $N = 32$ (Radix-4 Burst I/O, where $N$ is NOT a power of 4):
* **Total Scaling Required:** $1/32$
* **Stage Breakdown ($32 = 4 \times 4 \times 2$):**
  * Stage 1 (Radix-4): Scale by 4 $\rightarrow$ Binary `10`
  * Stage 2 (Radix-4): Scale by 4 $\rightarrow$ Binary `10`
  * Stage 3 (Radix-2): Scale by 2 $\rightarrow$ Binary `01` (Last Stage)
* **Resulting Vector:** `[01 10 10]`
