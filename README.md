# Platform Engineering Systems

## Overview

This repository contains system-level implementations demonstrating how application artifacts are deployed, operated, and managed across modern infrastructure.

The focus is not on application development, but on designing and operating systems that behave reliably in production-like environments.

---

## Getting Started

This repository relies on Git submodules for application source code.

Clone the repository with submodules:

```bash
git clone --recurse-submodules https://github.com/ibtisam-iq/platform-engineering-systems.git
cd platform-engineering-systems
```

If you already cloned without submodules, run:

```bash
git submodule update --init --recursive
```

---

## What This Repository Represents

Each system in this repository takes a built application artifact and transforms it into a running, observable, and scalable system.

This includes:

- Containerization
- Infrastructure provisioning
- Orchestration
- Monitoring and logging
- Scaling strategies
- Failure handling and recovery

---

## Repository Structure

```text
platform-engineering-systems/
│
├── systems/
│   ├── java-monolith/
│   ├── node-monolith/
│   └── python-monolith/
│
├── shared/
├── docs/
└── README.md
```

---

## Core Principle

Each system is **self-contained**.

This means:

- Each system has its own deployment configuration
- Each system defines its own infrastructure
- Each system can run independently

This avoids tight coupling and reflects real-world system ownership.

---

## What a “System” Includes

A system is not just deployment.

It includes:

- Runtime (Docker / Kubernetes)
- Infrastructure (Terraform)
- Observability (metrics, logs)
- Scaling (HPA, strategies)
- Recovery (failure handling)

---

## Systems Included

### Java Monolith

- Containerized Spring Boot application
- Kubernetes-based deployment
- Infrastructure provisioning via Terraform
- Monitoring and logging setup

(More systems will be added over time)

---

## Relationship with Pipelines

This repository assumes that application artifacts are already built.

For CI/CD pipelines:

👉 See: https://github.com/ibtisam-iq/devsecops-pipelines

---

## Engineering Focus

This repository demonstrates:

- Platform engineering thinking
- System design beyond deployment
- Understanding of runtime behavior
- Real-world infrastructure patterns

---

## Author

Muhammad Ibtisam Iqbal

DevOps Engineer | Cloud Infrastructure | Kubernetes (CKA, CKAD)
