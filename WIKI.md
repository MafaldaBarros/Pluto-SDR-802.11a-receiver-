# Pluto-SDR-802.11a-receiver-
Pluto SDR 802.11a receiver work for UC of Digital Communications

Check the wiki pages for weekly updates! [Wiki](https://github.com/MafaldaBarros/Pluto-SDR-802.11a-receiver-/wiki)

Class T01
Group 02

#  Week 1 Notes - IEEE 802.11a/g OFDM Receiver
The primary objective of this project is to implement an OFDM receiver for the IEEE 802.11a protocol using the ADALM Pluto SDR by Analog Devices. More information about the hardware can be found here. The code for this project is made using GNU Radio Companion, a popular open source platform with a variety of signal processing tools for software-defined radios.

In order to ultimately implement this receiver, we started by studying some core OFDM concepts and IEEE 802.11a specifications. Most of the research was based on the article “An IEEE 802.11a/g/p OFDM Receiver for GNU Radio”, in which the authors explain in detail how to implement a receiver chain using the GNU Radio Companion software.
In this page, we describe what we learned during the first week of the project.

---

## 1. IEEE 802.11a OFDM – Key Parameters

First, we need to understand the key parameters of the protocol and how this communication system is structured.

To clarify, IEEE 802.11a operates in the 5 GHz band, while the IEEE 802.11g standard operates in the 2.4 GHz band. Our goal is to be able to detect frames being sent in both bands. Other than this, the OFDM physical layer parameters are essentially the same:

| Parameter                  | Value                | Notes                                  |
|----------------------------|----------------------|----------------------------------------|
| Channel bandwidth          | 20 MHz              |                                         |
| Sub-carrier spacing        | 312.5 kHz           |  20 MHz / 64                                       |
| Total sub-carriers (FFT)   | 64                  |                                         |
| Data sub-carriers          | 48                  |                                         |
| Pilot sub-carriers         | 4                   |                                         |
| Unused sub-carriers        | 12                  | 1 DC + 11 guard (center + edges)        |
| Sampling rate              | 20 Msamples/s       |                                         |
| Useful symbol duration     | 3.2 µs              |                                         |
| Cyclic prefix duration     | 0.8 µs              | 16 samples                              |
| OFDM symbol length         | 4 µs (80 samples)   | 64 useful + 16 CP                       |

It is important to note that when 802.11 was standardized, in the early 2000s, computational power was much more limited than it is today. That being said, for this type of communication system, a 64-point FFT was chosen as a good tradeoff - small enough to be practical in terms of computational power and large enough to be effective for communication. In the IEEE 802.11 OFDM physical layer, not all 64 FFT subcarriers are used for data transmission; only 48 carry data, 4 are pilots, and the remaining 12 (guard bands + DC) are unused.
This design choice serves three key purposes:

  - First, the outer subcarriers are nulled to satisfy the spectral mask requirements, reducing out-of-band emissions and preventing interference with adjacent channels. 
  - Second, the central DC subcarrier is always deactivated because hardware impairments such as DC offset and local oscillator leakage introduce severe distortion at zero frequency.
  - Third, the presence of guard subcarriers improves synchronization robustness, noise estimation, and phase-tracking performance.

The article explicitly reflects this behavior, noting that the equalization block “removes DC, guard and pilot subcarriers,” leaving only the 48 data subcarriers for decoding.

Although this varies according to the region (Europe, US, Japan, etc.), the 2.4 GHz band IEEE 802.11g usually spans from 2.402 GHz to 2.483 GHz, giving us approximately 80 MHz of bandwidth. The channels are laid out as follows:
<img width="856" height="230" alt="image" src="https://github.com/user-attachments/assets/8dd459a3-e20e-4d66-9e88-f0c6635251e7" />

[Source: Cisco - WLAN Radio Frequency Design Considerations](url)

Since the center frequencies of each channel are spaced only 5 MHz from each other, while the channel bandwidth spans 20 MHz, this creates a channel overlap problem. Thus, channels 1, 6, and 11 are usually used in WiFi deployments to minimize interference, since they don't overlap. IEEE 802.11a has way more non-overlapping channel since the band it operates on has roughly 500 MHz.

In terms of modulations and data rates, both standards can transmit in the following modes:

  - 6, 9 Mbps (BPSK)
  - 12, 18 Mbps (QPSK)
  - 24, 36 Mbps (16-QAM)
  - 48, 54 Mbps (64-QAM)

**Training sequences (frame prefix):**
- **Short Training Sequence (STS):** 16-sample pattern repeated 10× → used for frame detection and coarse frequency offset correction.
- **Long Training Sequence (LTS):** 64-sample pattern repeated 2.5× → used for symbol alignment and channel estimation.

---
## 2. Section 2.1 - Overview
Reading through Section 2.1 of the mentioned article, it is important to understand:

Where are the mentioned tags?

- **Stream tagging:** Stream tagging: used to annotate the incoming sample stream with metadata (sampling frequency, carrier frequency, or timestamps) such as: 
  - start of frame  
  - frame length  
  - Modulation and Coding Scheme (MCS)  
- These tags allow the downstream blocks to know *where a frame begins, how long it is, and how to decode it*.  

- **Message passing:**  
  - Allows blocks to exchange complete packets (headers + payload) asynchronously.  
  - Makes it easier to handle MAC frames once they are extracted from the stream.  

- **VOLK (Vector Optimized Library of Kernels):**  
  - Uses SIMD (Single Instruction Multiple Data)  instructions to speed up operations at high sampling rates (10–20 Msps).  
  - Provides portable and optimized performance across different CPUs.  

---

## 3. Section 2.2 - Frame Detection
**a) Identify the blocks that calculate the autocorrelation**  
- The autocorrelation is computed using the repeated structure of the STS (short training sequence - every 16 samples).  
- Implemented with a summation block:
```math   
  a[n] = \sum_{k=0}^{N_{win}-1} s[n+k] \cdot s[n+k+16]^* 
```

