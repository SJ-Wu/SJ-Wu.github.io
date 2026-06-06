---
title: "KYC Liveness & Deepfake Detection"
date: 2026-03-01
summary: "A production computer-vision service that decides whether a face-verification submission is a real, live person or a spoof / deepfake."
showSummary: true
tags: ["Computer Vision", "Machine Learning", "PyTorch", "ONNX", "FastAPI"]
showReadingTime: false
---

A production **identity-verification** service that decides whether a
face-verification submission comes from a real, live person — or from a spoof or
**deepfake**.

## What it does

- A multi-stage decision pipeline: **quality gate → liveness gate → model
  consensus**, returning an `accept / review / reject` decision with reasons.
- Combines **physical liveness cues** (multi-frame RGB light-reaction checks)
  with an **ensemble of deep models** for deepfake detection.
- Served as an HTTP API for synchronous verification.

## Tech

- **Models** — convolutional networks (EfficientNet / Xception / CLIP-based) plus
  a gradient-boosting model, combined by consensus.
- **Stack** — Python, PyTorch, ONNX, FastAPI; end-to-end **train → evaluate →
  deploy** pipeline with automated retraining and reporting.
- **Results** — multi-model AUC ≥ 0.99 on held-out data; tuned to run within KYC
  latency budgets on CPU.
