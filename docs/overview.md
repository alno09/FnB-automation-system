# Oyabun Automation System — Overview

## What This System Is

Oyabun is an end-to-end **automated beverage dispensing system** that combines **distributed backend services**, **Android-based POS & kiosk apps**, and **ESP32-powered hardware dispensers** into a single production-grade platform.

This repository **does not expose full source code**. It represents the **engineering design, system architecture, and selected implementation samples** of a real-world system operating across software, hardware, and infrastructure layers.

The goal of this repo is simple:

> Show how a complex, multi-domain system is **designed, structured, and scaled** — not just how to write code.

---

## Why This Repo Exists (Context for Reviewers)

This system was built to answer real production questions:

* How do you keep **POS, kiosk, backend, and hardware** in sync?
* How do you design APIs that survive **network drops and retries**?
* How do you separate responsibilities when **one bug costs real money**?

This repo documents those answers.

---

## System Scope

### Included Domains

* **Backend**: Event-driven microservices (Order, Payment, Product, Inventory, QR, Analytics)
* **Mobile**: Android POS & self-service kiosk apps (Jetpack Compose, MVVM)
* **Hardware**: ESP32-based liquid dispenser (pump, relay, QR scanner, LCD)
* **Infrastructure**: Dockerized services, message queue, relational + OLAP databases

### Explicitly Excluded

* Full production source code
* Secrets, credentials, or environment-specific configs
* Vendor-specific payment logic

---

## High-Level Flow (TL;DR)

1. User creates an order via **POS or kiosk**
2. Backend validates stock & pricing
3. Payment is processed
4. A **time-limited QR code** is generated
5. QR is scanned by the **physical dispenser**
6. Dispenser validates QR → triggers pump
7. Analytics pipeline consumes events asynchronously

Each step is **idempotent**, **auditable**, and **failure-aware**.

---

## Architectural Principles

### 1. Separation by Responsibility

Each service owns **one reason to change**.
No shared databases. No god services.

### 2. Event-First Thinking

Synchronous calls are minimized.
Critical transitions emit events to avoid tight coupling.

### 3. Hardware Is an Unreliable Client

ESP32 devices are treated as:

* occasionally offline
* slow
* retry-heavy

APIs and QR validation are designed with this assumption.

### 4. Safety > Speed

In a system that moves real liquid:

* duplicate requests are normal
* duplicate dispensing is unacceptable

---

## Repository Navigation

* `docs/architecture/` → system & service design
* `docs/api/` → API contracts & examples
* `docs/mobile/` → POS & kiosk architecture
* `docs/hardware/` → dispenser design & ESP32 behavior
* `code-samples/` → selected implementation snippets
* `media/` → diagrams, screenshots, demos

---

## Who This Is For

* Backend / Platform Engineers
* Mobile Engineers working with hardware
* System / IoT Engineers
* Technical reviewers & hiring managers

If you're evaluating **system-level thinking**, you're in the right place.

---

## Status

* System architecture: **stable**
* Core flows: **implemented & validated**
* Documentation: **actively curated**

Performance benchmarking and load testing docs are tracked separately.

---

## License

All Rights Reserved.
This repository is for **review and evaluation purposes only**.