**b) Identify the blocks that calculate the power**  
- Power is calculated over the same window:
```math  
  p[n] = \sum_{k=0}^{N_{win}-1} |s[n+k]|^2 
```

**c) Try to obtain a graph like Figure 2**  
- The normalized autocorrelation coefficient is:  
```math  
    c[n] = \frac{|a[n]|}{p[n]}  
```
- A plateau in `c[n]` corresponds to the presence of the STS,  indicating the start of a frame.

With this knowledge, we implemented this part of the receiver chain in GNU Radio Companion and plotted the autocorrelation output:

<img width="1363" height="678" alt="image" src="https://github.com/user-attachments/assets/b6203363-0172-48a8-ad0c-4380aa66ae77" />

<img width="1368" height="784" alt="image" src="https://github.com/user-attachments/assets/c7553e30-9e0f-4f98-bb78-0f612a5e9293" />

Using the recordings available on Moodle, we were able to briefly see what appears to be a plateau as seen in the article.

**d) Explore the effect of varying Nwin**  
- Small `Nwin`:  
  - Faster detection, but less robust (more noise sensitive).  
- Large `Nwin`:  
  - Smoother, more robust detection, but introduces delay.  
- The paper reports **Nwin = 48** as a good trade-off.  
---
# Week 2 Notes - IEEE 802.11a OFDM Receiver

This week, we continued studying the proposed receiver chain, understanding the necessary blocks for its implementation in GNU Radio. To add to the previously shown block diagram, we need the OFDM Sync Short block.
## Understand the OFDM Sync Short block

- The OFDM Sync Short block performs frame detection.
- It decides when an OFDM frame starts.
- It does this by detecting the short training sequence at the beginning of every OFDM frame.
- The short preamble consists of a 16-sample pattern repeated 10 times → This repetition makes the autocorrelation high during this section.

## What the Block Actually Does

-It receives (as described in the article *):  
- The raw complex samples s[n]; 
- The normalized autocorrelation coefficient c[n];

**Important**: In the new versions of the gr-ieee802-11, this block has been renamed to WiFi Sync Short, where a new input port (abs) was implemented. This is connected to the output of the moving average block (the autocorrelation). By having the complex value of the autocorrelation, the block uses the phase to estimate the sampling frequency offset for correction.

-The block:

1. **Monitors** the value of `c[n]` to detect the presence of the short training sequence.
2. **Confirms the start of a frame** when `c[n]` remains above the threshold for **≥ 3 consecutive samples**, ensuring the detection is stable and not triggered by noise.
3. Forwards a fixed number of samples to the subsequent blocks in the receive chain if the autocorrelation is above a configurable threshold.
4. **If no plateau is detected**, all incoming samples are **discarded**.

