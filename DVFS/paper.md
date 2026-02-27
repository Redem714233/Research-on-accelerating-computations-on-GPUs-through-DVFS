# Improving GPU Energy Efficiency through an Application-transparent Frequency Scaling Policy with Performance Assurance

Authors: Yijia Zhang, Qiang Wang, Zhe Lin, Pengxiang Xu, Bingqiang Wang

Conference: EuroSys '24, April 22–25, 2024, Athens, Greece

DOI: https://doi.org/10.1145/3627703.3629584

------

## Abstract

Power consumption is one of the top limiting factors in high-performance computing systems and data centers, and dynamic voltage and frequency scaling (DVFS) is an important mechanism to control power1. Existing works using DVFS to improve GPU energy efficiency suffer from the limitation that their policies either impact performance too much or require offline application profiling or code modification, which severely limits their applicability on large clusters2.



To address this issue, we propose a novel GPU DVFS policy, **GEEPAFS**, which improves the energy efficiency of GPUs while providing performance assurance3. GEEPAFS is application-transparent as it does not require any offline profiling or code modification on user applications4. To achieve this, GEEPAFS models application performance online based on our quantitative analysis of a correlation between performance and GPU memory bandwidth utilization5. Based on their relationship, GEEPAFS builds a fold-line frequency-performance model for applications being executed, and it applies the model to guide the setting of GPU frequency to maximize energy efficiency while ensuring the performance loss is bounded6.



Through experiments on NVIDIA V100 and A100 GPUs, we show that GEEPAFS is able to improve the energy efficiency by 26.7% and 20.2% on average7. While achieving this improvement, the average performance loss is only 5.8%, and the worst-case performance loss is 12.5% among all 33 tested applications8.



------

## 1 Introduction

The rising power consumption has become one of the top limiting factors in high-performance computing (HPC) systems and data centers9. The current Top-1 HPC system, Frontier, consumes a peak power of 22.7 MW10. It is also estimated that all data centers in the world consume 1% of global energy production11. To reduce the electricity bill and environmental impacts, it is critical to improve the energy efficiency of large computing systems12.



Since the rapid growth of artificial intelligence, graphics processing units (GPU) are playing an important role in computing clusters, but recent generations of GPUs are giant power consumers, as the thermal design power (TDP) of NVIDIA GPUs goes from 300 W on GPU V100, 400 W on A100, and up to 700 W on H10013. In high-end GPU servers, the total power of all GPUs usually surpasses the total power of the remaining part14. As a result, improving GPU's energy efficiency is of central importance15.



Dynamic voltage-frequency scaling (DVFS) is an approach to regulate the power consumption of a processor by increasing/decreasing its frequency $f$ and voltages $V$. DVFS is a key control knob in the power management of computing systems because of the following reasons:

1. DVFS can modulate power in wide ranges since the dynamic power of a processor is directly proportional to $V^{2}f$161616.

   

   

   

2. DVFS in a proper range only affects the execution time of applications, and it does not affect their results (except for the undervoltage scenario)17.

   

   

3. The response time of DVFS is small, usually within 100 milliseconds or less18.

   

   

4. Finally, DVFS is generally considered mature on modern processors and it has been widely applied in many systems from personal computers to smartphones19.

   

   

Given these advantages of DVFS, many works have experimented with GPU DVFS, and they find that tuning voltage and frequency properly improves energy efficiency20.





*(Figure 1 description: The impact of GPU (V100) frequency on performance, power, and energy efficiency. It shows that max-efficiency frequency does not align with max frequency, offering opportunities for optimization)* 21.



However, as far as we know, production systems rarely adopt these policies, and the common concern is performance loss and application-intrusiveness22. Our experiments confirm that maximizing energy efficiency leads to significant performance loss (up to 34% on V100)23. To design a frequency scaling policy practical for production systems, it is important to offer performance assurance and design the policy in an application-transparent way24.



To address this challenge, we propose **GEEPAFS**: a GPU Energy-Efficient and Performance-Assured Frequency Scaling policy25.



- 

  **Performance Assurance:** The performance loss is bounded by a preset percentage (e.g., 10%)26.

  

  

- 

  **Application-Transparent:** Does not require offline profiling or code modification27.

  

  

- 

  **Method:** Models performance online using GPU hardware counters, specifically the correlation between GPU memory bandwidth utilization and application performance28.

  

  

**Contributions:**

