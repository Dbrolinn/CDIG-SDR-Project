The repository is organized into three main sections:

* **Code:** Contains the GNU Radio flowgraphs.
* **Doc:** Houses the project report, which provides a deep dive into how the protocol works.
* **Code (Root):** Includes the `test.pcap` file, where captured packets are saved once received through GNU Radio.

This README serves as the project wiki, mirroring the documentation delivered on Moodle by Miguel on moodle.


# Week 1 - 20/10/25

This week marked the beginning of the project, focusing on the theoretical foundations of the IEEE 802.11a physical layer and the initial frame detection stage of the GNU Radio receiver. We studied the **OFDM system design parameters** required to configure the receiver and analyzed the receiver's software architecture, specifically the frame detection logic presented in the paper.

---

## 1. 802.11a OFDM System Parameters

To implement the receiver, we first defined the fundamental parameters of the 802.11a PHY. These values dictate how the GNU Radio flowgraph samples and processes the incoming signal.

### Frequency Domain Parameters
* **Channel Bandwidth:** 20 MHz.
* **FFT Size ($N_{FFT}$):** 64 points.
* **Sub-carrier Spacing ($\Delta f$):** 312.5 kHz.
    * *Calculation:* $\frac{Bandwidth}{N_{FFT}} = \frac{20 \text{ MHz}}{64} = 312.5 \text{ kHz}$.
* **Number of Active Sub-carriers:** 52.
    * **Data Sub-carriers ($N_{SD}$):** 48 sub-carriers used for the payload (BPSK, QPSK, 16-QAM, or 64-QAM).
    * **Pilot Sub-carriers ($N_{SP}$):** 4 sub-carriers used for phase tracking, located at indices $\pm 7$ and $\pm 21$.

****

### Time Domain Parameters
The SDR operates in the digital domain using discrete samples. We derived the sample-based timings assuming a **20 MHz sampling rate**.
* **Sampling Rate:** 20 MHz (1 sample every 50 ns).
* **Inverse FFT Period ($T_{FFT}$):** $3.2 \mu s$ (64 samples).
* **Guard Interval Duration ($T_{GI}$):** $0.8 \mu s$ (16 samples).
* **Total OFDM Symbol Duration:** $4.0 \mu s$ ($3.2 + 0.8$).
* **Total Symbol Length (Samples):** 80 samples ($64 \text{ FFT} + 16 \text{ CP}$).


## 2. Unused Sub-carriers and Spectral Mask

We investigated why the 64-point FFT is not fully populated with data and where the "empty" carriers are located.

****

* **Why are there unused sub-carriers?**
    1.  **DC Carrier (Index 0):** This is set to zero to prevent **carrier leakage** (DC offset) from the Digital-to-Analog converter. If active, the local oscillator's own energy would corrupt the data at the center frequency.
    2.  **Guard Bands:** These are "empty" carriers at the edges of the spectrum. They ensure the signal energy decays sufficiently before entering adjacent channels, satisfying the spectral mask requirements to prevent interference.
* **Where are they located?**
    * **DC Null:** Index 0.
    * **Guard Sub-carriers:** Indices -32 to -27 (Lower edge) and +27 to +31 (Upper edge). Total of 11 guard sub-carriers + 1 DC = 12 Nulls.


## 3. Cyclic Prefix (CP) Strategy

We analyzed the Cyclic Prefix, a critical component of OFDM that solves the multipath propagation problem.

****

* **Length:** 16 samples ($0.8 \mu s$).
* **Why is it used?**
    1.  **Guard Interval:** It acts as a buffer. As long as the multipath delay spread is shorter than $0.8 \mu s$, the echoes from the previous symbol will settle within the CP window and will not interfere with the current symbol (eliminating Inter-Symbol Interference).
    2.  **Circular Convolution:** By making the signal appear periodic to the FFT, it converts the channel's linear convolution into a circular convolution. This allows simple, single-tap frequency domain equalization.


## 4. Frame Prefix Structure (Preamble)

Each 802.11a frame begins with a physical layer preamble, which is $16 \mu s$ long in total.

****

* **Short Training Sequence (STS):**
    * **Composition:** 10 identical short OFDM symbols.
    * **Length:** 16 samples each $\times$ 10 repetitions = **160 samples** total ($8 \mu s$).
    * **Purpose:** Used for signal detection, Automatic Gain Control (AGC), and coarse frequency offset estimation.
* **Long Training Sequence (LTS):**
    * **Composition:** A Guard Interval (32 samples, $1.6 \mu s$) followed by 2 identical long OFDM symbols (64 samples, $3.2 \mu s$ each).
    * **Length:** $32 + 64 + 64$ = **160 samples** total ($8 \mu s$).
    * **Purpose:** Used for channel estimation and fine frequency offset tuning.

---

## 5. Receiver Architecture (Section 2.1)

