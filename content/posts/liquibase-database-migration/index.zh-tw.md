---
title: "從導入到整合：使用 Liquibase 實現自動化資料庫版本控制的心路歷程"
date: 2024-09-27
draft: false
summary: "JCConf 2024 分享整理：如何用 Liquibase 將資料庫遷移自動化，並無縫整合進 Spring Boot 專案，兼顧穩定性與資安。"
tags: ["Liquibase", "資料庫", "Spring Boot", "DevOps"]
---

> 本文整理自我在 **JCConf 2024** 的分享。

在現代軟體開發中，程式碼的版本控制已經是標配，但資料庫架構（Schema）的變動管理卻往往是開發流程中的痛點。本文將分享如何透過 Liquibase 這款強大的工具，協助客戶將資料庫遷移（Database Migration）自動化，並無縫整合進專案架構中。

## 為什麼需要資料庫版本控制？

在深入工具之前，我們必須先釐清兩個容易混淆的概念：

- **資料遷移（Data Migration）**：側重於資料在不同系統、格式或儲存技術間的轉移。
- **資料庫遷移（Database Migration）**：則是對關聯式資料庫進行版本控制，包含 Schema 的更新或 Rollback 操作。

在實際專案中，開發者常面臨多個測試環境版本不一、人為執行 SQL 風險高，以及資安考量下 DDL/DML 權限需嚴格區隔等挑戰。導入自動化版控工具，正是為了解決這些穩定性與安全性的問題。

## 工具選型：Liquibase vs. Flyway

在 Java 生態系中，Liquibase 與 Flyway 是兩大主流選擇。它們都支援主流資料庫與 CI/CD 流程，但核心理念有所不同：

| 特性 | Liquibase | Flyway |
|---|---|---|
| **腳本格式** | 支援 XML、YAML、JSON、SQL，具跨資料庫靈活性。 | 以純 SQL 為主，簡單直覺。 |
| **Rollback** | 社群版即支援基礎 Rollback。 | 僅企業版支援 Rollback。 |
| **學習曲線** | 功能強大但配置相對複雜。 | 基於慣例命名，輕量且易上手。 |
| **進階功能** | 具備資料庫 Diff 功能，可比對差異。 | 整合 Spring Properties 較佳。 |

考量到客戶對跨環境腳本的靈活性需求，以及對 Rollback 功能的重視，Liquibase 往往是更全面的選擇。

## Liquibase 核心運作機制

Liquibase 的核心在於 **Changelog** 與 **Changeset**：

1. **Changeset**：定義每一次的資料庫變動。
2. **Changelog**：將多個 Changeset 組織成檔案（如 `changelog-master.yaml`），作為執行入口。
3. **Tracking Table**：Liquibase 會在資料庫建立 `DATABASECHANGELOG` 表，記錄已執行的 ID、作者與 MD5SUM 校驗碼，確保變更不會重複執行且內容未經非法竄改。

## 實戰分享：將 Liquibase 整合進 Spring Boot 專案

在導入現有專案時，我們的目標是「簡化客戶端施行複雜度」，讓服務啟動時自動完成變更，而不需要額外學習 CLI 操作。

### 1. 架構設計與權限隔離

為了符合資安規範，我們利用容器技術中的 **InitContainer**：

- **InitContainer**：使用具備 DDL 權限的帳號執行 `update` 指令。
- **Main Container**：主服務啟動後改用僅具 DQL 權限的帳號，並執行 `validate` 再次確認版本正確性。

### 2. 客製化擴充功能

Spring Boot 提供的 `SpringLiquibase` 預設僅支援 `update` 功能。為了應付更複雜的情境，我們可以透過繼承或擴充，加入 `changelogSync` 與 `rollback` 等指令支援。同時，實作 **Interceptor（攔截器）** 機制，可以在變動前後輸出類似 SQL Plus 的 LOG 紀錄，方便追蹤與審核。

## 提升開發效率：JpaBuddy 的妙用

手寫 XML 或 YAML 的 Changeset 雖然嚴謹，但效率較低。透過 IntelliJ IDEA 的 **JpaBuddy** 插件，開發者可以直接根據撰寫好的 JPA Entity 自動生成 Liquibase 腳本。這大幅降低了開發者的負擔，也減少了人為手誤的機會。

## 結語

自動化資料庫版控不只是安裝一個工具，更是一場開發流程的革新。從需求訪談、架構設計到最後的實作，Liquibase 提供了極高的靈活性與安全性保障。希望這份導入心得能幫助你在未來的專案中，更優雅地處理資料庫的版本難題。

---

### 補充資訊與資源

- **GitHub Repo**：[JCConf 2024 Liquibase Demo](https://github.com/SJ-Wu/JCConf2024)
- **常用指令**：`update`、`rollback`、`diff`、`changelogSync`
