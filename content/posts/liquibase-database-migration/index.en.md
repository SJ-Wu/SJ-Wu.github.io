---
title: "From Adoption to Integration: Automating Database Version Control with Liquibase"
date: 2024-09-27
draft: false
summary: "A write-up of my JCConf 2024 talk: how to automate database migration with Liquibase and integrate it seamlessly into a Spring Boot project, without sacrificing stability or security."
tags: ["Liquibase", "Database", "Spring Boot", "DevOps"]
---

> This post is based on the talk I gave at **JCConf 2024**.

In modern software development, version control for source code is a given — but
managing changes to the database schema is often a pain point in the workflow.
This post shares how I helped a client automate **database migration** with
Liquibase and integrate it seamlessly into their project architecture.

## Why database version control?

Before diving into tooling, let's clear up two terms that are easily confused:

- **Data migration** — focuses on moving *data* between different systems,
  formats, or storage technologies.
- **Database migration** — version control for a relational database, covering
  schema updates and rollback operations.

In real projects, developers often face inconsistent versions across multiple
test environments, the risk of running SQL by hand, and security requirements
that demand strict separation of DDL/DML privileges. Adopting an automated
version-control tool is exactly what solves these stability and security
problems.

## Choosing a tool: Liquibase vs. Flyway

In the Java ecosystem, Liquibase and Flyway are the two mainstream choices. Both
support the major databases and CI/CD pipelines, but their core philosophies
differ:

| Aspect | Liquibase | Flyway |
|---|---|---|
| **Script format** | Supports XML, YAML, JSON, and SQL — flexible across databases. | Primarily plain SQL — simple and direct. |
| **Rollback** | Basic rollback supported in the community edition. | Rollback only in the enterprise edition. |
| **Learning curve** | Powerful, but configuration is relatively complex. | Convention-based naming — lightweight and easy to pick up. |
| **Advanced features** | Database *diff* to compare differences. | Better integration with Spring properties. |

Given the client's need for flexible cross-environment scripts and their emphasis
on rollback, Liquibase was often the more complete choice.

## How Liquibase works

The heart of Liquibase is the **changelog** and **changeset**:

1. **Changeset** — defines a single database change.
2. **Changelog** — organizes multiple changesets into a file (e.g.
   `changelog-master.yaml`) that serves as the entry point.
3. **Tracking table** — Liquibase creates a `DATABASECHANGELOG` table in the
   database, recording each executed ID, author, and MD5SUM checksum. This
   ensures changes are never run twice and that their contents haven't been
   tampered with.

## In practice: integrating Liquibase into a Spring Boot project

When adopting Liquibase in an existing project, our goal was to *minimize the
complexity of running it on the client side* — let the service apply changes
automatically on startup, with no extra CLI operations to learn.

### 1. Architecture and privilege separation

To meet security requirements, we leaned on the container pattern of an
**init container**:

- **Init container** — runs the `update` command using an account with DDL
  privileges.
- **Main container** — once the main service starts, it switches to an account
  with DQL-only privileges and runs `validate` to re-confirm the schema version.

### 2. Custom extensions

Spring Boot's `SpringLiquibase` only supports `update` out of the box. For more
complex scenarios, you can subclass or extend it to add support for commands like
`changelogSync` and `rollback`. We also implemented an **interceptor** that emits
SQL\*Plus-style log output before and after each change, making changes easy to
trace and audit.

## Boosting productivity: JpaBuddy

Hand-writing changesets in XML or YAML is rigorous but slow. With the
**JpaBuddy** plugin for IntelliJ IDEA, you can generate Liquibase scripts
directly from your JPA entities. This greatly reduces the developer's burden and
the chance of manual mistakes.

## Closing thoughts

Automating database version control isn't just about installing a tool — it's a
shift in the development process itself. From requirements gathering and
architecture design through to implementation, Liquibase offers a high degree of
flexibility together with strong safety guarantees. I hope this experience helps
you handle database versioning more gracefully in your own projects.

---

### Resources

- **GitHub repo**: [JCConf 2024 Liquibase Demo](https://github.com/SJ-Wu/JCConf2024)
- **Common commands**: `update`, `rollback`, `diff`, `changelogSync`
