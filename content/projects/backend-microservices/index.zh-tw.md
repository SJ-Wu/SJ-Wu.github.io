---
title: "後端微服務平台"
date: 2025-05-19
summary: "客戶管理與交易平台的 Spring Cloud 微服務 — 重構、效能與工程實踐。"
showSummary: true
tags: ["Java", "Spring Cloud", "Redis", "微服務", "DevOps"]
showReadingTime: false
---

開發與維護一套支撐客戶管理與交易系統的 **Spring Cloud** 微服務平台。

## 重點

- **模組化與解耦** — 重構舊有模組,降低服務間耦合、提高聚合。
- **效能** — 優化熱點路徑,提升併發吞吐。
- **分散式交易** — 以 Seata 維持跨服務一致性,並用 Redis 做快取與協調。
- **框架升級** — 將服務從 **Spring Boot 2.x 升級至 3.1**。
- **品質** — 將測試覆蓋率從 0% 提升至約 40%,並把測試階段導入 CI/CD。
- **維運** — 以 Grafana 儀表板監控線上狀況,快速回饋與處理問題。
