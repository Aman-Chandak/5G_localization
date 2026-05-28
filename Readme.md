# 5G-Based UAV Localization in GNSS-Denied Indoor Environments

This repository accompanies my M.S. thesis at Arizona State University on indoor UAV localization using **5G New Radio Positioning Reference Signals (PRS)** as a substitute for unavailable GNSS. The work was sponsored by **Honeywell Aerospace Technologies** as the industry partner.

The thesis is currently **under embargo by Honeywell**, so the full implementation is not yet published here. This README provides a comprehensive overview of the problem, the system, the methods, and the headline results.

> **Citation.** A. Chandak, Y. Kumar, N. Rao, D. Muirhead, W. Zhang. *"5G-based Localization for Unmanned Aerial Vehicles."* AIAA SciTech Forum, 2026. (https://doi.org/10.2514/6.2026-0501)

---

## 1. Problem

UAVs increasingly need to fly in indoor facilities, warehouses, urban canyons, and disaster sites — exactly the environments where GNSS is blocked or heavily distorted. Vision and LiDAR pipelines exist but degrade in featureless corridors, glass atriums, and SWaP-limited platforms; ultra-wideband anchors give cm-level accuracy but require purpose-built infrastructure at every site.

5G New Radio offers a compelling middle ground: it reuses the cellular infrastructure that is already being deployed for connectivity, and its PRS waveforms have far wider bandwidth than legacy cellular positioning. The catch is **multipath**. The earliest detectable arrival at the receiver can be delayed by reflections and non-line-of-sight (NLOS) propagation, and a 10 ns excess delay translates into ~3 m of range error before OTDOA equations even get involved.

<p align="center">
  <img src="docs/nlos_concept.png" alt="Urban-canyon NLOS propagation" width="60%">
  <br>
  <!-- <em>Figure 1 — Multipath / NLOS is the dominant source of practical ranging error in indoor and dense-urban deployments.</em> -->
</p>

The thesis asks: *How can 5G PRS-based OTDOA localization be made accurate and reliable enough for UAV operation in GNSS-denied indoor environments when the measured arrival times are corrupted by geometry, multipath, and detector-dependent timing bias?*

## 2. Approach

The work is split into two complementary parts:

1. **An SDR hardware testbed** that demonstrates end-to-end indoor 5G PRS localization and exposes the dominant residual error sources.(https://doi.org/10.2514/6.2026-0501)
2. **A detector-aware probabilistic correction model** trained in a hardware-matched simulator that fixes per-link bias and feeds calibrated uncertainty into the OTDOA solver.

### 2.1 Hardware testbed

The testbed emulates the standardized 5G positioning split: synchronized ground transmitters broadcast PRS, a UAV-mounted receiver records I/Q samples, and OTDOA multilateration runs offline. Built around four Ettus USRP B210 software-defined radios driven by **OpenAirInterface**, with a shared 10 MHz + 1 PPS reference from an OctoClock-G distribution module.

<p align="center">
  <img src="docs/testbed_drone_studio.png" alt="Indoor drone studio with three gNB transmitters and UAV receiver" width="80%">
  <br>
  <em>Indoor drone studio at ASU (≈12 × 12 m) with three synchronized gNB transmitters (USRP B210) and the UAV-mounted receiver.</em>
</p>

| Hardware / signal parameter | Value |
|---|---|
| SDR platform | Ettus USRP B210 |
| Number of transmitters | 3 |
| Center frequency | 3.5 GHz |
| Subcarrier spacing | 30 kHz (μ = 1) |
| Resource blocks | 106 |
| Effective signal bandwidth | 38.16 MHz |
| Receiver sample rate | 46 MHz |
| Delay resolution | ≈ 26 ns (≈ 7.8 m) |
| Synchronization | OctoClock-G (10 MHz + 1 PPS) |

The signal chain is: PRS generation (3GPP TS 38.211 via OpenAirInterface) → over-the-air propagation → I/Q capture → correlation-based ToA extraction with a top-*K* first-arriving-path consistency check → inter-gNB timing calibration on a 10-point survey circle → weighted Gauss–Newton OTDOA solve.

**Three hardware experiments** were run on this testbed:

| Experiment | Setup | RMSE |
|---|---|---|
| Exp. 1 — Single-point baseline | Receiver at the geometric origin (best GDOP) | **1.50 m** |
| Exp. 2 — Eight-point static evaluation | Points distributed across the studio, including poor GDOP near the transmitter plane | **4.87 m** |
| Exp. 3 — Drone-mounted evaluation | UAV at ~1.5 m, off-center | **5.80 m** |

The hardware results validate that the synchronized PRS chain works end-to-end and produces meter-level accuracy at the best operating point. They also expose the bottleneck: **residual timing bias that survives even after careful calibration and first-arriving-path filtering** is the dominant remaining error source — not random noise — and it grows in geometrically challenging regions.

### 2.2 Hardware-matched simulator + learning pipeline

Because the residual bias is structured rather than random, ordinary least-squares solvers can't absorb it. The second half of the thesis builds a learning model targeted exactly at that bias.

**Simulator.** A MATLAB ray-tracing pipeline reproduces the PRS operating point, leading-edge timing extraction, bounded power-delay-profile observations, distance-dependent SNR, and random-walk receiver clock bias seen on the real hardware. This produces a training corpus of:

- 180 simulated realizations
- 540,000 timesteps
- **2,157,161 valid PRS links** spanning small, base, and large indoor room regimes

Crucially, the simulator exports **detector-facing measurements** — the same view as the OpenAirInterface receiver — so the learned model trains on the same signal statistics it sees on hardware.

**Learning approach.** A compact probabilistic correction model is trained on the simulated corpus to predict, for each individual radio link, both a bias correction *and* a calibrated uncertainty estimate from local, detector-derived signal features. The model is intentionally **environment-agnostic** — it never sees room identity — so it generalizes to previously unseen rooms rather than memorizing trajectory geometry.

**Integration.** Predicted corrections and variances are fed into a **weighted Gauss–Newton OTDOA solver** as per-link bias terms and inverse-variance weights. A severity-aware reference-transmitter selection rule picks the most trustworthy anchor at each timestep, which materially improves performance in geometrically challenging regions where the raw testbed previously struggled.

## 3. Headline results

Evaluation is performed on **held-out environments** — rooms the corrector never saw during training — so the numbers test genuine generalization rather than overfitting to a specific layout.

| Metric | Raw OTDOA baseline | Corrected pipeline |
|---|---|---|
| Validation horizontal RMSE | **8.493 m** | **1.267 m** |
| Held-out test horizontal RMSE | **11.827 m** | **2.080 m** |
| Improvement vs. baseline | — | **~82% error reduction** |
| Parameters vs. strongest attention-based ablation | — | **33.9% fewer**, within 0.036 m on test RMSE |

**What the numbers mean.**

- The corrector reduces held-out test RMSE from **11.83 m → 2.08 m**, closing most of the gap between raw indoor measurements and the **bandwidth-limited oracle bound** — the theoretical lower bound achievable at the same detector setting and PRS bandwidth.
- The improvement is **universal**: in the per-trajectory analysis every held-out test trajectory sees a reduction in error after correction, not just a favourable subset.
- A room-regime breakdown shows the largest absolute gain in **large rooms**, which are by far the hardest in the raw OTDOA setting yet the learned corrector still collapses mean error into the low-single-meter range.
- The proposed compact model **matches the strongest attention-based ablation** in point-correction accuracy while using **33.9% fewer trainable parameters**, an advantage when the corrector is eventually moved onto edge compute.

**Remaining limitation.** Point correction generalizes cleanly to unseen rooms, but the predicted uncertainty becomes slightly under-calibrated under distribution shift — i.e., the corrector is occasionally over-confident on environment families it did not see during training. This is the principal unresolved weakness identified in the thesis and the main direction for future work.

## 4. Contributions

1. A validated indoor 5G PRS UAV localization testbed built on synchronized Ettus USRP B210s and OpenAirInterface, with a complete OTDOA pipeline (correlation → top-*K* FAP consistency → inter-gNB calibration → weighted Gauss–Newton).
2. A hardware-matched ray-tracing simulator that preserves the detector viewpoint of the real receiver, producing a 2.15 M-link environment-agnostic training corpus.
3. A detector-aware probabilistic correction framework that predicts both a per-link bias correction and a calibrated uncertainty estimate from local, environment-agnostic signal features.
4. End-to-end integration of the learned predictions into a weighted Gauss–Newton OTDOA solver, with severity-aware reference-transmitter selection.
5. A comparative ablation showing that the proposed compact model matches the strongest attention-based alternative while using 33.9% fewer trainable parameters.

## 5. Acknowledgments

- **Dr. Wenlong Zhang** (committee chair) for guidance throughout this project.
- **Honeywell Aerospace Technologies** for funding, hardware testbed support, and continuous technical engagement.

## Author

**Aman Chandak** — M.S. Robotics and Autonomous Systems, Arizona State University.
[Portfolio](https://aman-chandak.github.io/portfolio/) · [LinkedIn](https://www.linkedin.com/in/aman-chandak-9094b5131/) · achan160@asu.edu
