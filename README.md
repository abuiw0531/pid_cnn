*[English Version Below](#english-version)*

# 以 1D CNN 探索 PID 增益調整

[![Status](https://img.shields.io/badge/Status-Work_in_Progress_(WIP)-orange.svg)](https://github.com)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![Hardware](https://img.shields.io/badge/Hardware-Arduino_%2F_DC_Motor-brightgreen.svg)](https://arduino.cc)

## 專案介紹

我使用 Python 與 TensorFlow 建立一個結合 PID 控制波形評估和 1D CNN 的調參原型，探索如何從量測到的馬達位置響應預測下一輪 PID 增益候選值。專案目前尚未完成完整驗證；不過，已有不同迭代的 step-up／step-down 優化結果圖，可用來呈現實驗過程中的波形變化。

## 系統流程

1. 專案透過序列通訊取得目標位置及回授位置等控制資料。
2. 將時間、目標位置、量測位置與追蹤誤差整理成四通道波形，重採樣為 200 個時間點作為 CNN 輸入。
3. 1D CNN 由 Conv1D、max pooling、global average pooling 與 Dense layers 組成，輸出 Kpp、Kpd、Kvp、Kvi 四個增益候選值。
4. 程式計算 tracking、ITAE 與 overshoot 相關指標，作為控制波形評分及候選結果比較依據。
5. 當波形符合程式中的更新條件，程式將特徵與增益標籤加入 replay buffer，並以 TensorFlow GradientTape 訓練 CNN，使預測增益接近歷史標籤；其訓練 loss 為增益預測與標籤間的 MSE。
6. CNN 產生的候選增益經探索、平滑及範圍限制後供下一輪使用；程式亦包含在結果較差時回退至已保存最佳增益的邏輯。

## 優化結果圖

以下圖片呈現專案資料中的不同迭代波形。它們是原型實驗的結果視覺化，不代表已完成多條件統計驗證或證明可普遍優於傳統調參方法。

### Step-up 響應

![Step-up 響應：第 1 次迭代](docs/images/iteration_01_step_up.png)

![Step-up 響應：第 19 次迭代](docs/images/iteration_19_step_up.png)

### Step-down 響應

![Step-down 響應：第 2 次迭代](docs/images/iteration_02_step_down.png)

![Step-down 響應：第 20 次迭代](docs/images/iteration_20_step_down.png)

## 梯度與硬體執行範圍（實作現況）

在實際的實驗與調參過程中，反向傳播（Backpropagation）與梯度更新並未真正主導控制器的參數優化：
1. **硬體端無梯度計算**：微控制器僅負責執行高頻即時控制迴路、感測數據回傳與接收增益指令，硬體本身未執行任何反向傳播或梯度更新。
2. **實務調參機制**：雖然主機端程式設計了以 TensorFlow `GradientTape` 擬合歷史經驗的訓練邏輯，但實際測試中，波形的改善主要依賴波形指標評估（ITAE / Tracking Error）、自適應探索雜訊（Noise Exploration）以及表現劣化時的回退機制（Rollback）；神經網路的反向傳播在實務上並未真正起到閉迴路收斂的作用。
3. **未來展望**：目前的實體受控體（馬達 plant）非可微分模型，無法進行端到端的梯度反向傳播。未來若要真正發揮機器學習優勢，希望能探索端到端可微分控制模型、強化學習（RL）或邊緣端學習（TinyML），期待未來能成功實現真正由梯度或學習演算法主導的嵌入式自適應控制。

## 尚待完成的工作

目前專案仍屬原型，尚未完成跨硬體與負載的重複測試、統計驗證及泛化能力評估。現有結果圖呈現迭代波形，不能單獨支持模型已收斂、已具穩定性保證或在各種條件下有效等結論。

## 專案意義

這個專案讓我探索機器學習如何與控制系統的量測資料和調參流程互動，也讓我注意到模型訓練、實際控制波形評估與硬體部署是不同層次的問題。這段經驗補充了我的控制與 AI/ML 學習，但我會將它定位為尚在發展中的控制調參原型。

---

<a id="english-version"></a>

# Exploring PID Gain Tuning with 1D CNN

*[Back to Top / 繁體中文](#以-1d-cnn-探索-pid-增益調整)*

[![Status](https://img.shields.io/badge/Status-Work_in_Progress_(WIP)-orange.svg)](https://github.com)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![Hardware](https://img.shields.io/badge/Hardware-Arduino_%2F_DC_Motor-brightgreen.svg)](https://arduino.cc)

## Project Overview

I developed a parameter-tuning prototype using Python and TensorFlow that couples PID control waveform evaluation with a 1D CNN, exploring how to predict candidate PID gains for subsequent control iterations based on measured motor position responses. The project is currently an experimental prototype pending comprehensive validation; nonetheless, step-up and step-down optimization response plots across various iterations are provided to illustrate waveform transitions throughout the experiments.

## System Workflow

1. The system retrieves control telemetry including target and feedback positions via serial communication.
2. Time, target position, measured position, and tracking error are formatted into a 4-channel waveform and resampled to 200 time steps as the input to the CNN.
3. The 1D CNN architecture consists of Conv1D, max pooling, global average pooling, and Dense layers, outputting candidate values for four gains: Kpp, Kpd, Kvp, and Kvi.
4. The program evaluates metrics including tracking error, ITAE, and overshoot to score control waveforms and compare candidate parameter sets.
5. When a waveform satisfies update criteria in the code, the extracted features and gain labels are stored in a replay buffer. The CNN is trained via TensorFlow GradientTape to align predicted gains with historical target labels using Mean Squared Error (MSE) loss.
6. Candidate gains inferred by the CNN undergo exploration perturbation, exponential smoothing, and boundary clipping before the next trial; rollback logic is incorporated to revert to previously saved optimal gains if performance degrades.

## Optimization Waveform Results

The figures below demonstrate response waveforms across different iterations from experimental records. They serve as visualizations of prototype trials and do not represent exhaustive multi-condition statistical validation or proven universal superiority over conventional tuning methodologies.

### Step-up Response

![Step-up Response: Iteration 1](docs/images/iteration_01_step_up.png)

![Step-up Response: Iteration 19](docs/images/iteration_19_step_up.png)

### Step-down Response

![Step-down Response: Iteration 2](docs/images/iteration_02_step_down.png)

![Step-down Response: Iteration 20](docs/images/iteration_20_step_down.png)

## Scope of Gradients and Hardware Execution (Implementation Reality)

In actual experimental trials, backpropagation and gradient updates did not effectively drive the controller gain optimization:
1. **No Embedded Gradient Computation**: The microcontroller solely handled deterministic real-time control loops, sensor telemetry acquisition, and gain updates; it did not perform any on-device backpropagation or gradient updates.
2. **Practical Tuning Drivers**: While the host code includes a TensorFlow `GradientTape` routine designed to fit historical high-scoring parameter trajectories, actual waveform improvements were largely driven by heuristic mechanisms—namely waveform metric evaluations (ITAE, tracking error), adaptive Gaussian noise exploration, and safety rollback to best-known gains—rather than closed-loop neural network gradient updates.
3. **Future Aspirations**: The physical motor plant is not formulated as a differentiable environment, preventing end-to-end gradient propagation. Looking ahead, I hope to explore differentiable physics models, reinforcement learning (RL), or TinyML on-device adaptation, with the ultimate goal of successfully integrating true gradient-driven or learning-based adaptation into physical control systems.

## Pending Work

The project remains an early-stage prototype and has not yet undergone repetitive testing across diverse hardware setups, dynamic load conditions, statistical benchmarking, or generalizability assessments. The present result figures illustrate iterative waveform evolutions but cannot independently support claims of confirmed model convergence, guaranteed stability, or efficacy across varied operating regimes.

## Project Significance

This project provided valuable insight into how machine learning techniques can interface with empirical telemetry and tuning pipelines in control systems. It underscored the critical distinctions between offline model training, empirical control waveform evaluation, and embedded deployment. This experience enhanced my understanding of control systems and AI/ML, and I position it as an evolving prototype in data-driven control tuning.
