# Week 1 - 20/10/25

This week's work focused on researching the fundamental parameters of the IEEE 802.11a OFDM system and analyzing the initial frame detection stage of the GNU Radio receiver presented in the paper.

---

## IEEE 802.11a

IEEE 802.11a operates within a 20 MHz bandwidth and employs a 64-point IFFT/FFT, which establishes a total of 64 subcarriers. The spacing between these subcarriers is 0.3125 MHz (or 312.5 kHz). Of the 64 subcarriers, 52 are actively used: 48 are designated as data subcarriers for information transmission, and 4 are pilot subcarriers used for phase and frequency tracking. The remaining 12 subcarriers are unused; these include the central DC subcarrier (index 0), which is nulled to avoid issues with DC offsets in transceivers, and 11 subcarriers at the band edges (indices ±27 to ±31), which act as guard bands to reduce interference with adjacent channels.

The system's sampling rate is 20 MHz. The useful duration of one OFDM symbol (the IFFT/FFT period) is 3.2 µs, which corresponds to 64 samples. To combat multipath interference, a cyclic prefix (CP) with a duration of 0.8 µs (or 16 samples) is added to the beginning of each symbol. This brings the total OFDM symbol duration to 4.0 µs (64 samples + 16 CP samples = 80 total samples).

Each 802.11a frame begins with a physical layer preamble, or prefix, which is 16 µs long in total. This prefix is divided into two parts: a Short Training Field (STF) and a Long Training Field (LTF). The STF consists of 10 repetitions of a short training symbol, each 0.8 µs long, for a total STF duration of 8 µs; it's used for signal detection, gain control, and coarse frequency offset estimation. The LTF follows, also lasting 8 µs, and consists of a 1.6 µs guard interval followed by two 3.2 µs long training symbols. The LTF is used for fine-tuning frequency offset and for channel estimation.

## Section 2.1

The OFDM receiver's architecture is divided into two functional parts, the frame detection and frame decoding. The first part is responsible for detecting the start of a frame, while the second part handles the actual decoding process. The key to this rapid performance is the Vectorized Library of Kernels (VOLK), which uses Single Instruction Multiple Data (SIMD) instructions to speed up signal processing tasks. To coordinate the flowgraph, Stream Tagging is used to attach metadata to samples, allowing blocks to signal the start of an OFDM frame and later pass the frame's length and encoding scheme. Finally, Message Passing is also used as an alternative way to handle packet-based data, encapsulating arbitrary information, which can be utilized to complete packets and switch to stream-based processing.

## Section 2.2

The first stage of the OFDM receiver is responsible for Frame Detection, leveraging the known characteristics of the IEEE 802.11a/g/p short preamble, which features a pattern that spans 16 samples and repeats ten times. To detect this, an autocorrelation function with a 16-sample lag is implemented using two main branches, as seen in the block diagram. The top branch calculates the autocorrelation numerator: the incoming signal is split, with one path delayed by 16 samples and the other path complex conjugated before they are multiplied together. This result is then processed by a Decimating FIR Filter block, which, by having its Decimation parameter set to 1, is cleverly used as a moving window summer to perform the summation step of the autocorrelation. Simultaneously, the lower branch calculates the signal's average power, which is ultimately used to normalize the autocorrelation coefficient. This normalization makes the frame detection reliable and prevents false positives that could be caused by high signal power.

At last, we explored the effect of our variable `window_size` ($N_{Win}$) parameter, which controls the length of the moving average filter for frame detection. We observed a clear trade-off: a small value like 16 resulted in noisy spikes, leaving an unstable plateau, a large value like 96 produced a very smooth but dispersed plateau of the signal, making the frame's start and end indistinct. As for the value of 48, it provided an optimal balance, helping to generate a clean and stable plateau.

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

# Week 3

For Week 3, we continued our theoretical analysis, focusing entirely on the **OFDM Sync Long** block and its two critical functions. This followed our work from last week, as we are still blocked on the practical implementation due to the module download issue. The main objective was to deeply understand the algorithms for **Frequency Offset Correction (Section 2.3)** and **Symbol Alignment (Section 2.4)**. We analyzed the implementation of the frequency offset algorithm, identified its core 16-sample delay as a non-tunable constant, and distinguished it from the external Delay(240) block, which we identified as a system-level pipeline buffer. We also identified the tunable parameters for this algorithm and investigated the modern "tagging" implementation from the repository, which solves the limitations of the 2013 paper's design.

---

## Analysis of the OFDM Sync Long Block

### 1. Block Function and Objective
The OFDM Sync Long block is the second and final stage of synchronization. It is a stateful, multi-mode block that receives a chunk of samples from the Sync Short block (via the Delay(240) block) after a frame has been detected.

This block has two distinct and sequential responsibilities:

1.  **Frequency Offset Correction (Section 2.3):** First, it re-uses the short preamble sequence (which is at the start of the chunk) to perform a high-precision calculation of the frequency offset between the transmitter and receiver.
2.  **Symbol Alignment (Section 2.4):** Second, it uses the long preamble sequence (which follows the short) to find the exact starting sample of the first data symbol (the Signal Field).