-This block does not perform any decoding. Its sole function is to detect the start of a frame and forward the corresponding samples to the next processing stages.

## Meaning of the Threshold

Threshold is applied to the normalized autocorrelation:

𝑐[𝑛]=∣𝑎[𝑛]∣ / 𝑝[𝑛]

-𝑎[n]: autocorrelation over lag 16
-p[n]: average signal power

Interpretation:
- High c[n] → strong repetitive pattern present → likely short preamble.
- Low c[n] → random samples/no frame.

The threshold defines how “clear” the repetition must be to consider it a frame.

## Effect of Changing the Threshold

| Threshold Setting | Effect | Consequence |
|------------------|--------|-------------|
| **Too Low**       | Noise may exceed the threshold | **False detections** → the receiver forwards samples that do not belong to any valid frame → later processing fails |
| **Too High**      | The actual preamble may not exceed the threshold | **Frames are not detected** → reduced detection probability |
| **Optimal Setting** | Only the true short preamble consistently exceeds the threshold | **Stable and reliable** frame start detection |


### Effect of Window Size (`Nwin`) on Preamble Detection

| Window Size (`Nwin`) | Effect | Consequence |
|----------------------|--------|-------------|
| **Small**            | Less smoothing of the autocorrelation | `c[n]` becomes noisier → harder to identify a stable plateau |
| **Large**            | Greater smoothing of the autocorrelation | The plateau becomes flatter → detection reacts more slowly to transitions |


### Limitations of the OFDM Sync Short Approach

| Limitation | Description | Consequence |
|-----------|-------------|-------------|
| **Fixed frame length** | The block forwards only a fixed number of samples after detecting the short preamble. | Only frames up to that maximum length can be decoded; longer frames are truncated. |
| **Cannot detect closely spaced frames** | While forwarding samples from one frame, the block does not look for a new plateau. | A following frame that arrives shortly after (e.g., **CTS** after **RTS**) may be missed. |
| **Parameter sensitivity** | Detection depends on proper threshold tuning and window size (`Nwin`) relative to SNR and channel conditions. | Poor parameter selection may lead to **false detections** or **missed frames**. |
| **Suboptimal detection method** | Uses autocorrelation instead of matched filtering to reduce computation. | Less robust at low SNR; matched filtering would perform better but requires more processing. |


---
#  Week 3 - IEEE 802.11a OFDM Receiver

## Understand the OFDM Sync Long block

- The OFDM Sync Long block performs applies frequency offset correction and symbol alignment. This is required because the sender and the receiver might work on slightly different frequencies.

## Understand the algorithm for frequency offset correction


- **Where is the frequency offset correction algorithm implemented?**  
  - The frequency offset correction is implemented inside the GNU Radio block OFDM Sync Long, which is part of the receiver chain described in Section 2.3 of the article.
  
  - This block This block operates immediately after frame detection and before the FFT stage and is responsible for:
     - Estimating the frequency offset between the sender and the receiver, using the short training sequence.
     - Applying the correction to each incoming sample by multiplying it with the complex exponential exp(i * n * df)
     - Performing symbol alignment using matched filtering with the long training sequence


- **Which parameters could you vary?**  
Averaging window length (N_short):  Controls how many samples are used for the sum, $$s[n] \cdot \overline{s[n+16]}$$
    - Smaller window → faster reaction, more sensitive to noise.

Peak detection thresholds
- Used during matched filtering of the long training sequence
- Changing these thresholds affects symbol alignment sensitivity.

Why is there a delayed input? Could you change that value and what would the impact be of doing that?

Why the delayed input exists?
The frequency offset estimator requires multiplying a sample with another sample **16 positions later**:

$$df = \frac{1}{16} \, \arg \left( \sum_{n=0}^{N_{\text{short}}-1-16} s[n] \, \overline{s[n+16]} \right)$$

This requires a delay line of exactly 16 samples inside the block.
Additionally, symbol alignment requires buffering 64 samples to perform matched filtering with the long training sequence.

#### Can you change the delay?
You can change it in code, but the algorithm depends on the **exact periodicity** of the OFDM preamble.
Changing the delay breaks the core assumption that the short training sequence repeats every 16 samples.

Impact of changing the delay

If the delay is modified:

- The correlation peak disappears.
- Frequency offset estimation becomes incorrect.
- Symbol alignment fails.
- FFT window is misaligned.
- The receiver fails to decode frames (PDR approaches zero).

  
 #  Week 4 - IEEE 802.11a OFDM Receiver
