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
  \[  a[n] = \sum_{k=0}^{N_{win}-1} s[n+k] \cdot s[n+k+16]^*  \]

**b) Identify the blocks that calculate the power**  
- Power is calculated over the same window:  
  \[  p[n] = \sum_{k=0}^{N_{win}-1} |s[n+k]|^2  \]

**c) Try to obtain a graph like Figure 2**  
- The normalized autocorrelation coefficient is:  
  \[  c[n] = \frac{|a[n]|}{p[n]}  \]  
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
