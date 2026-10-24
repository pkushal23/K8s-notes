# K8s-notes

My notes, labs, and debugging logs while learning Kubernetes for AI infrastructure.

## Goal
Reach CKA-level fluency, then build an LLM serving project on Kubernetes with GPU metrics, autoscaling, and networking.

## Setup
- Windows laptop, WSL2 (Ubuntu), Docker Desktop
- kind cluster: 1 control plane + 3 workers (`labs/kind-config.yaml`)

## Progress
| Week | Topic | Status |
|---|---|---|
| 0 | Setup | Done |
| 1 | Control plane and the apply flow | In progress |
| 2 | Pods, resources, probes | |
| 3 | Scheduling | |
| 4 | Rollouts, HPA, PDB | |
| 5 | Networking | |
| 6 | Storage, config, RBAC | |
| 7 | Cluster ops, Helm | |
| 8 | Kueue, open source | |

## Notes
- [Day 1: Control plane and the apply flow](notes/day-01-apply-flow.md)

## How each entry is written
Each note follows [TEMPLATE.md](TEMPLATE.md): what it is, what I ran, what broke, how I diagnosed it, what I learned.