We analyzed the software architecture proposed in the paper. The receiver is divided into two main functional parts: **frame detection** and **frame decoding**.
* **Flowgraph Coordination:**
    * **Stream Tagging:** Used to attach metadata to specific samples (e.g., marking the start of a frame or passing the frame's length/encoding scheme).
    * **Message Passing:** Utilized for packet-based data handling, allowing the encapsulation of arbitrary information to switch between stream-based processing and packet processing.

---

## 6. Frame Detection Implementation (Section 2.2)

The first stage of the receiver is Frame Detection, which leverages the periodic nature of the short preamble (16-sample pattern repeating 10 times). We implemented an autocorrelation function with a 16-sample lag using two parallel branches:

* **Autocorrelation Branch (Top):** Calculates the numerator. The incoming signal is split; one path is delayed by 16 samples, and the other is complex conjugated. They are multiplied together, and the result is summed using a **Decimating FIR Filter** (with Decimation=1), effectively acting as a moving window summer.
* **Power Branch (Bottom):** Calculates the average power of the signal. This is used to normalize the autocorrelation coefficient, ensuring detection is based on signal similarity rather than raw amplitude.

### Experiment: Window Size ($N_{Win}$)
We explored the effect of varying the `window_size` parameter, which controls the length of the moving average filter.
* **Small ($N=16$):** Resulted in noisy spikes and an unstable detection plateau.
* **Large ($N=96$):** Produced a smooth but dispersed plateau, making the precise start of the frame indistinct.
* **Optimal ($N=48$):** Provided the best balance, generating a clean and stable plateau for reliable detection.
---

# Week 2 - 27/10/25

This week's work was focused on understanding how the blocks sync short and sync long worked. There was an obstacle that we faced which we will try to solve during the next week of work. The problem was that we were incapable of successfully making the download of the module for the 802.11 blocks, making it impossible to implement the frame decoding part of the flowgraph in gnu radio, we investigated an alternative, where we can implement ourself the code in a python and add it to the library of our project. Then, due to the impossibility of implementing and testing the practical part, which was suggested in the guide on moodle, we decided to focus on understanding how both blocks, OFDM Sync Short and OFDM Sync Long are constructed, and what their purposes in the decoding is.

---

## Analysis of the OFDM Sync Short Block

### 1. Block Function and Objective
The OFDM Sync Short block, as described in the paper, functions as the primary "gatekeeper" for the entire receive pipeline. Its objective is to perform coarse frame detection by continuously monitoring the incoming sample stream for a specific signal signature.
This signature is the short preamble sequence, which marks the beginning of every IEEE 802.11a/g/p frame. When the block determines with high confidence that a preamble has arrived, it acts as a "valve," opening to pass a fixed-size chunk of samples to the next part of the pipeline: the OFDM Sync Long block, which handles fine synchronization and decoding.

### 2. Implementation and Parameters
The block's detection algorithm is based on the structure of the short preamble, which consists of a 16-sample pattern repeated 10 times (totaling 160 samples).
The core of the implementation is a normalized autocorrelation with a 16-sample lag. This process continuously compares the incoming signal stream with a 16-sample-delayed version of itself.

* During random noise, the correlation is low.
* When the 10 repeating 16-sample patterns of the preamble enter the block, the signal becomes highly correlated with its 16-sample-delayed version, causing the normalized autocorrelation value to jump to a high value (near 1.0).

This high correlation value is not used in its raw form. As noted in the paper (Section 2.2), it is first smoothed by a 48-sample moving average window. This is a crucial design choice:

$$16 \text{ samples} \times 3 \text{ patterns} = 48 \text{ samples}$$

By averaging over three full preamble patterns, the block "low-pass filters" the autocorrelation output. This filtering produces the stable, clean "plateau" seen in Figure 2, making the detection highly robust against momentary noise spikes.

This smoothed signal is then checked against two configurable parameters:

**Threshold (e.g., 0.8)**
* **Meaning:** The threshold defines the minimum amplitude the smoothed autocorrelation signal must reach to be considered part of a valid preamble. It is the sensitivity setting for the detector.
* **Effect of Change:** This parameter controls the trade-off between sensitivity and false positives.
    * **A Low Threshold (e.g., 0.5):** Increases sensitivity, allowing the receiver to detect weaker or more distant frames. However, it also significantly increases susceptibility to false positives, where random noise might be mistaken for a frame, wasting valuable processing time.
    * **A High Threshold (e.g., 0.9):** Decreases sensitivity, making the receiver highly robust against noise. It will only trigger on very clean, strong signals, but it will likely miss weaker (yet still valid) frames.
* The 0.8 value used in the paper is a balanced compromise, offering good noise immunity while still being sensitive enough for typical operation.

**Min Plateau (e.g., 2 or 3)**
* **Meaning:** This value defines the minimum number of consecutive smoothed samples that must all be above the threshold to confirm a detection.
* **Effect of Change:** This parameter provides a second layer of robustness. A single sample spiking above the threshold could be a computational anomaly or a noise burst that (even after smoothing) looks like a signal. By requiring 2 or 3 consecutive samples to be high, the block confirms it has found a stable plateau, not a random spike, before "opening the valve."

### 3. Limitations of the Paper's Approach
This implementation, while effective, has significant limitations directly caused by the "fixed-size window" approach. This method is a workaround for the processing latency of a non-real-time SDR system; a dedicated COTS (Commercial Off-The-Shelf) receiver would decode the Signal Field in real-time to determine the exact frame length.
The reliance on a fixed-size capture window (e.g., 5.2k samples) creates two critical failure cases.

* **Break-Case 1: Frame is Larger than the Window**
    This is the most straightforward limitation. If the receiver captures a frame with a large data payload (e.g., a 1500-byte packet at a low data rate), the total number of symbols may exceed the fixed window size. The Sync Short block will "close the valve" prematurely, cutting off the end of the frame. This results in an unrecoverable packet loss as the payload is incomplete. While increasing the window size to the protocol's maximum can mitigate this, it worsens the second problem.

* **Break-Case 2: Consecutive Back-to-Back Frames**
    This is a more common and critical failure, which the authors explicitly acknowledge. The 802.11 protocol relies on high-speed frame exchanges, such as an RTS (Request to Send) frame followed immediately by a CTS (Clear to Send) frame after a very short (SIFS) gap.
    In this scenario:
    1.  The Sync Short block detects the preamble of the RTS frame.
    2.  It "opens the valve" and captures its large, fixed-size chunk of samples.
    3.  This chunk contains: `[ RTS Frame | SIFS Gap | CTS Preamble | Start of CTS Frame | ...noise... ]`
    4.  The receiver is now "busy" processing this single large chunk. It completely misses the preamble of the CTS frame because it was "blind" during that time.
    5.  Even worse, the CTS frame itself has been "decapitated"—its preamble and beginning are now incorrectly attached to the end of the RTS frame's capture chunk, ensuring that both frames are lost.

### 4. The Solution (Updated Repository Version)
This "blindness" to back-to-back frames was a major flaw in the paper's original design. As confirmed by the author, the gr-ieee802-11 repository fixes this with a more intelligent implementation.
Instead of acting as a "valve" that copies large, blind chunks, the new Sync Short block functions as a continuous "tagger":

* It continuously monitors the incoming sample stream (Input 1) and the smoothed autocorrelation stream (Input 2).
* When it detects a valid plateau (e.g., 3 samples > 0.8), it does not copy any data.
* Instead, it adds a stream tag (e.g., `frame_start`) to the exact sample in the original stream that corresponds to the start of the preamble.
* Crucially, it then immediately resumes scanning for the next plateau.
* This new method solves the "back-to-back" problem perfectly. When an RTS/CTS exchange occurs, the block simply outputs the continuous stream of samples with two separate tags: one marking the start of the RTS frame and another, just a few hundred samples later, marking the start of the CTS frame. The downstream blocks can then use these tags to process each frame individually.

---

# Week 3 - 06/10/25

For Week 3, we focused on the internal architecture of the **OFDM Sync Long** block, corresponding to **Section 2.3** of the receiver guide. Unlike the "Sync Short" block, which runs continuously to detect frames, the "Sync Long" block is a triggered state machine . We analyzed its input structure (the Delay Buffer) and the first of its two signal processing tasks: **Fine Frequency Offset Correction**.

---

## Analysis of the OFDM Sync Long Block (Section 2.3)

We dissected the block's operation into two main components: handling the processing latency via a split-path architecture and performing fine-grained frequency correction using the Long Training Sequence (LTS).

### 1. The Split-Path Architecture and Delay Buffer
We investigated the question: *"Why is there a delayed input?"* . The receiver employs a split-path design to handle the time gap between "detecting" a frame and "processing" it.

****

* **Input 0 (Control/Tag Path):** Receives the direct stream from the Sync Short block. This stream carries the `wifi_start` stream tag, which acts as the trigger to wake up the Sync Long state machine .
* **Input 1 (Data/Buffered Path):** Receives the signal passed through a **Delay Block** (set to 240 samples) .
* **Why is the delay necessary?**
    * The detection logic in the previous block (Sync Short) requires a finite window of time to calculate the autocorrelation plateau and confirm a frame is present .
    * By the instant the "Frame Detected" decision is made, the actual start of the frame (the preamble) has already flowed past the block .
    * The 240-sample delay acts as a **pre-roll buffer**, holding the "past" samples in memory. When the Sync Long block is triggered, it reads from this delayed input, effectively "reaching back in time" to capture the frame from the very first sample of the Short Training Sequence, ensuring zero data loss .

* **Impact of Changing the Delay Value ($N=240$):**
    * **Making it Shorter (e.g., < 160):** If the delay is reduced too much, it may become shorter than the processing latency of the detection block. The `Sync Long` block would "wake up" too late, reading a stream where the start of the preamble has already passed. This would cause the receiver to miss the Long Training Sequence entirely, resulting in synchronization failure and data loss.
    * **Making it Larger (e.g., > 300):** Increasing the delay adds unnecessary latency to the entire receive pipeline. While it provides a larger safety margin for the software scheduler, it delays the arrival of the decoded packet at the MAC layer.

### 2. Fine Frequency Offset Correction
Once the frame data is retrieved from the buffer, the block performs Fine Frequency Correction. We distinguished this from the "Coarse" correction done in Week 2 .

* **The Signal (LTS):** This step uses the Long Training Sequence, which consists of a Guard Interval followed by two identical OFDM symbols ($T_1$ and $T_2$), each **64 samples** long .
* **The Mathematical Principle:**
    * A Carrier Frequency Offset (CFO) causes the constellation to spin, manifesting as a phase rotation over time .
    * Since $T_1$ and $T_2$ are identical, any phase difference between them is solely due to the frequency offset accumulated over the duration of one symbol (64 samples) .
* **The Algorithm:**
    The block estimates the fine offset ($\Delta f_{fine}$) by calculating the phase angle between the two symbols:
    $$\Delta f_{fine} = \frac{1}{64} \arg\left( \sum_{k=0}^{63} r[n+k] \cdot r^*[n+k+64] \right)$$
    * $r[n+k]$ is the sample from the first symbol ($T_1$).
    * $r^*[n+k+64]$ is the conjugate of the sample from the second symbol ($T_2$).
    * The `arg` function extracts the total phase rotation.
    * Dividing by 64 normalizes this to the phase shift *per sample* .

* **Code Implementation:**
    We identified the specific C++ implementation in `sync_long.cc` that implements this logic:
    ```cpp
    // sync_long.cc: search_frame_start()
    // Calculate the angle of the product of the two 64-sample symbols
    d_freq_offset = arg(first * conj(second)) / 64;
    ```
    

* **Application:**
    This value is added to the coarse frequency offset. During the "COPY" state, the block applies a counter-rotation to every single sample in the payload to nullify the drift:
    ```cpp
    // sync_long.cc: general_work()
    out[o] = in_delayed[i] * exp(gr_complex(0, d_offset - d_freq_offset));
    ```
    

---

# Week 4 - 13/10/25

This week, we completed the synchronization subsystem by analyzing **Symbol Alignment (Section 2.4)**. The goal was to understand how the receiver identifies the precise start of the payload (to avoid ISI) and how it formats the data for the FFT.

---

## Symbol Alignment and FFT Preparation (Section 2.4)

After frequency correction, the signal is clean, but the receiver still does not know exactly where the symbols start. Being off by even a few samples would result in the FFT window capturing the Cyclic Prefix of the next symbol, causing Inter-Symbol Interference (ISI) and phase rotations .

### 1. Symbol Timing Alignment (Matched Filtering)
To solve the alignment problem, the Sync Long block uses a Matched Filter .

****

* **The Algorithm:**
    The block convolves the received signal with a known time-domain template of the Long Training Sequence (LTS).
    ```cpp
    // sync_long.cc
    d_fir.filter(d_correlation, in, ...);
    ```
    
* **Matched Filtering vs. Autocorrelation:**
    * *Why matched filtering used for symbol alignment but not for frame detection?* 
    * Matched filtering is computationally expensive ($\approx 64$ multiplications per sample) .
    * However, since the Sync Long block only runs this check on the preamble (a small subset of the stream), we can afford the "cost" to gain the **high precision** required for symbol alignment, whereas the Sync Short block needs a cheaper algorithm (autocorrelation) to scan the continuous stream .

### 2. Peak Detection and the +64 Offset
The matched filter produces peaks when the template aligns with the received LTS. We analyzed the specific mathematical logic used to define the "Frame Start" index.

* **Peak Selection:** The algorithm identifies the peaks corresponding to the repeated LTS symbols. It selects the latest valid peak ($\max(N_p)$) to align with the second LTS symbol, as it provides the most stable reference .
* **The Calculation:**
    $$ n_{start} = \max(N_p) + 64 $$
    
* **Why do you add 64?** 
    1.  The matched filter peak occurs at the **end** of the matched sequence pattern.
    2.  Therefore, $\max(N_p)$ points to the sample where the **second LTS symbol finishes** .
    3.  However, the standard defines the LTS as two 64-sample symbols. The data payload (Signal Field) begins immediately after the LTS. By adding 64 samples (the duration of the second LTS symbol) to the peak index, the algorithm calculates the exact sample index where the preamble ends and the Signal Field begins .

### 3. Data Preparation for the FFT
Once aligned, the block transitions to the "COPY" state and acts as a gatekeeper for the FFT.


* **Preamble Stripping:** The block discards the first 320 samples (160 Short + 160 Long Preamble) as they have served their purpose .
* **Cyclic Prefix Removal:** For the rest of the frame, the incoming stream consists of 80-sample chunks (64 Data + 16 CP). The block strips the first 16 samples (Guard Interval) from every symbol to eliminate ISI .
* **Stream to Vector:**
    * *What does the stream to vector block do?* 
    * The output is a continuous stream of 64-sample "core" symbols. However, the FFT block requires inputs as discrete vectors. The **Stream to Vector** block collects these 64 serial samples and bundles them into a single `std::vector` of length 64, satisfying the input requirement of the FFT block .

---

# Week 5

This week, we examined the Phase Offset Correction mechanism detailed in Section 2.5 and implemented within the frame_equalizer_impl.cc file. We analyzed how hardware imperfections—specifically Sampling Clock Offset (SFO) and Carrier Frequency Offset (CFO)—manifest as phase slope and common phase error, respectively. The study dissected the Frame Equalizer's three-stage algorithm, which models phase error as a linear equation. This process involves pre-correcting the "slope" based on accumulated drift, determining the "intercept" using pilot subcarriers to remove Common Phase Error (CPE), and employing a feedback loop to refine the residual timing drift estimate for subsequent symbols.

---

## Analysis of Section 2.5 and frame_equalizer_impl.cc

The frame equalizer, and therefore this file, provides a comprehensive explanation of the Phase Offset Correction mechanism used in the GNU Radio IEEE 802.11 receiver.

### 1. Why do you need to correct the phase?

According to Section 2.5 of the paper, phase correction is required to counteract two specific hardware imperfections that occur when the sender and receiver clocks are not perfectly synchronized.

**Sampling Clock Offset (SFO)**

The receiver's clock ticks slightly faster or slower than the sender's.
* **Effect:** This causes a timing delay that accumulates over the duration of the packet, which can lead to the receiver not being able to decode the message.
* **Visual:** In the frequency domain, a time delay manifests as a **Phase Slope**. Low frequencies rotate a little; high frequencies rotate a lot. The constellation twists into a spiral.

**Carrier Frequency Offset (CFO)**

The receiver's center frequency is slightly off.
* **Effect:** This causes a constant phase rotation for every subcarrier equally.
* **Visual:** The entire constellation rotates in a circle (Common Phase Error).

If these are not corrected, the constellation points will rotate out of their target zones, causing bit errors.

### 2. Find the code that estimates the phase offset

The logic is located in the Frame Equalizer block.

**File:** `lib/frame_equalizer_impl.cc`

**Function:** `frame_equalizer_impl::general_work`

Specifically, the estimation and correction happen inside the main loop iterating over symbols:
```cpp
// in frame_equalizer_impl.cc, general_work function

while ((i < ninput_items[0]) && (o < noutput_items)) {

    // 1. Correction of Sampling Offset (Slope)
    current_symbol[i] *= exp(gr_complex(0, 2 * M_PI * d_current_symbol * 80 * (d_epsilon0 + d_er) * (i - 32) / 64));

    // 2. Estimation of Pilot Phase (Beta)
    beta = arg((current_symbol[11] * p) + (current_symbol[39] * p) + ...);

    // 3. Correction of Common Phase Error (Intercept)
    current_symbol[i] *= exp(gr_complex(0, -beta));

    // 4. Update of Sampling Error (Feedback Loop)
    d_er = (1 - alpha) * d_er + alpha * er;

}
```

### 3. How are you correcting the phase?

The correction is applied in a two-step process (Linear Regression model) followed by a feedback update. The code treats the phase error $\Phi_k$ for subcarrier $k$ as a line equation:

$$\Phi_k = \text{Slope} \times k + \text{Intercept}$$

Where the **Slope** is the iteration drifting error, and the **Intercept** is beta which is calculated every symbol.

#### Step A: Correcting the Slope (Sampling Offset)

Before measuring the pilots, the code corrects the symbol based on the estimated clock drift, using:
```cpp
for (int i = 0; i < 64; i++) {
    current_symbol[i] *= exp(gr_complex(0, 2 * M_PI * d_current_symbol * 80 * (d_epsilon0 + d_er) * (i - 32) / 64));
}
```

This applies a phase rotation $e^{j\theta}$ to every subcarrier $i$.

* **`(2 * M_PI)`:** Converts the result into Radians.
* **`d_current_symbol * 80` (Time):** This calculates the Physical Time elapsed. An OFDM symbol is 64 samples (Data) + 16 samples (Cyclic Prefix). Even though the Cyclic Prefix is removed before this block, the physical clock drifted during those 16 samples, so using 64 here would under-correct the error.
* **`d_epsilon0 + d_er` (Drift Rate $\epsilon$):**
  * `d_epsilon0`: The coarse initial estimate calculated from the Preamble.
  * `d_er`: The fine residual estimate tracked by the loop (starts at 0).
* **`(i - 32) / 64` (Frequency $f$):**
  * `i - 32`: Shifts the index so 0 is the center (DC). Range becomes -32 to +31.
  * `/ 64`: Normalizes the frequency to the bandwidth.

**Equation Implemented:**

$$S_k' = S_k \cdot e^{j 2\pi \cdot N_{\text{sym}} \cdot 80 \cdot (\epsilon_0 + \epsilon_r) \cdot (k/64)}$$

#### Step B: Estimation and Correction of Intercept (CPE)

Now that the "slope" is flattened, the code calculates the "intercept" (the average rotation) using the 4 Pilot Subcarriers.
```cpp
// 1. Calculate Beta (Intercept)
double beta = arg((current_symbol[11] * p) + (current_symbol[39] * p) +
                  (current_symbol[25] * p) + (current_symbol[53] * -p));

// 2. Apply Correction
for (int i = 0; i < 64; i++) {
    current_symbol[i] *= exp(gr_complex(0, -beta));
}
```

* **Descrambling (`* p`):** The pilots are transmitted with a pseudo-random polarity $p \in \{1, -1\}$. Multiplying by $p$ removes this randomness.
  * $\text{Recv}_p = (\text{Pilot} \times p) \times p = \text{Pilot} \times p^2 = \text{Pilot} \times 1$
* **Inverting Pilot 53 (`* -p`):** The IEEE 802.11 standard defines the pilot values as $\{1, 1, 1, -1\}$. The last pilot (Index 53) is natively negative. To average them constructively, we must flip it back by multiplying by $-p$.
* **`arg(...)`:** This sums the 4 descrambled pilot vectors and calculates the angle of the resulting sum vector. This angle $\beta$ represents the Common Phase Error (CPE).
* **Correction:** The code rotates the entire symbol by $-\beta$ to align the constellation with the real axis.

#### Step C: The Feedback Loop (Refining the Slope)

Finally, the code checks if the "Slope" correction applied in Step A was perfect. Even after correcting the Phase Slope and the Intercept, a tiny residual Sampling Frequency Offset (SFO) might remain. The receiver tracks this by comparing the current pilots against the previous symbol's pilots.

**The Code:**
```cpp
// 1. Measure Instantaneous Phase Drift (radians)
double er = arg((conj(d_prev_pilots[0]) * current_symbol[11] * p) + ... );

// 2. Convert Phase Drift to Clock Error Ratio (dimensionless)
er *= d_bw / (2 * M_PI * d_freq * 80);

// 3. Update the Slope Tracker (IIR Filter)
d_er = (1 - alpha) * d_er + alpha * er;

// 4. Save current pilots for the next iteration
d_prev_pilots[...] = current_symbol[...] * p;
```

**1. Measuring Phase Drift (`arg(...)`)**

The code calculates the phase difference between the pilots of the current symbol and the pilots of the previous symbol.
* **Method:** Multiplying a complex number by the Complex Conjugate (`conj`) of another calculates the angle difference between them.
* **Logic:** If the "Slope" correction in Step A was perfect, the pilots would be perfectly stationary from one symbol to the next (after removing the SFO rotation). If the pilots have rotated relative to the previous symbol, it means the clock drift estimate (`d_er`) is slightly off and needs adjustment.

**2. Scaling Factor (The Physical Derivation)**

The value calculated above (`er`) is an angle in Radians. However, the variable we need to update (`d_er`) represents a Clock Error Ratio. To convert the measured angle into a clock error, we invert the physics of the system.

The Physics Chain:
Hardware Crystal Error ($\epsilon$) → Frequency Offset ($\Delta f$) → Phase Rotation ($\Delta \Phi$).

The relationship is defined by:

$$\Delta \Phi = 2\pi \times \Delta f \times T_{\text{symbol}}$$

Substituting Variables:
* **Frequency Offset:** $\Delta f = f_{\text{carrier}} \times \epsilon$
* **Symbol Duration:** $T_{\text{symbol}} = \text{Samples}/\text{Bandwidth} = 80/BW$

**The Combined Equation:**

$$\Delta \Phi = 2\pi \times (f_{\text{carrier}} \times \epsilon) \times \frac{80}{BW}$$

**Solving for Error ($\epsilon$):**

We isolate $\epsilon$ to find the scaling factor used in the code:

$$\epsilon = \Delta \Phi \times \frac{BW}{2\pi \times f_{\text{carrier}} \times 80}$$

**Code Implementation:**

This matches the code line exactly:
```cpp
er *= d_bw / (2 * M_PI * d_freq * 80);
```

**3. The Update Loop (Low Pass Filter)**

The calculated `er` is the instantaneous error for just this one symbol. Because wireless channels are noisy, we do not simply overwrite `d_er`. Instead, we use an Infinite Impulse Response (IIR) filter (also known as an Exponential Moving Average).
* **Formula:** `d_er = (1 - alpha) * d_er + alpha * er;`
* **Behavior:** `alpha` is a small gain factor (e.g., 0.001). This allows the loop to slowly converge on the true clock drift while smoothing out random noise spikes.

### Summary of Values Changed

* **`current_symbol` (Data):** Modified twice. First to remove the frequency-dependent twist (SFO slope) in Step A, and second to remove the constant rotation (CPE intercept) in Step B.
* **`d_er` (State):** Updated every symbol to track the changing time drift. This value is carried over to Step A of the next symbol.
* **`d_prev_pilots` (State):** The current pilots are saved to serve as the reference for the next iteration.

---
# Week 6

This week, we advanced to the final stages of the physical layer processing before payload decoding. We analyzed **Channel Estimation (Section 2.6)**, specifically focusing on magnitude correction and its critical role in higher-order modulation schemes like QAM. We identified the specific limitations of the current implementation regarding magnitude assumptions. Furthermore, we examined **Signal Field Decoding (Section 2.7)**, understanding how the receiver extracts metadata (Length and MCS) from the first symbol following the preamble and uses Stream Tagging to pass this configuration to the MAC decoder. Finally, we made practical modifications to the GNU Radio flowgraph to address errors and optimize performance.

---

## Analysis of Channel Estimation (Section 2.6)

While the previous week focused on correcting the *phase* (rotation) of the subcarriers, this week's analysis focused on correcting the *magnitude* (amplitude). This logic is typically housed within the same `OFDM Equalize Symbols` block.

### 1. Why do we need Magnitude Correction?

In Phase Shift Keying (PSK) modulations like BPSK or QPSK, information is carried entirely in the phase of the signal; the amplitude is constant. However, for higher data rates, IEEE 802.11 uses **Quadrature Amplitude Modulation (QAM)**, specifically 16-QAM and 64-QAM.

* **The Physics:** In QAM, the distance of a constellation point from the origin (its magnitude) carries bits of information.
* **The Problem:** Hardware filters (imperfect channel filters) or multipath fading can attenuate certain subcarriers more than others. If the receiver does not "equalize" these magnitudes (i.e., amplify the weak ones and attenuate the strong ones), the constellation points will fall into the wrong decision boundaries.
* **Paper Reference:** The authors state that magnitude correction is "especially important if QAM-16 or QAM-64 encoding is utilized, where also the magnitude carries information" [cite: 2491246.2491248.pdf].



### 2. Implementation and Limitations

We analyzed the specific approach taken by the authors for magnitude correction and found it to be a simplified model rather than a full Least Squares (LS) channel estimation.

* **Assumption:** The implementation assumes the magnitude distortion follows a specific "sinc-shaped" curve, likely caused by the hardware filters of the USRP/daughterboard [cite: 2491246.2491248.pdf].
* **Correction:** Instead of estimating the channel magnitude per subcarrier using the preamble, the block corrects based on this static sinc-shape assumption.
* **The Limitation:** The authors explicitly note that this static assumption is not robust. They observed that the shape "depend[s] also on the sender" (e.g., different WiFi cards have different filter shapes) [cite: 2491246.2491248.pdf].
* **Impact:** Because this magnitude equalization is not adaptive, it is currently insufficient for distinguishing the amplitude levels of complex constellations. Consequently, this limitation **"restricts our receiver to BPSK and QPSK modulations"** [cite: 2491246.2491248.pdf].

### 3. Subcarrier Subsetting

This block performs one final data-formatting task. The FFT outputs 64 subcarriers, but not all contain data. The Equalizer block removes the non-data subcarriers:
* **Inputs:** 64 Complex Samples (Data + Pilots + Guard + DC).
* **Action:** It strips out the 4 Pilot subcarriers (used for phase tracking), the 11 Guard subcarriers (edges), and the DC subcarrier (center).
* **Outputs:** 48 Complex Samples (Data only) [cite: 2491246.2491248.pdf].

---

## Analysis of Signal Field Decoding (Section 2.7)

After the preamble (Short + Long) is processed, the very first data symbol in the frame structure is the **SIGNAL Field**. We analyzed the `OFDM Decode Signal` block to understand how the receiver configures itself for the payload.

### 1. Block Function and Objective

The objective of this block is to decode the "header" of the physical layer to answer two questions: *How long is this packet?* and *How is it encoded?*

* **Robustness:** The Signal Field is *always* encoded using **BPSK modulation** with a coding rate of **1/2** [cite: 2491246.2491248.pdf]. This is the most robust modulation available in 802.11, ensuring that even if the channel is too noisy for high-speed data (like 54 Mbps), the receiver can still decode the header.
* **Library:** The paper notes that the project utilizes the **IT++ library** to handle the convolutional decoding required for this field [cite: 2491246.2491248.pdf].

### 2. The Tagging Mechanism

This block is the bridge between the physical layer and the MAC layer. It employs the **Stream Tagging** feature of GNU Radio to pass metadata downstream.

1.  **Decode:** The block demodulates the BPSK symbol and decodes the bits.
2.  **Validate:** It checks the parity bit to ensure the Signal Field was decoded correctly.
3.  **Tag:** If valid, it attaches a tag to the sample stream.
    * **Tag Content:** A tuple containing `{Encoding Scheme, Frame Length}`.
4.  **Downstream Usage:** The subsequent block (`OFDM Decode MAC`) reads this tag to dynamically reconfigure its demodulator (e.g., switching from BPSK to QPSK) and its length counters for the payload that follows [cite: 2491246.2491248.pdf].

# Week 7 - 03/11/25

This week, we compressed the final stages of the project into a single intensive session. We moved from signal equalization to the final "digital" layers of the receiver: the **OFDM Decode MAC** module (Section 2.8) and the frame parsing logic. The goal was to understand how to recover the original user bits from the complex symbols and verify the receiver's operation against real-world IEEE 802.11 signals.

---

## Analysis of the OFDM Decode MAC Block

This module acts as the bridge between the physical layer (complex I/Q samples) and the data link layer (bits/bytes). We analyzed its internal operations—Slicing, De-interleaving, and Descrambling—answering the specific questions posed in the guide.

### 1. Digital Demodulation (Symbol Slicing)
The block processes the incoming OFDM symbols to extract raw bits.

* **How many symbols are demodulated at once?** The system processes vectors of **48 constellation symbols**.
* **Why 48?** The IEEE 802.11a standard uses a 64-point FFT. However, not all 64 "bins" carry data.
    * **12 Null Subcarriers:** Used for guard bands at the edges ($\pm27$ to $\pm31$) and the DC carrier (index 0).
    * **4 Pilot Subcarriers:** Indices $\pm7$ and $\pm21$, used for phase tracking.
    * **Remaining:** Exactly **48 subcarriers** are left for the Data Field.
* **Can you change this?** No. This value is hardcoded into the 802.11a physical layer specification. Changing it would break orthogonality with standard transmitters, as the receiver would try to interpret pilots or nulls as data, or miss actual data subcarriers.
* **The Process:** The code implementation (Symbol Slicing) takes the complex symbol (represented as an integer index in the constellation) and extracts the bits using bitwise operations. This depends on the modulation determined by the Signal Field (e.g., extracting 1 bit for BPSK, 4 bits for 16-QAM).

### 2. De-interleaving
We analyzed the `deinterleave` function in `decode_mac.cc` to understand its necessity.

**![Image here: Illustration of how interleaving spreads adjacent bits across different frequencies]**

* **What is it?** De-interleaving is the process of reordering bits to reverse the shuffling (interleaving) applied at the transmitter.
* **Why is it used?** Wireless channels suffer from frequency-selective fading. A deep fade might corrupt a block of adjacent subcarriers, causing a "burst error" of many consecutive corrupted bits. The Viterbi decoder (convolutional decoding) is excellent at fixing random, scattered errors but fails against burst errors.
* **How it works:** By reversing the transmitter's permutation, the receiver scatters the burst of errors across the entire message. This turns one "fatal" burst into many manageable single-bit errors that the Viterbi decoder can easily fix.
* **Implementation:** The code uses two permutations: a frequency mapping and a bit mapping (for higher-order modulations like 16-QAM) to place bits back into their original sequence.

### 3. De-scrambling
The final physical layer step is de-scrambling.

* **What is it?** The transmitter multiplies the data stream by a pseudo-random sequence generated by a Linear Feedback Shift Register (LFSR) using the polynomial $S(x) = x^7 + x^4 + 1$.
* **Why is it used?** It "whitens" the spectrum, preventing long sequences of `0`s or `1`s that would cause synchronization loss or high power peaks (PAPR).
* **How to de-scramble without the seed?** The receiver uses a clever trick involving the **Service Field**. The standard mandates that the first 7 bits of the Data Field must be zeros before scrambling.
    * Since Scrambling is an XOR operation ($Data \oplus Seed = Output$) and $Data=0$, the received bits *are* the seed ($0 \oplus Seed = Seed$).
* **Implementation:** The code reads these first 7 bits to initialize its own LFSR state, allowing it to generate the correct key to decode the rest of the payload.

---

## Framing, Signal Field, and Interoperability

After the physical decoding, we examined how the receiver interprets the frame structure (framing) and validated the system.

### 1. Delimiting the Frame (Signal Field)
We looked up how the receiver knows where a frame starts and ends.

* **The Signal Field:** This is a single OFDM symbol that immediately follows the preamble.
* **Information Sent:** It contains the **RATE** (4 bits) and **LENGTH** (12 bits).
* **Why is this necessary?** The payload (Data Field) can be modulated with high-density schemes (like 64-QAM) that the receiver cannot blindly guess. The Signal Field tells the receiver *how* to demodulate the payload (RATE) and *how long* the frame lasts (LENGTH).
* **Encoding:** To ensure this critical info is always readable, the Signal Field is always encoded with the most robust settings: **BPSK at rate 1/2**.

### 2. Validation and CRC
* **CRC Check:** After de-scrambling, the receiver calculates a CRC-32 checksum over the payload. It compares this to the transmitted FCS (Frame Check Sequence). If the residue matches the standard value (558161692), the frame is accepted; otherwise, it is dropped as corrupt.
* **Interoperability Results:** We tested the receiver with real 802.11a signals.
    * **Constellations:** We observed clear **16-QAM constellations** (as visualized in our report), proving that the frequency correction and symbol alignment (Sync Short/Long) were successful.
    **![16_QAM(./Figures/16_QAM)]**

    * **Wireshark:** We piped the decoded PDUs to Wireshark. The receiver successfully parsed Beacon frames, identifying the SSID "raspberrypi" and "UPorto," confirming the entire receive chain is functional.
    **![wireshark_ssid(./Figures/wireshark_ssid.png)]**

---

# Week 8 - 10/11/25

In this final week, we consolidated our understanding of the complete decoding process (Section 2.8) and verified the practical operation of the receiver. The objective was to confirm interoperability and ensure the data flow from the antenna to the final application is functional.

---

## Final Decoding Steps and Interoperability

### 1. Cyclic Redundancy Check (CRC)
After demodulation, de-interleaving, Viterbi decoding, and descrambling, the receiver obtains the raw MAC frame.
* **FCS Check:** The code calculates a CRC-32 checksum over the entire MAC header and payload.
* **Validation:** It compares this calculated value with the **Frame Check Sequence (FCS)** located at the end of the received frame. If they do not match, the frame is corrupted and is discarded (`checksum wrong -- dropping`). This ensures that only valid, error-free data is passed to the application layer.

### 2. Interoperability (Section 3)
We reviewed Section 3 of the article, which validates the receiver's design against real-world scenarios.
* **Compatibility:** The authors proved the receiver works with commercial off-the-shelf (COTS) hardware (e.g., Intel Ultimate-N 6300, Atheros chipsets) and IEEE 802.11p prototypes (Cohda MK2).
* **Frame Capabilities:** The receiver can successfully capture and decode:
    * **Management Frames:** Such as Beacons.
    * **Control Frames:** Such as RTS (Request to Send).
    * **Data Frames:** Standard payload packets.
* **Historical Limitation (Resolved):** The original paper noted a difficulty in capturing **CTS (Clear-to-Send)** frames. This was because CTS frames arrive immediately after RTS frames (within a SIFS interval), and the original "fixed-window" detection logic was "blind" while processing the RTS. As confirmed in our Week 2 analysis, the modern GitHub code solves this by using a continuous stream tagging architecture, allowing it to detect and decode back-to-back frames like RTS/CTS exchanges.

### 3. Practical Project Status
We have successfully finalized the practical implementation of the receiver.
* **Flowgraph:** The GNU Radio flowgraph (`802_a_receiver.grc`) is fully assembled and verified.
* **Hardware:** The system is configured with the SDR Source (PlutoSDR) and is processing signals in real-time.
* **Output:** We implemented data export functionality. The decoded packets are then sent to a wireshark connector block which then forwards it to a file sink, creating a PCAP file. This allows us to open the captured traffic in Wireshark, where we can inspect the MAC headers, verify the FCS, and analyze the payload data, proving the receiver is fully functional. We also implemented an alternative using a message debug block that allows us to observe on the terminhal what is being received, but due to the limitations of our hardware, the usage of such block normaly leads to the crash of gnu radio after receiving 2 or more frames. This also happens when we enable any log or debug on our blocks.
