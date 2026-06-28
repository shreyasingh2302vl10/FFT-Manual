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
 
## 6. Configuration Channel

The Configuration Channel is an AXI4-Stream slave interface used to supply runtime parameters to the FFT core.

### Channel Interfaces (Table 4 Summary)

| Port Name | Width (Bits) | Direction | Description |
| :--- | :--- | :--- | :--- |
| **`s_axis_config_tdata`** | Variable | Input (I) | Configuration payload packet. Contains fields: `CP_LEN`, `FWD_INV`, `NFFT`, and `SCALE_SCH`. |
| **`s_axis_config_tvalid`** | 1 | Input (I) | Asserted by external master when valid configuration data is present on the bus. |
| **`s_axis_config_tready`** | 1 | Output (O) | Asserted by the FFT core when it is ready to accept the configuration packet. |

*Design Note: The exact bit-width of `s_axis_config_tdata` dynamically adjusts in the AMD Vivado IDE depending on which configuration fields are enabled during IP generation.*
## 7. Configuration Channel TDATA Fields Detailed Breakdown

The `s_axis_config_tdata` vector consolidates all runtime configuration parameters. Below is the detailed structural mapping of each field within the vector:

| Field Name | Bit Width | Padded to 8-bit? | Functional Description |
| :--- | :--- | :--- | :--- |
| **`NFFT`** | 5 bits | **Yes** | Specifies runtime transform size. Value = $\log_2(\text{point size})$. <br>*Example:* For a 1024-point max core, setting `NFFT` to 9 switches it to 512-point mode. |
| **`CP_LEN`** | $\log_2(\text{max point size})$ | **Yes** | **Cyclic Prefix Length:** Defines the number of samples from the end of the transform to be prepended to the output frame. Range: `0` to `point size - 1`. |
| **`FWD_INV`** | 1 bit per data channel | **No** | **Direction Control:** <br>• `1` = Forward FFT <br>• `0` = Inverse FFT (IFFT). <br>Bit 0 maps to Channel 0, Bit 1 to Channel 1, etc. |
| **`SCALE_SCH`** | • $2 \times \text{stages}$ (Burst)<br>• $2 \times (\text{stages}/2)$ (Pipelined) | **Field level padding** | **Scaling Schedule:** Array of 2-bit values determining bit-shifts per stage to prevent calculation overflow. Ordered from **last stage to first stage** ($[ \text{Last} \dots \text{First} ]$). |

---

### Key Scaling Schedule Constraints & Examples

* **Burst Architectures (Radix-4 / Radix-2):** Each stage gets 2 bits. 
  * *Example (1024-pt Radix-4):* `[1 0 2 3 2]` defines shifts from last to first stage.
* **Pipelined Streaming Architecture:** 2 bits are specified for every *pair* of Radix-2 stages.
  * *Example (256-pt):* `[2 2 2 3]`
  * *Non-power-of-4 constraint (e.g., 512-pt):* The last stage bit growth is limited. Maximum value for the two MSBs can only be `00` or `01`. (e.g., `[1 2 2 2 2]` is valid, `[2 2 2 2 2]` is **invalid**).

>  **Hardware Optimization Note:** All padded fields must be extended to the nearest 8-bit boundary. Driving unused padding bits to a constant logic level (like ground/`0`) optimizes FPGA routing and reduces resource consumption.