## 1. How is symbol alignment done (algorithm logic)?

Symbol alignment is performed using the Long Training Sequence (LTS), which is a known 64-sample pattern repeated 2.5 times in the IEEE 802.11 preamble. The receiver computes a **matched filter** (correlation) between the incoming samples and the known 64-sample LTS pattern:

$$
NP = argmax_3 \left( \sum_{k=0}^{63} s[n+k] \cdot \overline{LT[k]} \right)
$$

This correlation produces several sharp peaks. The block selects the **three largest peaks**, since the LTS contains 2.5 repetitions.

  - LTS = [64 samples] [64 samples] [half 64 samples], illustrated in figure 1 (G12,T0,T1).

The **last peak** corresponds to the **start of the final LTS symbol**.

The indices of the three peaks, NP = [n1, n2, n3] are calculated by:

$$
NP = argmax_3 \left( \sum_{k=0}^{63} s[n+k] \cdot \overline{LT[k]} \right)
$$

Once this index is known, the receiver knows the exact position of the first OFDM data symbol. Each symbol is then extracted by removing the 16-sample cyclic prefix (CP) and keeping the next 64 samples for the FFT, as showed in figure 1 in OFDM Symbols section.

<img width="1147" height="191" alt="image" src="https://github.com/user-attachments/assets/3cffa6a8-42be-459a-b608-db291c35dc40" />

## 2. Why is matched filtering used for symbol alignment but not for frame detection?

- Frame detection uses the **Short Training Sequence (STS)**, which consists of 10 repetitions of a 16-sample pattern. Because this sequence is highly repetitive, **autocorrelation** is enough to detect it reliably. Autocorrelation is also much cheaper computationally, which is necessary since frame detection must run at the full sample rate (10–20 Msps).

- Symbol alignment, in contrast, must be **extremely precise**. The 64-sample LTS has a unique structure that produces narrow, unambiguous peaks when matched filtering is applied. Autocorrelation does not provide the required timing accuracy for FFT alignment.

## 3. Code implementing equation (6)

$$
NP = argmax_3 \left( \sum_{k=0}^{63} s[n+k] \cdot LT[k] \right)
$$

In the implementation, this appears as a sliding correlation window. For each sample index n, the block multiplies the next 64 received samples by the conjugate of the LTS and sums the result.
The three largest correlation peaks are stored in NP, corresponding to the repeated LTS symbols.

## 4.Why is 64 added in expression (7)?

**nP = max(NP) + 64**

Max(NP) is the **start** of the last LTS symbol.
Each LTS symbol is exactly **64 samples long**.

Adding 64 moves the index from:

- the **start of the final LTS**, to -> the **start of the first OFDM data symbol** (as showed in figure 1)

## 5.What does the Stream to Vector block do?

The **Stream to Vector** block converts a continuous stream of samples into fixed-size vectors. In the context of Section 2.4, after the symbol alignment stage determines the exact start of each OFDM symbol, the receiver groups samples into **vectors of length 64**, which correspond to the FFT input.

Why this block is needed?
The FFT block in GNU Radio **does not operate on streams**.
It requires **vectors of exactly N samples** (here, N = 64 data samples per OFDM symbol).

references: [1]  E. Sourour, H. El-Ghoroury, and D. McNeill.
Frequency Offset Estimation and Correction in the
IEEE 802.11a WLAN. In IEEE VTC2004-Fall, pages
4923–4927, Los Angeles, CA, September 2004. IEEE.


 # Week 5 - IEEE 802.11a OFDM Receiver
 
## Understand Phase Offset Correction (Section 2.5)

### Why do you need to correct the phase?
- After the FFT, each subcarrier ideally has a fixed and predictable phase.
- However, in practice a phase offset appears because:

1. The transmitter and receiver sampling clocks are not perfectly synchronized.
Even small timing mismatches cause a linear phase rotation across subcarriers.

2. Symbol alignment is never perfectly exact.
If the FFT window is shifted slightly (even by one sample), the result is a frequency-dependent phase rotation.

- The consequence is:

    - all constellation points (BPSK, QPSK, 16-QAM, etc.) rotate in the complex plane
    - demodulation errors increase dramatically if this rotation is not removed
    
Therefore, the receiver must estimate and correct this phase offset for every OFDM symbol.

