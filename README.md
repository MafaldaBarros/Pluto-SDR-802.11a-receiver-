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

-1)Monitors c[n] to detect when the short training sequence is present.
-2)When c[n] stays above a threshold for ≥ 3 samples, it assumes a frame has started. This ensures the detection is stable and not triggered by noise.
-3)It opens a “valve” and forwards a fixed number of samples to the next blocks.
-4)If no plateau is detected → samples are discarded.
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

| Threshold Setting | Efeito | Consequência |
|------------------|--------|--------------|
| **Demasiado Baixo** | Ruído pode ultrapassar o limiar | **Deteções falsas** → o receptor encaminha amostras que não pertencem a nenhum quadro → falhas no processamento seguinte |
| **Demasiado Alto** | O pré-âmbulo verdadeiro pode não atingir o limiar | **Quadros não detetados** → redução da probabilidade de deteção |
| **Ajuste Ótimo** | Apenas o pré-âmbulo curto verdadeiro excede o limiar de forma consistente | **Deteção estável e fiável** do início do quadro |

### Efeito da Dimensão da Janela (`Nwin`) na Deteção do Pré-Âmbulo

| Tamanho da Janela (`Nwin`) | Efeito | Consequência |
|---------------------------|--------|--------------|
| **Pequena**               | Menos suavização da autocorrelação | `c[n]` fica mais ruidoso → mais difícil identificar um plateau de forma estável |
| **Grande**                | Maior suavização da autocorrelação | O plateau torna-se mais “achatado” → deteção fica mais lenta e menos reativa a transições |

### Limitações do Método de Deteção (OFDM Sync Short)

| Limitação | Descrição | Consequência |
|----------|------------|--------------|
| **Tamanho máximo do quadro** | O bloco encaminha apenas um número fixo de amostras após detetar o pré-âmbulo curto. | Apenas quadros até um certo tamanho podem ser decodificados; quadros maiores são truncados. |
| **Quadros próximos podem não ser detetados** | Se um segundo quadro chegar logo após o primeiro (ex.: **CTS** imediatamente após **RTS**), o bloco pode ainda estar a copiar o primeiro. | O segundo quadro pode **não ser detetado** porque o sistema não volta a procurar um novo plateau. |
| **Suscetível à afinação de parâmetros** | A deteção depende do *threshold* e do tamanho da janela (`Nwin`), que variam com SNR, ganho de RF e multipercurso. | *Threshold* mal ajustado → **falsos positivos** ou **quadros perdidos**. |
| **Método de deteção não ótimo** | A autocorrelação é eficiente mas menos precisa que *matched filtering*. | A deteção pode falhar em SNR baixo; *matched filtering* seria mais robusto, mas tem maior custo computacional. |



