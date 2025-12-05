# Pluto-SDR-802.11a-receiver-
Pluto SDR 802.11a receiver work for UC of Digital Communications

Check the wiki pages for weekly updates! [Wiki](https://github.com/MafaldaBarros/Pluto-SDR-802.11a-receiver-/wiki)

Class T01
Group 02

# 📡 IEEE 802.11a OFDM Receiver – Week 1 Notes

This document summarizes the initial study of the article  
*“An IEEE 802.11a/g/p OFDM Receiver for GNU Radio”* (Bloessl et al., 2013)  
and answers the Week 1 questions from the project guide.  

---

## 🔎 1. IEEE 802.11a OFDM – Key Parameters

| Parameter                  | Value                | Notes                                  |
|----------------------------|----------------------|----------------------------------------|
| Channel bandwidth          | 20 MHz              |                                         |
| Sub-carrier spacing        | 312.5 kHz           |                                         |
| Total sub-carriers (FFT)   | 64                  |                                         |
| Data sub-carriers          | 48                  |                                         |
| Pilot sub-carriers         | 4                   |                                         |
| Unused sub-carriers        | 12                  | 1 DC + 11 guard (center + edges)        |
| Sampling rate              | 20 Msamples/s       |                                         |
| Useful symbol duration     | 3.2 µs              |                                         |
| Cyclic prefix duration     | 0.8 µs              | 16 samples                              |
| OFDM symbol length         | 4 µs (80 samples)   | 64 useful + 16 CP                       |

**Training sequences (frame prefix):**
- **Short Training Sequence (STS):** 16-sample pattern repeated 10× → used for frame detection and coarse frequency offset correction.
- **Long Training Sequence (LTS):** 64-sample pattern repeated 2.5× → used for symbol alignment and channel estimation.

---
## 📖 Q2. Read Section 2.1 and understand what it means  
**Where are the mentioned tags?**

- **Stream tagging:** used to annotate the incoming sample stream with metadata such as:  
  - start of frame  
  - frame length  
  - modulation and coding scheme (MCS)  
- These tags allow the downstream blocks to know *where a frame begins, how long it is, and how to decode it*.  

- **Message passing:**  
  - Allows blocks to exchange complete packets (headers + payload) asynchronously.  
  - Makes it easier to handle MAC frames once they are extracted from the stream.  

- **VOLK (Vector Optimized Library of Kernels):**  
  - Uses SIMD instructions to speed up operations at high sampling rates (10–20 Msps).  
  - Provides portable and optimized performance across different CPUs.  

---

## 🧮 Q3. Work through Section 2.2  
**a) Identify the blocks that calculate the autocorrelation**  
- The autocorrelation is computed using the repeated structure of the STS (every 16 samples).  
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
- A plateau in `c[n]` corresponds to the presence of the STS → this indicates the start of a frame.  

**d) Explore the effect of varying Nwin**  
- Small `Nwin`:  
  - Faster detection, but less robust (more noise sensitive).  
- Large `Nwin`:  
  - Smoother, more robust detection, but introduces delay.  
- The paper reports **Nwin = 48** as a good trade-off.  

---

## 📝 Additional Analysis of the Paper

- **Strengths:**  
  - First open-source 802.11a/g/p receiver in GNU Radio.  
  - Modular, reproducible, interoperable with commercial Wi-Fi NICs and DSRC devices.  
  - Comparable performance (PDR) to consumer hardware.  

- **Limitations:**  
  - Fixed number of samples forwarded after detection → CTS frames may be missed.  
  - Limited to BPSK/QPSK (channel equalization not sufficient for 16-QAM/64-QAM).  
  - Not yet optimized for high Doppler vehicular scenarios.  

- **Impact:**  
  - Shows that **GNU Radio + USRP** can serve as a low-cost experimental platform for Wi-Fi and vehicular networks research.  
  - Opens the door to reproducible PHY/MAC experiments in academia.  

---
# 📡 IEEE 802.11a OFDM Receiver – Week 2 Notes

## Understand the OFDM Sync Short block

- The OFDM Sync Short block performs frame detection.
- It decides when a Wi-fi (802.11a) frame starts.

- It does this by detecting the short training sequence at the beginning of every OFDM frame.
- The short preamble consists of a 16-sample pattern repeated 10 times → this repetition makes the autocorrelation high during this section.

## What the Block Actually Does

-It receives: 
- The raw complex samples s[n]; 
- The normalized autocorrelation coefficient c[n];

-It:

1. **Monitors** the value of `c[n]` to detect the presence of the short training sequence.
2. **Confirms the start of a frame** when `c[n]` remains above the threshold for **≥ 3 consecutive samples**, ensuring the detection is stable and not triggered by noise.
3. **Opens a “valve”** and forwards a fixed number of samples to the subsequent blocks in the receive chain.
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
# 📡 IEEE 802.11a OFDM Receiver – Week 3 Notes

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
  ( in wiki )
  
 # 📡 IEEE 802.11a OFDM Receiver – Week 4 Notes
 WIKI - MODLEE

 # 📡 IEEE 802.11a OFDM Receiver – Week 5 
 
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

   # 📡 IEEE 802.11a OFDM Receiver – Week 6

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
INCOMPLETE 

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

The SIGNAL field conveys the essential PHY-layer transmission parameters —  
namely the modulation scheme, convolutional coding rate, and PSDU length —  
allowing the physical layer to correctly configure its demodulation and decoding pipelines  
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

