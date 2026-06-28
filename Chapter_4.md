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
## 3. Event Signals

Real-time, non-AXI status signals updated cycle-by-cycle. If left unconnected, synthesis tools optimize them away.

| Signal Name | Duration | Description / Trigger Condition |
| :--- | :--- | :--- |
| **`event_frame_started`** | 1 clock cycle | Asserts when the core begins processing a new frame. Used for frame synchronization/counting. |
| **`event_tlast_missing`** | 1 clock cycle | **Error:** Upstream frame size is larger than core config (`tlast` missing at expected frame end). |
| **`event_tlast_unexpected`** | 1 clock cycle | **Error:** Upstream frame size is smaller than core config (`tlast` arrived too early). |
| **`event_fft_overflow`** | Every cycle | **Error:** Data overflow detected during scaled or pseudo floating-point operations. |
| **`event_data_in_channel_halt`** | Every cycle | **Stall:** Input channel is empty when the core expects data. <br>• *Realtime:* Processing continues (frame corrupted).<br>• *Non-Realtime:* Core stalls safely until data arrives. |
| **`event_data_out_channel_halt`** | Every cycle | **Stall:** Output channel buffer is full. Core halts safely. *(Non-Realtime mode only)*. |
| **`event_status_channel_halt`** | Every cycle | **Stall:** Status channel buffer is full. Core halts safely. *(Non-Realtime mode only)*. |
#  AXI4-Stream Considerations

Except for clocking, resets, and event signals, all data movement in and out of the FFT core uses standard **AXI4-Stream channels**.

## The Handshake Mechanism
Data transfer happens exclusively when a synchronous handshake occurs between the source and the destination:
* **`TVALID` (Source):** Asserted when valid data is present on the channel.
* **`TREADY` (Destination):** Asserted when the receiver is ready to accept data.
* **Transfer Condition:** A data word is transferred on the rising edge of `aclk` only when both `TVALID` and `TREADY` are **HIGH**.

## Channel Composition
Each stream channel bundles a handshake mechanism with a data payload:

| Port | Type | Description |
| :--- | :--- | :--- |
| **`TVALID`** | Control | Indicates valid data payload is available. |
| **`TREADY`** | Control | Indicates receiver readiness (flow control). |
| **`TDATA`** | Payload | The actual mathematical data (operands/results). |
| **`TLAST`** | Payload | Marks the boundary of the current frame (packet end). |
| **`TUSER`** | Payload | Optional sideband information accompanying the data. |

## 5. Basic Handshake & Channel Rules

### Protocol Handshake Logic
* **Master (Source):** Controls `TVALID`, `TDATA`, `TUSER`, and `TLAST`.
* **Slave (Receiver):** Controls `TREADY`.
* **Execution:** A transfer happens **only** when `TVALID` and `TREADY` are simultaneously **TRUE** on a clock edge.

---

## AXI Channel Rules

### 1. Data Alignment (Little Endian)
* All sub-fields inside `TDATA` and `TUSER` are packed in **Little Endian** format (Bit 0 of any sub-field aligns with Bit 0 of the container vector).

### 2. Dynamic Port Presence
* Signals are physically generated **only if required** by the Vivado GUI configuration. 
* *Example:* If FFT point-size is configured as fixed, the `NFFT` input field is completely excluded from the core interface.

### 3. Byte Padding (8-bit Boundary)
* The overall width of `TDATA` and `TUSER` buses must always be a **multiple of 8 bits**.
* If concatenated fields do not naturally finish on an 8-bit boundary, the core automatically pads the most significant bits (MSBs) with zeros to round up.