1. We propose GEEPAFS, which improves energy efficiency while ensuring meeting a performance target29.

   

   

2. We build a fold-line model to quantify the frequency-performance relation online based on GPU memory bandwidth utilization30.

   

   

3. Experiments on NVIDIA V100 and A100 GPUs show GEEPAFS improves energy efficiency by 26.7% and 20.2% on average, with bounded performance loss31.

   

   

4. We show that the accuracy of performance modeling is bottlenecked by the latency of frequency tuning and the resolution of hardware counters32.

   

   

------

## 2 Related Works

The impact of GPU frequency on power and performance has been analyzed in many works33. However, most previous works require offline application profiling or code modification34.





**Table 1: Comparison with GPU DVFS policies proposed in related works.** 35



| **Related Works**                | **Perf. Awareness** | **No Code Change** | **No Offline Profiling** | **No Workload Limit** | **No Offline Training** | **Scale (GPUs)** | **Platform** |
| -------------------------------- | ------------------- | ------------------ | ------------------------ | --------------------- | ----------------------- | ---------------- | ------------ |
| Ma, 2012 [18]; Komoda, 2013 [17] | ✓                   | ✓                  |                          |                       |                         | 1                | NVIDIA       |
| Paul, 2013 [38]                  |                     |                    |                          |                       |                         | 1                | AMD          |
| Abe, 2014 [1]                    |                     |                    |                          |                       |                         | 1x4              | NVIDIA       |
| Paul, 2015 [37]                  |                     |                    |                          |                       |                         | 1                | AMD          |
| Guerreiro, 2015 [11]             |                     |                    |                          |                       |                         | 1x4              | NVIDIA       |
| Majumdar, 2017 [19]              |                     |                    |                          |                       |                         | 1                | AMD          |
| Fan, 2019 [8]                    |                     |                    |                          |                       |                         | 1x2              | NVIDIA       |
| Guerreiro, 2019 [12]             |                     |                    |                          | ✓                     |                         | 1x5              | NVIDIA       |
| Ilager, 2020 [14]                | ✓                   |                    |                          |                       |                         | 1                | NVIDIA       |
| Zou, 2020 [52]                   | ✓                   |                    |                          |                       |                         | 1                | NVIDIA       |
| Nabavinejad, 2022 [26]           | ✓                   |                    |                          |                       |                         | 1                | NVIDIA       |
| Wang, 2022 [47]                  |                     |                    |                          |                       |                         | 1                | NVIDIA       |
| Ali, 2022 [2]                    |                     |                    |                          |                       |                         | 1                | NVIDIA       |
| **Ours (GEEPAFS)**               | **✓**               | **✓**              | **✓**                    | **✓**                 | **✓**                   | **8 / 4 / 1**    | **NVIDIA**   |

Some works applied reinforcement learning (RL) on mobile devices, but these are hard to apply to large clusters due to training costs 36. Others used simulators, which rely on counters not available online37. None evaluated on multi-GPU parallel applications38.



------

## 3 Design Principles

Our goal is to design an application-transparent GPU frequency scaling policy applicable on production systems39.



Design Principle 1: The policy will capture workload characteristics and build frequency-performance models online.

There is not a unique freq-perf relation; it varies significantly from one application to another (e.g., GeMM is sensitive, CNN_1.5M is not) 40.

Design Principle 2: The policy cannot modify user applications' source codes and cannot assume the existence of application execution progress report.

It is impractical to ask all users to add formatted progress reports or rely on tools like CUPTI that require code modification 41.

Design Principle 3: The frequency scaling policy cannot rely on offline application profiling.

Profiling requires running each application at least once before execution, which is impractical for many systems42424242.

Design Principle 4: The frequency scaling policy cannot require repetitively running an application.

Machine hours are valuable; repetitively running applications is not acceptable 43.

Design Principle 5: The policy will capture an application's runtime behavior instead of simply relying on its category.

Simply changing parameters (e.g., batch size or model size) can dramatically change the freq-perf relation (as shown in Figure 2 with CNN/ViT sizes) 44.

Design Principle 6: The frequency scaling policy will rely on models with explainability.

Lack of explainability in ML models raises concerns for low-level system controls 45.

------

## 4 The GEEPAFS Policy

### 4.1 Overview of GEEPAFS

The policy executes Steps 1-5 iteratively on each GPU46.



- 

  **Step 1 & 2 (Probing Phase):** Tune frequency and sample hardware counters (GPU utilization, GPU memory bandwidth utilization, power)47.

  

  

