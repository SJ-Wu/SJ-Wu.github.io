---
title: "Backend Microservices Platform"
date: 2025-05-19
summary: "Spring Cloud microservices for a customer-management and transaction platform — refactoring, performance, and engineering practices."
showSummary: true
tags: ["Java", "Spring Cloud", "Redis", "Microservices", "DevOps"]
showReadingTime: false
---

Building and maintaining a **Spring Cloud** microservices platform behind a
customer-management and transaction system.

## Highlights

- **Modularization & decoupling** — refactored legacy modules to lower coupling
  and raise cohesion across services.
- **Performance** — optimized hot paths and improved concurrent throughput.
- **Distributed transactions** — Seata-based consistency across services, with
  Redis for caching/coordination.
- **Framework upgrade** — migrated services from **Spring Boot 2.x to 3.1**.
- **Quality** — raised test coverage from 0% → ~40% and wired the test stage
  into CI/CD.
- **Operations** — Grafana dashboards for live monitoring and fast incident
  feedback.