### Find the code that estimates the phase offset 

- The phase offset is estimated in the block OFDM Equalize Symbols.

- The algorithm uses the four pilot subcarriers defined by IEEE 802.11.
- Each pilot subcarrier carries a known BPSK symbol (±1), but the pilot pattern changes with the symbol index.

- The code does:
1. Extract the four pilot subcarriers from the current OFDM symbol.
2.Compare the received pilot phase with the expected pilot phase.
3. Perform a linear regression over pilot subcarrier indices because the phase offset is linear in frequency.
4. Estimate the slope and intercept of the phase rotation across all subcarriers.

- This slope/intercept pair gives the phase correction function for the entire OFDM symbol.

  ### How are you correcting the phase? (Which values are being changed?)

- The correction is applied directly to the frequency-domain subcarrier symbols after the FFT.

- Let:

  - X[k] = received subcarrier at index k

  - ϕ(k) = estimated phase rotation at that subcarrier

  - X_corr[k] = corrected subcarrier

- The correction is:
   X_corr[k] = X[k] * exp(-j * ϕ(k))
- What is being changed?

   - The angle of each complex subcarrier value is rotated back to where it should be.
   - The magnitude is not changed (only the phase).

- Since ϕ(k) is linear with frequency:
    ϕ(k) = slope * k + intercept
- This precisely removes the accumulated phase error across all 48 data subcarriers.
- After correction, the constellation points lie again on the expected positions, enabling correct demodulation

   # Week 6 - IEEE 802.11a OFDM Receiver

## Understand the OFDM Equalize Symbols module 
### What does it do?

-The OFDM Equalize Symbols block is the first block operating in the frequency domain after the FFT. It performs the following functions:
  1. Phase offset correction using the four pilot subcarriers.
  2.Channel magnitude correction (simple equalization based on a sinc-shaped assumption).
  3. Removal of DC, guard, and pilot subcarriers.
  4. Extraction of the 48 data subcarriers from the 64-point FFT output.
  5. Preparation of clean frequency-domain symbols for demodulation in the OFDM Decode Signal block.

- In short, this block “cleans” the FFT output and makes the data symbols ready for decoding.

### Why is the implementation limited to deal with BPSK and QPSK modulation?

- The limitation comes from the simplified channel equalization used:

- The current implementation assumes a sinc-shaped magnitude response of the subcarriers.
- This approximation is not accurate enough for constellations where amplitude carries important information (e.g., 16-QAM, 64-QAM).
- Incorrect magnitude equalization would distort these constellations and make decoding unreliable.
- BPSK and QPSK depend mostly on phase, so the simplified equalizer works well enough for them.
- Higher-order QAM requires more advanced channel estimation, which is not implemented.

 ### Which other functions are performed in this block?

- Besides equalization, the block also:

    - Uses pilot subcarriers to estimate and correct linear phase offset.
    - Removes non-data carriers (DC, guard bands, pilots).
    - Normalizes and outputs only the 48 data subcarriers.
    - Propagates tags such as the symbol index, needed for pilot tracking.
    - Passes corrected symbols to OFDM Decode Signal for demapping and decoding.

 ## Work through Section 2.7

# Section 2.7 — Signal Field Decoding
# How does an 802.11 frame begin?

An 802.11 OFDM frame always starts in this order:

1. **STS** – short training sequence used only to detect that a frame is arriving.  
2. **LTS** – long training sequence used to find the exact start of the symbols.  
3. **SIGNAL Field** – the first symbol that contains useful information for the receiver.  
4. **DATA Field** – the MAC header and payload.

So, the SIGNAL field is the first symbol that tells the receiver how to read the rest of the frame.

# How do you delimit a frame? (Remembering Computer Networks)

<img width="722" height="296" alt="image" src="https://github.com/user-attachments/assets/340e3b06-1680-4e18-8e8d-66bac3b1a588" />

# What is the Signal Field?

The Signal Field is one OFDM symbol transmitted right after the training sequences.  
It is sent using:

- **BPSK modulation**
- **Rate 1/2 convolutional coding**

This makes the Signal Field extremely robust, since it must always be decoded correctly for the frame to be understood.

The Signal Field contains:

- **RATE** → modulation and coding used for the payload  
- **LENGTH** → number of bytes in the payload  
- **1 parity bit** → quick correctness check  
- **6 tail bits** → flushes the convolutional decoder  