- 

  **Step 3 (Online Modeling):** Build freq-perf-power model based on sampled data48.

  

  

- 

  **Step 4 (Obtain Optimal Freq):** Search for frequency that improves energy efficiency while ensuring performance loss is bounded by $\delta$ ($5\% \le \delta \le 30\%$)49494949.

  

  

  

- **Step 5:** Tune GPU to the optimal frequency.



*(Figure 3: Overview of GEEPAFS policy)*50.



### 4.2 Hardware counter sampling

We use **NVML** to read hardware counters. It is accessible during job execution and requires no code modification 51.



### 4.3 The GPUbwUt-performance correlation

We discover that GPU memory bandwidth utilization (GPUbwUt) is the key to performance modeling52.

Experiments show that GPUbwUt is directly proportional to application performance under frequency scaling:



$$Q(f) / Q(f_{max}) = U(f) / U(f_{max})$$

where $Q$ is performance and $U$ is GPUbwUt53. This allows us to use $U(f)/U(f_{max})$ as an estimation of normalized performance54. Other metrics like GPU utilization or Power do not show such simple relations55.



### 4.4 Fold-line model of frequency-GPUbwUt relation

We use fold-line models to fit the freq-GPUbwUt relations56. We assume a critical frequency $f_c$ separates two segments (compute-bound vs. memory-bound)57575757:



$$U(f) = \begin{cases} a_1 f + b_1, & f < f_c \\ a_2 f + b_2, & f \ge f_c \end{cases}$$

(Eq. 1) 58



We sample GPUbwUt at three or more frequencies and use the least squares method to fit this model59.



### 4.5 Algorithms of the GEEPAFS policy



Algorithm 1: GEEPAFS Policy 60

Input: performance constraint $\delta$, intervals $T_1, T_2$, probing list $F$, samples $M$

1. **while** True **do**
2. $t \leftarrow$ current time;
3. **if** isProbingPhase(t) **then**
4. $G, U, P \leftarrow$ Collect GPU util, GPUbwUt, Power;
5. SetNextProbingFreq();
6. wait($T_2$);
7. **else if** isEndOfProbingPhase(t) **then**
8. $Model \leftarrow$ FitFreqPerfModel($U, F$);
9. $f_{bound} \leftarrow$ CalcPerfAssuredFreq($Model, \delta$);
10. $f_{eff} \leftarrow$ GetEnergyEfficientFreq($Model, P$);
11. $f_{cap} \leftarrow$ CalcGpuFreqCap($G, F, \delta$); // Eq. (2)
12. $f_{opt} \leftarrow \min(\max(f_{bound}, f_{eff}), f_{cap})$;
13. SetGpuFreq($f_{opt}$);
14. wait($T_1 - M \cdot T_2$);
15. **end**
16. **end**

The frequency cap is calculated as:



$$f_{cap} = \max_i \frac{(1-\delta) \cdot f_i}{G(f_i)}$$



(Eq. 2) 61. This bounds frequency when GPU utilization is low62.

### 4.6 Tuning frequency with performance assurance

Algorithm 2 calculates $f_{bound}$. It finds the minimal frequency where predicted performance is $\ge (1-\delta)$ of max performance63.





**Algorithm 2 Logic:** 64



- If Linear Model ($U(f) = af+b$):
  - Solve $a \cdot f_{bound} + b = (1-\delta)(a \cdot f_{max} + b)$ (Eq. 3).
- If Fold-line Model:
  - Solve $a_i \cdot f_{bound} + b_i = (1-\delta)U_{max}$ (Eq. 4/5).

### 4.7 Discussion on parameter selection

Parameters should be chosen based on GPU type.

- Probing range should cover max frequency and max-efficiency frequency65.

  

  

- We use $M=8$ samples, $T_2=200$ ms, $T_1=15$ s. Probing overhead is $M \cdot T_2 / T_1 \approx 10.7\%$66.

  

  

- Suggested $\delta$ is 5%–30%67.

  

  

------

## 5 Experimental Methodology



**Table 2: GPU servers in our experiments.** 68



| **GPU Specifications** | **DGX station** | **DGX-1 MaxQ** | **A100**        |
| ---------------------- | --------------- | -------------- | --------------- |
| **Number of GPUs**     | 4               | 8              | 1               |
| **GPU Type**           | V100            | V100-MaxQ      | A100            |
| **Max Power**          | 300 W           | 163 W          | 400 W           |
| **Base / Max Freq**    | 1297 / 1530 MHz | 817 / 1440 MHz | 1155 / 1410 MHz |
| **Memory Bandwidth**   | 898 GB/s        | 829.44 GB/s    | 2039 GB/s       |



