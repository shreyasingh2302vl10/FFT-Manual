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
## 🔧 AXI4-Stream Configuration Channel (`s_axis_config_tdata`) Architecture

This section documents the structural mapping and bit-packing rules applied to the FFT/IFFT Core Configuration Channel (`s_axis_config_tdata`), adhering strictly to Xilinx/AMD Product Guide **PG109 (Figure 3: Configuration Channel TDATA Format)**.

### 📐 Structural Layout Hierarchy (Right to Left / LSB to MSB)
The configuration register follows a strict physical alignment boundary. When advanced features like the **Cyclic Prefix (CP)** are omitted or included, the vector dynamically adjusts (collapses) while maintaining individual sub-field padding.

$$\text{[MSB]} \longleftarrow \text{Global PAD} \longleftarrow \text{SCALE\_SCH} \longleftarrow \text{FWD\_INV} \longleftarrow \text{CP\_LEN} \longleftarrow \text{NFFT} \longleftarrow \text{[LSB]}$$

---

### 🎛️ Field-by-Field Core Rules & Division Math

#### 1. `NFFT` (Transform Size) | *Padded: Yes*
* **Physical Width:** Dedicated 5 bits in the IP core.
* **Behavior:** Holds the value $\log_2(\text{Transform Size})$. For an 8-point transform, $\log_2(8) = 3$ (`5'b00011`).
* **Alignment Padding:** Must be padded internally to hit the nearest byte boundary (3 bits of zero extension), resulting in an 8-bit block: `00000011`.

#### 2. `CP_LEN` (Cyclic Prefix Length) | *Padded: Yes (Left-Aligned)*
* **Behavior:** The hardware samples the top `NFFT` bits from the MSB side of this field. 
* **The Shift Phenomenon:** For a maximum core size of 128 ($\log_2(128) = 7\text{ bits}$), requesting a CP length of 4 (`3'b100`) requires left-aligning the bits inside the 7-bit container (`7'b1000000` = Decimal 64).
* **Alignment Padding:** Padded with 1 leading zero to complete an 8-bit block: `01000000`.

#### 3. `FWD_INV` (Direction Control) | *Padded: No*
* **Behavior:** Contains exactly **1 bit per physical FFT channel** configured in the IP GUI.
* `1` = Forward FFT (Receiver Mode)
* `0` = Inverse FFT / IFFT (Transmitter Mode)

#### 4. `SCALE_SCH` (Scaling Schedule) | *Padded: Whole Field*
* **Pipelined Streaming Specifics:** Specified using **2 bits for every pair of Radix-2 stages**. 
* **The Scaling Factor:**
  * `2'b00` (0) $\rightarrow$ No scaling ($\div 1$)
  * `2'b01` (1) $\rightarrow$ Right-shift by 1 bit ($\div 2$)
  * `2'b10` (2) $\rightarrow$ Right-shift by 2 bits ($\div 4$)
  * `2'b11` (3) $\rightarrow$ Right-shift by 3 bits ($\div 8$)
* **Non-Power-of-4 Constraint:** For transforms like $N=8$ (3 stages $\rightarrow$ 2 pairs) or $N=512$ (9 stages $\rightarrow$ 5 pairs), the last stage is left un-paired. Since a standalone stage can only handle a max growth of 1 bit, the two MSBs of `SCALE_SCH` are strictly constrained to `00` or `01`.

---

### 💻 Validated Reference Configuration Words

The following hex values represent verified control patterns for a **1-Channel, 8-Point Transform** with **No Cyclic Prefix**:

## 📤 Data Output Channel (`m_axis_data_tdata`) Architecture

The Data Output channel is an AXI4-Stream master interface that carries the real and imaginary results of the FFT transform on the `TDATA` vector, along with real-time per-sample status metadata on the `TUSER` vector.

---

### 1. Structural Bit-Packing Layout (TDATA)
Following the Xilinx standard complex data packing convention (**Figure 8 / Table 11**), the complex frequency-domain output samples ($XK$) are tightly packed into a single container where the **Imaginary component resides at the MSB side** and the **Real component resides at the LSB side**.

$$\text{m\_axis\_data\_tdata[31:0]} = \underbrace{\text{XK\_IM [31:16]}}_{\text{Signed Imaginary Output (Q)}} \;\mid\; \underbrace{\text{XK\_RE [15:0]}}_{\text{Signed Real Output (I)}}$$

* **No Sub-field Padding:** Since our system is configured for a **Single-Channel** environment with native **16-bit word widths**, each internal component natively aligns to a clean 8-bit byte boundary (16 bits = 2 bytes). Therefore, no extra padding or trailing bits are injected within the 32-bit `TDATA` stream.

---

### 2. Xilinx Sign-Extension Rule (Twos Complement)
For configurations where the output bit-width does not cleanly finish on an 8-bit boundary (e.g., 12-bit or 14-bit core settings), the IP core enforces strict **Sign-Extension** instead of zero-padding to maintain mathematical polarity:
* **Positive Samples (MSB = `0`):** Padded with leading zeros (`0000`).
* **Negative Samples (MSB = `1`):** Padded with leading ones (`1111`) to preserve 2's complement value integrity.

*Note: Since our system uses native 16-bit boundaries, the raw values are directly accessible without manual mask-shifting.*

---

### 3. Per-Sample Status Tracking (`m_axis_data_tuser`)
To guarantee that tracking metadata never gets out of synchronization with the high-speed data stream, the `TUSER` vector runs completely parallel to `TDATA`:

1. **`XK_INDEX` (Bits [2:0]):** A real-time hardware bin index counter. For our 8-point FFT, this loops continuously from `0` to `7` on consecutive clock cycles, explicitly identifying the current frequency bin ($f_0$ to $f_7$).
2. **`OVFLO` (Bit [3]):** A 1-bit sticky overflow status flag. If the internal butterfly execution stages experience numerical clipping due to an aggressive or inadequate scaling schedule (`SCALE_SCH`), this bit asserts high (`1'b1`).

---

### 4. AXI4-Stream Output Handshake Control

* **`m_axis_data_tvalid` (Output):** Asserted high by the FFT core when the frame processing is complete and valid frequency-domain samples are ready to be read.
* **`m_axis_data_tready` (Input):** Controlled by our custom RTL state machine. Must be asserted high (`1'b1`) to allow the core to unload its pipeline registers. Dropping `tready` to `0` instantly freezes the core's output stage.
* **`m_axis_data_tlast` (Output):** Asserted high by the core strictly during the transmission of the last sample of the frame (when `XK_INDEX = 7`), signaling downstream logic to close the frame packet.

---

### 💻 Verilog Real-Time Data Extraction

```verilog
// ⚠️ CRITICAL: Always declare extraction wires as SIGNED 
// to ensure negative twos-complement numbers compile correctly.
wire signed [15:0] fft_out_real;
wire signed [15:0] fft_out_imag;
wire [2:0]         fft_bin_index;
wire               fft_overflow_flag;

// Slicing TDATA into distinct Real (I) and Imaginary (Q) components
assign fft_out_real      = m_axis_data_tdata[15:0];   // Lower 16-bits (LSB)
assign fft_out_imag      = m_axis_data_tdata[31:16];  // Upper 16-bits (MSB)

// Unpacking TUSER metadata for synchronization tracking
assign fft_bin_index     = m_axis_data_tuser[2:0];    // Maps current frequency bin (0-7)
assign fft_overflow_flag = m_axis_data_tuser[3];      // Hardware overflow warning indicator