After performing these two tasks, the block transitions to its "data" state: it strips the preambles, removes the 16-sample cyclic prefix from all subsequent 80-sample symbols, and streams the resulting 64-sample data chunks to the FFT.

### 2. Algorithm 1: Frequency Offset Correction
This section addresses the algorithm from Section 2.3 of the paper, which is the first task performed inside the Sync Long block.

**Where is it implemented?**
The algorithm is implemented inside the **OFDM Sync Long** block. It operates on the first part of the sample chunk (the 160-sample short preamble) that it receives from its data input.

**The Algorithm's 16-Sample "Delayed Input"**
The core of the algorithm is a complex autocorrelation with a 16-sample delay, as shown in Equation 4:

$$df = \frac{1}{16} \arg\left(\sum s[n] \cdot \text{conj}(s[n+16])\right)$$

This 16-sample delay is not a tunable parameter; it is a fundamental constant mandated by the IEEE 802.11 standard.

* **Why 16?** The short preamble is defined as a 16-sample pattern that repeats 10 times. In an ideal system, sample $s[n]$ would be identical to sample $s[n+16]$.
* **The Math:** A frequency offset ($\Delta f$) between the transmitter and receiver causes a constant, accumulating phase rotation ($\theta$) with every sample. The total phase rotation after 16 samples is exactly $16\theta$.
* **How it Works:** By multiplying a sample $s[n]$ by the complex conjugate of $s[n+16]$, the algorithm isolates this $16\theta$ phase difference. (The unknown underlying signal $s[n]$ is canceled out, leaving only a phase rotation and the signal's power).
* **The Result:** By summing (averaging) this calculation over the preamble, the block gets a single, stable complex number whose phase angle is $16\theta$. The algorithm takes the `arg()` of this number to get the phase, and then divides by 16 to get the exact per-sample phase offset ($df$).

*What would be the impact of changing that 16-sample value?*

You cannot change this value. If you changed the delay to 15 or 17, the algorithm would be comparing two samples that have no correlation ($s[n]$ and $s[n+17]$). The result of the `sum()` would be random noise, the calculated $df$ would be meaningless, and all subsequent decoding (FFT, etc.) would fail.

### 3. The External Delayed Input: The Delay(240) Block

The most confusing part of the diagram (Figure 1) is the external Delay(240) block. This delay is **not** related to the 16-sample delay in the frequency correction algorithm.

This is a **software pipeline buffer** designed to manage the "lookahead" time in the GNU Radio scheduler.

Here is the flow:
* **Fast Path (Detection):** The raw signal goes directly into the Frame Detection logic. The Sync Short block finds a plateau early (e.g., at sample 50) and sends a trigger message to the Sync Long block.
* **Slow Path (Processing):** The raw signal also goes into the Delay(240) block.
* **The "Lookahead":** When the Sync Short sends its trigger at Time 50, the Sync Long block's data input (the slow path) is still seeing sample $50 - 240 = -190$ (i.e., noise from the past).
* **Purpose:** This gives the GNU Radio scheduler 190 samples of "warning time" to stop the Sync Long block (which was just processing noise) and schedule it to process the incoming frame before that frame's data arrives. 240 samples (equal to 3 OFDM symbols) is a safe, robust buffer size to ensure this software-based scheduling happens without losing samples.

**What would be the impact of changing that 240-sample value?**

* **Making it Smaller (e.g., 160):** This would reduce the pipeline latency, but it also reduces the "warning time" for the scheduler. If the Sync Short triggers at sample 50, the scheduler would only have $160 - 50 = 110$ samples to react. This is "tighter" and risks missing the start of the frame if the system has any scheduling lag.
* **Making it Larger (e.g., 500):** This would be even safer, giving the scheduler more time. However, it would add unnecessary latency to the entire pipeline.

**Conclusion:** The 240-sample delay is a system-level tuning parameter for pipeline stability, balancing latency against the risk of data loss due to software scheduling delays.

### 4. Other Configurable Parameters

The main tunable parameter for the frequency offset algorithm is the averaging window ($N_{short}$ in Equation 4). The paper uses the length of the short preamble. For faster processing, a smaller window could be used, however, at the cost of susceptibility to noise, leading to a less precise frequency offset correction.

### 5. The Modern Repository Implementation (The "Fix")

The complex "fast path / slow path / delay" architecture described in the 2013 paper is the original implementation. The author's gr-ieee802-11 repository now uses a much cleaner, more modern approach.

This new design removes the Delay(240) block and the "fixed chunk" logic entirely.

The Sync Short block no longer acts as a "valve." Instead, it acts as a **"tagger."**

1.  It continuously monitors the stream and, upon detection, adds a **stream tag** (e.g., `frame_start`) to the exact sample in the original, undelayed stream where the preamble began.
2.  The Sync Long block simply receives this continuous, tagged stream. When it sees a `frame_start` tag, it knows to start its internal process (frequency correction, then symbol alignment) at that exact sample.

This updated "tagging" architecture is far superior, as it solves the "back-to-back" frame (RTS/CTS) problem and greatly simplifies the flowgraph logic.

---

# Week 4

This week, we analyzed the high-precision symbol alignment process from Section 2.4. The study focused on how the OFDM Sync Long block uses a matched filter on the Long Training Sequence (LTS) to find the exact starting sample of the Signal Field. We investigated the core design trade-off between this precise, computationally expensive matched filter and the "good enough," cheaper autocorrelation used for initial frame detection. The analysis also covered the specific implementation details, such as adding 64 to the final peak index to find the data start, and the function of the Stream to Vector block in preparing the aligned sample stream for the FFT.

---

## Analysis of Symbol Alignment (Section 2.4)

This section details the second, high-precision phase of synchronization performed by the OFDM Sync Long block, which occurs immediately after the frequency offset has been corrected.

### 1. Symbol Alignment Algorithm Logic

The algorithm's goal is to find the exact sample index that marks the end of the preamble and the start of the first data-carrying symbol (the Signal Field). It achieves this by using the Long Training Sequence (LTS), which is 160 samples long (a 32-sample guard interval followed by two 64-sample training symbols).

The logic follows these steps:
* **Matched Filtering:** The algorithm performs a matched filter operation. It slides a known, 64-sample "template" of the long preamble pattern ($LT[k]$) across the received, frequency-corrected signal ($s[n+k]$).
* **Correlation Score:** At every sample position n, it calculates a correlation score, which is the sum of the products of the template and the signal (this is the `sum(...)` part of Equation 6).
* **Peak Detection:** When the template perfectly aligns with the real LTS in the signal, this score "pings" with a very high, narrow peak (as shown in Figure 3). The LTS repeats twice, so this creates two distinct peaks.
* **Robustness (The $\arg \max_3$):** To protect against random noise creating a single false peak, the algorithm doesn't just find the single highest peak. It finds the indices of the three highest peaks in the search area.
* **Final Peak Selection ($\max(N_p)$):** From this list of 3 candidates, the algorithm selects the one with the latest time index (the `max()` function). This ensures it synchronizes to the second and final repetition of the long training symbol, which is the most stable and most recent timing reference. This value ($\max(N_p)$) is the sample index for the start of the final 64-sample training symbol.

*Why do you add 64 in Equation 7?*

Equation 7 is: $$n_p = \max(N_p) + 64$$

The value $\max(N_p)$ gives the algorithm the start of the final 64-sample training symbol. However, the data does not begin until this final training symbol is finished.

The + 64 is added because the long training symbol has a length of 64 samples. By adding 64 to the start index, the algorithm calculates the exact sample index ($n_p$) for the end of the preamble, which is the precise start of the Signal Field.

### 2. Matched Filtering vs. Autocorrelation

This is a key design choice based on a trade-off between processing cost and required precision.

*Why Not Matched Filtering for Frame Detection (Sync Short)?*

* **Job:** The Sync Short block must run continuously on the full 20 Msps (20 million samples/second) stream from the SDR.
* **Cost:**
    * The Autocorrelation ($s[n] \times \text{conj}(s[n+16])$) is computationally cheap. It requires only one complex multiplication per sample.
    * A Matched Filter is computationally expensive. A 64-sample matched filter requires 64 complex multiplications per sample.
* **Conclusion:** It is computationally infeasible to run an expensive matched filter on the full-speed, continuous stream. The 1-tap autocorrelation is "good enough" for the coarse job of just detecting that a frame has arrived.

*Why Matched Filtering for Symbol Alignment (Sync Long)?*

* **Job:** The Sync Long block runs only after a frame is detected. It processes only a small, fixed-size chunk of data, not the continuous stream.
* **Precision:** This task requires exact precision. Being off by even one sample would ruin the FFT.
* **Conclusion:** Because the block is "offline" (i.e., not running all the time) and only processing a small subset of data, the high computational cost is acceptable. The system "spends" its processing budget here to gain the high precision that only a matched filter can provide.

The paper states this directly in a footnote (Section 2.2) and in Section 2.4:
> "As this block needs only act on a subset of the incoming sample stream, and as the alignment has to be very precise, we employ matched filtering for this operation."

### 3. The "Stream to Vector" Block

This block is a standard GNU Radio utility that acts as a grouper or buffer.

* **Input:** The Sync Long block (once aligned) outputs a continuous stream of individual samples (the 64-sample data chunks, one after another, with no prefixes).
* **Output:** The FFT block, however, cannot operate on one sample at a time. Its FFT Size: 64 configuration means it must receive data in vectors of 64 samples.
* **Function:** The Stream to Vector block (with Num Items: 64) sits in the middle. It collects samples from its input stream one by one until it has 64. It then bundles these 64 samples into a single "vector" and sends this vector as one item to the FFT block. It then repeats this process for the next 64 samples, and so on.

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