**Workloads:** 33 workloads covering CUDA Samples, PolyBench, Rodinia, PyTorch (CNN, ViT, ResNet), etc.696969.





**Table 3: List of workloads (abbreviated).** 70



- *CUDA Samples:* BlackScholes, BandwidthTest, FFT, GeMM, etc.
- *PolyBench:* 3MM, BICG, FDTD-2D.
- *Rodinia:* CFD, SRAD.
- *DL:* ResNet50/101/152, ViT_base/small/tiny, CNN_1.5M/1.8M/2.4M.



**Baseline Policies:** 71



1. **MaxPerf:** Always max frequency.
2. **EfficientFix:** Fixed max-efficiency frequency.
3. **UtilizScale:** Linearly proportional to GPU utilization.
4. **NVboost:** Default NVIDIA policy.



**Metrics:** Performance (work/time), Power, Energy Efficiency (perf/power), ED2P 72.



------

## 6 Results and Discussions

### 6.1 Compare frequency scaling policies

On **V100 GPUs (DGX Station)**:

- 

  **GEEPAFS ($\delta=10\%$):** Improves energy efficiency by **26.7%** on average73. Performance loss is **5.8%** (avg) and **12.5%** (worst-case)74747474.

  

  

  

  

- 

  **EfficientFix:** Improves efficiency by 43.9% but suffers 13.8% avg performance loss (worst case 34.7%) 75.

  

  

- 

  **NVboost:** Improves efficiency by only 0.4%76.

  

  

- 

  **UtilizScale:** Improves efficiency by 11.5%77.

  

  



*(Figures 8 & 9 show these comparisons)* 78.



### 6.2 Compare EDP, ED2P, and Oracle

GEEPAFS achieves the lowest ED2P among baselines, lower than UtilizScale by 3.5% and EfficientFix by 8.9%79.

Compared to an Oracle policy (ideal), GEEPAFS is close (Oracle achieves 5% better energy)80.

### 6.3 Compare different machines and thresholds

- 

  **DGX-1 MaxQ:** GEEPAFS improves energy efficiency by 18.8%81.

  

  

- 

  **A100:** GEEPAFS improves energy efficiency by 20.2%82.

  

  

- Worst-case performance loss is bounded around 10% for both.

### 6.4 Demands for faster and more accurate GPU sampling/tuning tools

Accuracy is bottlenecked by the 1% resolution of GPUbwUt in NVML83. This creates artifacts (zigzag curves) in modeling84. Improving counter accuracy to 0.1% would help85.

Latency of frequency tuning (~25ms) limits the probing interval $T_2$86.

### 6.5 Overhead and limitations of GEEPAFS

Overhead comes from the probing phase. Reducing $T_1$ increases overhead (from 1.5% to 4.5% loss on GeMM)87. GEEPAFS may not work well for very short-lived applications (<1s) due to probing latency88.



------

## 7 Conclusion

We propose an application-transparent GPU frequency scaling policy, **GEEPAFS**. It models performance online using the correlation between performance and GPU memory bandwidth utilization. Experiments show it improves energy efficiency by 26.7% (V100) and 20.2% (A100) with bounded performance loss 89.



------

## A Artifact Appendix

### A.3 Set-up

Source codes are available at `https://github.com/zyjopensource/geepafs`.

1. Edit `dvfs.c` to select `MACHINE`.
2. `make`.
3. Run: `sudo ./dvfs mod Assure p90`.

### A.4 Evaluation workflow

- **Experiment (E1):** Compare GEEPAFS and MaxFreq.
- **Experiment (E2):** Evaluate baselines (EfficientFix, NVboost, UtilizScale).
- **Experiment (E3):** Evaluate on DGX-1 MaxQ and A100.
- **Experiment (E4):** Compare different $\delta$.

------

## References

[1] Abe et al., 2014. Power and Performance Characterization...

[2] Ali et al., 2022. Optimal GPU Frequency Selection...

...

[48] Wang and Chu, 2020. GPGPU Performance Estimation...

[52] Zou et al., 2020. Indicator Directed Dynamic Power Management...



(Note: Full references [1-52] are available in the original PDF source 90)