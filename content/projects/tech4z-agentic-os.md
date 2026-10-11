---
title: "TECH4Z Agentic OS — Engineering a Governed Multi-Agent AI Platform"
summary: "How I designed, built, and operate a multi-agent AI platform with GPU infrastructure, LLM orchestration, governed execution, and automated delivery — from bare metal up"
date: 2026-10-08
tags: ["agentic-ai", "gpu-infrastructure", "proxmox", "automation", "llm-orchestration", "governed-ai", "homelab"]
weight: 1
ShowToc: true
TocOpen: true
---

## Overview

TECH4Z Agentic OS is a multi-agent AI platform I designed, built, and operate as a solo engineer. The platform orchestrates GPU compute, LLM inference, image generation, and workflow automation across a multi-node Proxmox homelab — with an owner-approval gate governing what reaches external systems.

The first production application running on this platform is an autonomous content pipeline: web intelligence gathering through LLM-driven editorial decisions to image generation and scheduled publishing. But the platform itself is broader than any single application — it provides GPU compute allocation, tiered LLM serving, workflow orchestration, observability, and infrastructure lifecycle management.

The system runs on dedicated hardware with GPU passthrough per workload, mixed NVIDIA/AMD compute, and infrastructure monitoring through Grafana and Prometheus. This is a working operational system, not a demo or proof of concept. The architecture continues to evolve toward more formally governed execution, and this case study distinguishes what is operational from what is in active development.

---

## Architecture

{{< figure src="/images/tech4z-architecture.svg" alt="TECH4Z Agentic OS architecture diagram showing delivery, approval, orchestration, compute, and storage layers" >}}

---

## Key Engineering Decisions

### Workload Separation Across Compute Nodes

Training, experimentation, inference, and production generation run on separate Proxmox nodes, each with its own GPU passthrough assignment. Development workloads on the training node operate in different VMs/LXCs from production inference and generation workloads.

This separation reduces the blast radius of a failed experiment or bad model swap — a development workload crashing or consuming its GPU doesn't contend with production resources. The isolation is enforced at the hypervisor level through separate virtual machines and containers on distinct physical nodes with dedicated GPU assignments, not solely through software-level resource pools.

This mirrors how ML platform teams separate workloads at scale. The principle — workload segregation through compute-level boundaries — is the same whether implemented with Kubernetes node pools or Proxmox nodes with dedicated passthrough.

### Tiered LLM Serving

Heavy reasoning (32B parameter models) runs on dedicated GPU via Ollama. Lightweight inference (7B and 3B models) runs on separate CPU-bound instances. Every service references canonical network addresses, never localhost — the architecture is location-independent and could migrate to cloud endpoints without rewiring the orchestration layer.

This tiered approach keeps GPU resources focused on tasks that need them while handling simpler classification and formatting on CPU, reducing contention and improving throughput across the pipeline.

### Multi-Agent Pipeline

The content pipeline is a multi-agent system with defined roles and interfaces:

1. **Web intelligence** — Crawl4AI gathers trending topics and competitive signals
2. **CEO Agent** — An LLM (qwen2.5:32b on Ollama) evaluates inputs, decides what to produce, and generates downstream specifications
3. **Guardrail layer** — Automated quality and policy checks before content is generated
4. **Generation** — ComfyUI executes image generation with controlled prompt architectures and pinned parameters
5. **Delivery** — n8n workflows handle scheduling, platform-specific formatting, and publishing via APIs
6. **Owner approval gate** — Human review and approval before content reaches external platforms

Each agent has a defined responsibility and interface contract. The CEO agent produces a specification; it doesn't know or care how ComfyUI executes it. This separation of concerns makes the system testable and its failures diagnosable — when output quality drops, I can isolate whether the problem is in the intelligence gathering, the editorial decision, the generation parameters, or the delivery formatting.

### Evolving Governance Architecture

The platform is actively evolving toward a more formally governed execution model. This includes structured policy enforcement, explicit authorization boundaries, and formalized approval workflows beyond the current owner-approval gate.

The core multi-agent pipeline described above is operational and validated. The governance and policy-enforcement layers are in active development and will be documented as they reach production readiness.

### Backup and Recovery

Every system state represents iterative engineering work — trained models, tuned prompts, workflow configurations, infrastructure state. Proxmox Backup Server runs on a ZFS mirror with nightly sync to offline storage. This design has been validated through actual hardware-failure recovery scenarios — bare-metal restores from these backups have been performed successfully.

The backup architecture treats the entire platform as reproducible infrastructure: hypervisor configs, VM/LXC definitions, application state, and data are all captured. Recovery procedures are documented in internal runbooks.

---

## Technical Highlights

### GPU Integration: NVIDIA RTX 5070 Ti on Proxmox

Integrating NVIDIA's RTX 5070 Ti (Blackwell architecture) into a Proxmox-hosted Rocky Linux 9 workload required diagnosing several interacting issues across the driver, container runtime, and device-access layers.

