# k8s-security-reasoning — A Minimal Learning Asset

## Purpose

This repository is a **minimal learning asset** designed to explore cloud security
*reasoning* through a small, concrete Kubernetes exercise.

It focuses on understanding **security boundaries**, **failure modes**, and
**risk tradeoffs** that emerge at the intersection of:

- the Kubernetes control plane
- the cloud provider control plane (IAM, compute, network, storage)
- workload identity, networking, and persistence

Rather than presenting best-practice checklists, this repository emphasizes
**first principles**, **systems thinking**, and **hands-on exploration**.

The goal is not Kubernetes mastery, but clarity around *where security
responsibility lives* — and where it does not.

---

## Who This Is For

This repository is intended for:

- Cloud engineers, SREs, and DevOps practitioners
- Security engineers and architects
- Customer Success Engineers, TAMs, and Solutions Architects
- Anyone seeking a grounded way to reason about Kubernetes and cloud security

It assumes basic familiarity with Kubernetes concepts, but **does not require
deep operational expertise**.

This is **not** production-ready infrastructure.
It is a shared reasoning surface.

---

## Learning Goals

By working through this repository, you should be able to:

- Explain what Kubernetes *does* and *does not* secure
- Understand Kubernetes RBAC as control-plane authorization
- Reason about default failure modes (open vs. closed)
- Identify where cloud compute, network, and storage intersect with Kubernetes workloads
- Understand why security and cost risk often exists *between* systems
- Hold a grounded conversation about Kubernetes security using first principles

---

## Why NGINX?

This repository uses **NGINX** as the example workload intentionally.

NGINX is:
- widely understood
- operationally boring
- easy to expose over the network
- free of application-specific complexity

We are **not** teaching NGINX.

NGINX serves as a predictable execution surface so attention stays on:
- identity
- compute placement
- network exposure
- storage persistence
- control-plane boundaries

---

## How to Use This Repository

This repository contains two artifacts:

- `README.md` — the reasoning guide
- `main.yaml` — a single Kubernetes manifest containing all resources

Recommended approach:

1. Read the README to understand *why* each resource exists
2. Review `main.yaml` to see how those ideas are expressed declaratively
3. Create and inspect a Kubernetes cluster
4. Apply the manifest
5. Observe Kubernetes resources
6. Observe related cloud control-plane resources
7. Deliberately break things and reason about what fails and why
8. Tear everything down and verify cleanup across layers

---

## Kubernetes as an Abstraction Layer

Kubernetes is an abstraction layer for:

- **Compute**
- **Network**
- **Storage**

It provides a consistent control plane across cloud providers and infrastructure types.

From a security perspective, Kubernetes is a **coordination system**, not a complete
security solution.

Many critical controls live outside Kubernetes — particularly in cloud compute,
networking, and storage services.

Understanding where Kubernetes responsibility ends is essential.

---

## Grounding Security Principles

This exercise is grounded in three principles:

### 1. Least Privilege
Access should be narrowly scoped and explicitly granted.

In Kubernetes:
- ServiceAccounts
- Roles and RoleBindings
- Namespace scoping

In GCP/AWS:
- IAM roles
- service identities
- access to APIs, storage, and network

### 2. Defense in Depth
No single control is sufficient.

Security emerges from layering:
- identity
- network
- storage
- workload configuration

### 3. Visibility
Security requires the ability to observe and reason about system state and change
across **all layers**, not just Kubernetes.

---

## A Simple Threat Reasoning Approach

This exercise is not a formal threat model.
It is a **way of orienting your thinking** when working with Kubernetes as an
abstraction layer over cloud infrastructure.

The goal is to reason clearly about **defaults, failure modes, and attacker
opportunity**, even when you do not yet know every detail of the system.

---

### Defaults and Failure Modes

A useful starting question in any security discussion is:

> *When something is missing or misconfigured, does the system fail open or fail closed?*

Examples you will encounter in this exercise:

- **Kubernetes RBAC**: default **closed**
- **Cloud IAM**: default **closed**
- **Kubernetes networking**: default **open**
- **NetworkPolicy**: once applied, default becomes **closed**

---

### Reasoning from a Foothold

Assume a simple starting condition:

> *An attacker has execution inside a container.*

From there, reasoning can unfold in a predictable sequence.

---

### 1. Identity

What identity does the workload already have?

Consider:
- ServiceAccount tokens
- Mounted credentials or secrets
- Cloud workload identity

---

### 2. Network

What can the workload reach?

Consider:
- other pods and namespaces
- cluster services
- external endpoints

---

### 3. Data

What data is accessible?

Consider:
- persistent volumes
- databases or object storage
- data that outlives the pod

---

### 4. Escape and Expansion

Can containment be broken?

Consider:
- privileged containers
- node access
- cloud API access

---

### Visibility as a Multiplier

Logs, metrics, traces, and higher-level CNAPP platforms do not prevent compromise.
They change how quickly and clearly activity can be understood.

---

## What `main.yaml` Contains

The single manifest intentionally includes:

- **Namespace** — isolation boundary
- **ServiceAccount** — workload identity
- **Role & RoleBinding** — least-privilege Kubernetes API access
- **Deployment (NGINX)** — workload lifecycle and compute placement
- **PersistentVolumeClaim** — state and data persistence
- **Service** — network exposure
- **NetworkPolicy** — explicit network allow rules

Each resource exists to make a security-relevant boundary visible.

---

For [AWS Insturctions](AWS_Instructions.md)

For [GCP Instructions](GCP_Instructions.md)

---

## Reflection

After teardown, reflect:

- Which layers cleaned up automatically?
- Which required human verification?
- Where does Kubernetes abstraction hide risk?

---

## Contributing & Discussion

This repository is intentionally minimal and exploratory.

It is not a best-practices guide or production reference.
It exists as a **shared reasoning surface**.

If you notice inaccuracies, missing considerations, or alternative ways to
reason about these boundaries, you’re invited to:

- open an issue
- submit a pull request
- or reach out directly to the maintainers and contributors

Thoughtful discussion and real-world counterexamples are welcome.
