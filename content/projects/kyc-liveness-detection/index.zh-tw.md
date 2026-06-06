---
title: "KYC 活體與深偽偵測"
date: 2026-03-01
summary: "一套生產環境的電腦視覺服務，判斷人臉驗證的影像來自真實活人，還是攻擊樣本／深偽。"
showSummary: true
tags: ["電腦視覺", "機器學習", "PyTorch", "ONNX", "FastAPI"]
showReadingTime: false
---

一套生產環境的**身分驗證**服務,判斷人臉驗證提交的影像來自真實活人,還是攻擊樣本或
**深偽(deepfake)**。

## 做了什麼

- 多段式判決流程:**品質閘門 → 活體閘門 → 模型共識**,回傳
  `accept / review / reject` 並附上原因。
- 結合**物理活體訊號**(多張影像的 RGB 光反應檢查)與**多模型深度學習集成**
  進行深偽偵測。
- 以 HTTP API 形式提供同步驗證。

## 技術

- **模型** — 卷積網路(EfficientNet / Xception / CLIP 系列)搭配梯度提升模型,
  以共識方式整合。
- **技術棧** — Python、PyTorch、ONNX、FastAPI;完整的**訓練 → 評估 → 部署**
  pipeline,含自動重訓與報告產出。
- **成果** — 多模型在保留測試集 AUC ≥ 0.99;並調校至可在 CPU 上符合 KYC 延遲需求。