**Driver compatibility:** Blackwell-generation GPUs require NVIDIA's open kernel modules (`--kernel-module-type=open` during driver installation). This requirement is documented by NVIDIA, but the failure mode when using the proprietary modules is silent — the driver installs without error but GPU compute operations fail at runtime. Diagnosing this required systematically testing driver configurations against the specific Proxmox + Rocky Linux 9 environment.

**LXC device access vs. VFIO passthrough:** GPU access in this deployment uses LXC container device permissions (cgroup device controllers and bind-mounted device nodes), which is distinct from PCI/VFIO passthrough used for full VM-level GPU isolation. The LXC approach provides lower overhead but introduces its own constraints — notably, eBPF-based cgroup device controllers must be correctly initialized.

**Container lifecycle behavior:** A clean container stop (`pct stop`) properly resets the cgroup device state. However, a `reboot` initiated from inside the container does not fully reinitialize the cgroup permissions, leaving the GPU device nodes inaccessible until the container is stopped and restarted from the Proxmox host. This distinction between in-guest reboot and host-managed stop/start behavior required empirical testing to identify.

### AMD RX 7900 XTX GPU Reset Issue

The RX 7900 XTX exhibits the well-known AMD GPU reset problem under Proxmox — after a VM or container shutdown, the GPU can enter a state where it does not reinitialize cleanly without a full host reboot.

This is a recognized issue across AMD RDNA3 GPUs in virtualized environments. The exact remediation applied in this deployment has not been independently re-verified against current logs, so the specific workaround is not described here. The problem and its general class of solutions (vendor-specific reset modules, host-level device reset scripts, kernel parameter tuning) are well-documented in the Proxmox and VFIO communities.

*This section will be updated with the verified remediation details once the configuration and logs are recovered.*

### Controlled Generation Parameters

Content generation uses a structured prompt architecture with pinned seeds, resolutions, model weights, and generation settings. Prompt and parameter metadata are embedded in output files and committed to version control.

This approach supports repeatable experiments and regression troubleshooting — when a LoRA fine-tune or base model change produces unexpected output, controlled parameters help isolate whether the regression is in the model, the prompt, the seed, or the generation settings.

Exact byte-identical reproduction may additionally depend on software versions, driver behavior, GPU execution paths, and dependency versions. The controlled-parameter approach provides strong practical reproducibility for debugging and quality assurance rather than guaranteed bitwise-identical output.

---

## Infrastructure at a Glance

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Hypervisor | Proxmox VE | Cluster orchestration, GPU passthrough, LXC/VM management |
| GPU Compute | RTX 5070 Ti, RX 7900 XTX, RTX 5060 Ti | Training, inference, production generation (mixed vendor) |
| LLM Serving | Ollama (tiered: 32B GPU / 7B CPU) | Agentic reasoning, content evaluation, lightweight classification |
| Image Generation | ComfyUI | Controlled image synthesis with LoRA fine-tunes |
| Orchestration | n8n | Pipeline automation, scheduling, delivery workflows |
| Observability | Grafana, Prometheus | Infrastructure metrics and pipeline health |
| Backup | Proxmox Backup Server on ZFS mirror | Full infrastructure and data protection with offsite sync |
| Version Control | Gitea (self-hosted) | Prompt architectures, configurations, runbooks |
| Asset Management | NetBox | Infrastructure inventory and network documentation |
| Web Intelligence | Crawl4AI | Trend analysis and competitive signal gathering |

---

## What This Demonstrates

- **Bare-metal GPU platform engineering** — Multi-GPU integration with mixed NVIDIA/AMD vendors, driver and device-access debugging on current-generation hardware, and workload separation using Proxmox virtualization
- **Multi-agent AI system design** — Production pipeline with defined agent roles, interface contracts, guardrail layers, controlled generation parameters, and human-in-the-loop approval gates
- **Infrastructure automation** — n8n workflow orchestration with API integrations, tiered compute allocation, and autonomous batch operations under governed execution
- **Production reliability engineering** — Backup/restore with validated recovery, observability stack, version-controlled configurations, workload separation, and documented operational procedures
- **Full-stack infrastructure ownership** — Sole engineer across every layer: hardware selection and procurement, hypervisor management, GPU driver integration, application deployment, LLM serving architecture, workflow automation, and operational monitoring

---

## Verified Engineering Results

*This section is reserved for sanitized evidence demonstrating operational results — test outcomes, pipeline execution metrics, monitoring dashboards, and validated recovery exercises. Evidence will be published as it is sanitized for public presentation.*

Supporting evidence, configurations, and detailed technical documentation are available upon request for interview discussions.

---

*This case study describes the TECH4Z Agentic OS platform as designed, built, and operated by Jaime Ramirez. Architecture diagrams are sanitized for public presentation. The platform architecture is actively evolving — capabilities described as in development are distinguished from operational components throughout.*