# Why is this necessary?

The receiver cannot decode the payload unless it knows:

- the modulation (BPSK/QPSK/QAM)  
- the coding rate  
- the total payload length  

The SIGNAL field conveys the essential PHY-layer transmission parameters (namely the modulation scheme, convolutional coding rate, and PSDU length) allowing the physical layer to correctly configure its demodulation and decoding pipelines  
for the subsequent OFDM data symbols.


# How is that information encoded?
The Signal Field consists of 24 bits:
- 4 bits  → RATE (modulation + coding of the payload)
- 12 bits → LENGTH (payload size in bytes)
- 1 bit   → Parity
- 6 bits  → Tail bits (flush the convolutional encoder)


  
Then:
1. Convolutionally encoded (rate 1/2), producing 48 coded bits
2. Interleaved to improve robustness against burst errors
3. Modulated using BPSK (each bit → +1 or –1)
4. Mapped onto a single OFDM symbol (48 data subcarriers)


   #  Week 7 - IEEE 802.11a OFDM Receiver

## Work through decoding the payload (Section 2.8)

### Which abstract receiver blocks are within the module OFDM Decode MAC?

Before reaching the **OFDM Decode MAC** module, the receiver has already converted the RF waveform into corrected, equalized OFDM data symbols.

Inside this block, the receiver performs the **final digital baseband reconstruction of the payload**.  
According to Section 2.8 of the article, the *OFDM Decode MAC* module contains the following abstract processing blocks:

#### **Demodulator**
- Converts each of the **48 constellation points** into **soft bits** based on the modulation scheme (BPSK, QPSK, etc.).

#### **De-interleaver**
- Reverses the bit permutations applied at the transmitter, according to the **Modulation and Coding Scheme (MCS)**.

#### **Convolutional Decoder + Puncturing Handler**
- Removes redundant bits due to convolutional encoding on the transmitter side, allowing different code rates on the same transmission.

#### **Descrambler**
- Uses the first **seven bits of the SERVICE field** (which are always zero) to determine the scrambler's initial state and remove the scrambling sequence.
  
Together, these operations reconstruct the **original MAC-layer payload** from the equalized OFDM symbols.

---
### How many constellation symbols are demodulated at once? Why?

The block demodulates **48 constellation symbols at once**, because each OFDM symbol contains:

- 64 total subcarriers  
  Subtracting:
    - 4 pilot carriers  
    - 1 DC carrier  
    - 11 guard/null carriers  

→ leaves **48 data subcarriers**, each carrying **one constellation point**. (BPSK or QPSK)
This number is **defined by the IEEE 802.11a/g/p standard**, so it cannot be freely changed.

#### Can this be changed?

Yes, this can be changed. One can have multiple constellation points per subcarrier (QAM).

#### Impact of changing the number of demodulated carriers

- We can send more data
- The frame decoding is more complex and heavy
- The probability of error is higher, needing a higher SNR to keep the same FER. 
---
### What is the actual process of digital demodulation?

#### **Input**
A vector of **48 complex constellation points**, already equalized and phase-corrected.

#### **Output**
A vector of **soft bits** — floating-point values representing the likelihood of each bit being 0 or 1.

#### **How the Conversion Works**

1. Each constellation point is compared to the ideal constellation positions defined by the modulation scheme  
   (e.g., BPSK, QPSK, 16-QAM, 64-QAM).

2. The distance between the received point and each ideal point determines the probability of the bit values.

3. For each constellation point:
   - **BPSK → 1 soft bit**  
   - **QPSK → 2 soft bits**  
   - **16-QAM → 4 soft bits**  
   - **64-QAM → 6 soft bits**

Soft bits are used because **Viterbi decoding performs better with likelihood metrics** rather than hard 0/1 decisions.

---
### Understanding De-interleaving

De-interleaving is the reversal of the transmitter’s interleaving process.

Before reaching this stage, the receiver has already recovered the 48 data subcarriers for each OFDM symbol and corrected for frequency, timing, phase, and channel effects.  
However, the bits coming from demodulation **are not yet in their original order**, because 802.11 applies *interleaving* before transmission.

### **Why Interleaving Exists**

Interleaving is used to protect the transmission against **burst errors**.  
In a wireless channel, noise or fading often affects *several consecutive bits* at once.

If these bits stayed together inside the same codeword:

- a single fade could corrupt multiple bits of the same word,  
- exceeding the error-correction capability of the convolutional code,  
- making the codeword impossible to decode.

To avoid this problem, interleaving **spreads consecutive bits across different codewords and different subcarriers**.

This means:

> Instead of having multiple errors concentrated in one codeword, the errors are *distributed* across several codewords, where each codeword only receives a small number of errors.

### **What De-interleaving Does**

At the receiver:

- The **inverse permutation** is applied to restore the original bit ordering.
- This operation depends on the selected **Modulation and Coding Scheme (MCS)**.
- Once de-interleaved, the bits are again grouped in their correct sequence for input to the convolutional decoder.

Correct de-interleaving is essential because:

- If bits are fed in scrambled form, the convolutional code structure breaks, and decoding fails.

---
### Understanding Descrambling

#### **Why scrambling exists**
A pseudo-random scrambler is used in IEEE 802.11 to prevent long sequences of identical bits.  
Such sequences would negatively impact:
- synchronization,
- clock recovery,
- and spectral efficiency.

Scrambling ensures that the transmitted bitstream appears statistically random, even if the payload contains repeated patterns.

#### **How descrambling works**
The receiver implements the same **7-bit Linear Feedback Shift Register (LFSR)** used by the transmitter.  
To correctly descramble the bitstream, the receiver must determine the **initial state** of the scrambler.

#### **How the receiver deduces the scrambler state**
The receiver can recover the scrambler seed because of a key property of the IEEE 802.11 PHY:

- The first **7 bits of the SERVICE field** are **always zero**.
- After convolutional decoding, these bits appear at the start of the payload.
- The receiver compares these 7 bits against a lookup table of possible LFSR outputs.
- This allows the receiver to infer the **initial scrambler state** used by the transmitter.
- Once the correct seed is known, the receiver generates the same pseudo-random sequence and **XORs** it with the received bits to recover the original payload.

#### **Which field is used?**
- **SERVICE field (bits 0–6)**  
  These bits are always zero and are specifically used to deduce the scrambler's initial state.

---
#### **Decoded frames**
At this point, we started experimenting with the full receiver chain shown below:

<img width="770" height="435" alt="image" src="https://github.com/user-attachments/assets/2de82c0f-91c2-4489-a436-b5bac955d5b3" />

Using the Message Debug block, we can observe the decoded packets as they arrive in the terminal. For the data source, the recordings available on Moodle were used.
Below is an example of a packet that was received:

decode_mac :info: encoding: 4 - length: 307 - symbols: 26
parse_mac :info: length: 303
***** VERBOSE PDU DEBUG PRINT ******
((address 3 . 54:8a:ba:17:9e:0c) (address 2 . 54:8a:ba:17:9e:0c) (address 1 . ff:ff:ff:ff:ff:ff) (sequence number . 2424) (ssid . raspberrypi) (subtype . Beacon) (type . management) (duration . 0) (dlt . 105) (frame bytes . 307) (encoding . 4) (snr . 22.8954) (nominal frequency . 2.45e+09) (frequency offset . 25875.3) (beta . 2.46072) (csi . #[(0.0280669,0.0416872) (0.04785,0.0407505) (0.057741,0.038741) (0.07745,0.0438908) (0.0822286,0.0513226) (0.0841846,0.0667423) (0.0804586,0.0736604) (0.0725615,0.063261) (0.0585334,0.0662361) (0.0408872,0.053024) (0.0346504,0.0439203) (0.0195954,0.0185439) (0.019837,0.000162425) (0.0151125,-0.0241559) (0.0272753,-0.0559146) (0.0475085,-0.0773479) (0.0715504,-0.0969507) (0.0916895,-0.101854) (0.126404,-0.106131) (0.144398,-0.105959) (0.173585,-0.0840717) (0.18325,-0.0670141) (0.17941,-0.0401931) (0.184233,-0.0176918) (0.187823,0.0028242) (0.173624,0.0289136) (0.12589,0.0622309) (0.103026,0.0633536) (0.0849338,0.062489) (0.063937,0.0534365) (0.0525451,0.0440301) (0.0467674,0.0263795) (0.0445494,0.0265091) (0.0334592,0.0162587) (0.0577812,0.0110455) (0.0525863,0.00947341) (0.0661837,0.0119797) (0.067101,0.0250755) (0.0614101,0.0317764) (0.0695245,0.0345314) (0.0546006,0.0411687) (0.0495476,0.0428713) (0.0418433,0.0498351) (0.0314546,0.0421788) (0.0227848,0.0397646) (0.00903822,0.0238152) (0.0067861,0.0127072) (0.0039367,-0.00542243) (0.00857316,-0.0192193) (0.00968787,-0.028923) (0.0162549,-0.0402511) (0.0175329,-0.0462985)]))
pdu length =        303 bytes
pdu vector contents = 
0000: 80 00 00 00 ff ff ff ff ff ff 54 8a ba 17 9e 0c 
0010: 54 8a ba 17 9e 0c 80 97 59 31 57 ee f7 0a 00 00 
0020: 64 00 11 05 00 0b 72 61 73 70 62 65 72 72 79 70 
0030: 69 01 05 24 b0 48 60 6c 03 01 64 05 04 00 01 00 
0040: 00 07 3c 50 54 20 24 01 14 28 01 14 2c 01 14 30 
0050: 01 14 34 01 14 38 01 14 3c 01 14 40 01 14 64 01 
0060: 1b 68 01 1b 6c 01 1b 70 01 1b 74 01 1b 78 01 1b 
0070: 7c 01 1b 80 01 1b 84 01 1b 88 01 1b 8c 01 1b 0b 
0080: 05 12 00 0a 8d 5b 2d 1a 6f 08 03 ff ff 00 00 00 
0090: 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 00 
00a0: 00 00 30 14 01 00 00 0f ac 04 01 00 00 0f ac 04 
00b0: 01 00 00 0f ac 02 28 00 3d 16 64 05 04 00 00 00 
00c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
00d0: 7f 08 00 10 00 00 00 00 00 40 bf 0c 32 78 89 33 
00e0: fa ff 00 00 fa ff 00 00 c0 05 01 6a 00 fc ff c3 
00f0: 04 02 ca ca ca dd 18 00 50 f2 02 01 01 88 00 03 
0100: a4 00 00 27 a4 00 00 42 43 5e 00 62 32 2f 00 85 
0110: 1e 00 00 8f 00 0f 00 ff 03 40 00 46 45 55 50 2e 
0120: 49 2e 30 33 32 32 30 31 2e 32 00 12 00 00 2c 
************************************

We can already observe some useful information:

**Network Details**:

  - **SSID:** raspberrypi (bytes at 0x002C: 72 61 73 70 62 65 72 72 79 70 69)
  - **Device:** Probably a Raspberry Pi acting as an access point
  - **MAC Address:** 54:8a:ba:17:9e:0c (both source and BSSID)
  - **Broadcast to:** ff:ff:ff:ff:ff:ff (all devices)
  
**Frame Structure Breakdown:**

  - 80 00 = Frame control (Beacon)
  - 00 00 = Duration
  - ff ff ff ff ff ff = Destination (broadcast)
  - 54 8a ba 17 9e 0c = Source/BSSID (Raspberry Pi)
  - 59 31 57 ee f7 0a = Timestamp
  - 64 00 = Beacon interval (100 TUs = 102.4ms)
  - Extra: Looking at the end of the vector contents, we can see the ASCII string 46 45 55 50 2e 49 2e 30 33 32 32 30 31 2e 32 -> FEUP.I.032201.2
  
This proves that this chain is indeed capable of successfully detecting and decoding 802.11a frames at 2.45GHz, and we are able to extract useful information.


   #  Week 8 - Experimenting with Pluto

At this point, we ran the same program as before, but now using the actual Pluto SDR instead of the recordings.

We created the channel_scanner. This program plots the constellation and displays a drop-down box for selecting the current channel the SDR is scanning.

<img width="482" height="426" alt="image" src="https://github.com/user-attachments/assets/790aa7f4-11fb-4f79-93f3-00feeaf040ba" />

<img width="1044" height="672" alt="image" src="https://github.com/user-attachments/assets/6adcab81-543a-4346-9287-7ff09b7870f4" />

We were able to successfully record various frames in different channels, as seen in the wireshark results. The following were recorded in the 5 GHz band, in Channel 100.

<img width="961" height="551" alt="image" src="https://github.com/user-attachments/assets/1ac338ad-5bbf-4858-862a-31bc7c78cee9" />

We also noticed that sometimes the program needs to be running for quite an amount of time (several minutes) for it to detect any frames.
