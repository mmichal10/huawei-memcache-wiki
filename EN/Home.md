# MemCache Wiki

Home page for MemCache Hybrid documentation, covering release planning, feature descriptions, case studies, and FAQs.

## Overview

- [Roadmap](Roadmap.md)
- [FAQ](FAQ.md)
- [MemFabric FAQ](https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md)

## Features

- [Metadata high availability (Kubernetes Lease)](How-to-Deploy-a-MemCache-HA-Cluster.md)
- [Metadata high availability (etcd, recommended)](How-to-Deploy-a-MemCache-HA-Cluster-etcd.md)
- [Cluster scaling](Cluster-Scaling.md)
- [UBSIO Kubernetes CSI raw block: StatefulSet + volumeClaimTemplates](UBSIO-K8s-CSI-Raw-Block-Solution.md)

## Case Studies

#### DSV4
- [DSV4-Flash + A2 single-node co-deployment](MMC-Best-Practice-DSV4-Flash-A2-Single-Node.md)
- [DSV4-Flash + A2 multi-node PD disaggregation](MMC-Best-Practice-DSV4-Flash-A2-PD-Disaggregation.md)
- [DSV4-Flash + A3 multi-node PD disaggregation, HCCS multi-node pooling (recommended)](MMC-Best-Practice-DSV4-Flash-A3-PD-Disaggregation-Recommended.md)
- [DSV4-Flash + A3 multi-node PD disaggregation, PCIe single-node pooling (alternative)](MMC-Best-Practice-DSV4-Flash-A3-PD-Disaggregation-Alternative.md)

#### GLM-5.2
- [GLM-5.2 + four A3 nodes, PD disaggregation](MMC-Best-Practice-GLM-5.2-A3-4-Node-PD-Disaggregation.md)
- [GLM-5.2 + four A3 nodes, multi-tier pooling and PD disaggregation](MMC-Best-Practice-Multi-Tier-Pooling-GLM-5.2-A3-4-Node-PD-Disaggregation.md)

#### GLM-5.1
- [GLM-5.1 + four A3 nodes, PD disaggregation](MMC-Best-Practice-GLM-5.1-A3-4-Node-PD-Disaggregation.md)
- [GLM-5.1 + eight A2 nodes, ROCE multi-node pooling (recommended)](MMC-Best-Practice-GLM-5.1-A2-8-Node-PD-Disaggregation-ROCE-Multi-Node-Pooling.md)
- [GLM-5.1 + eight A2 nodes, PCIe single-node pooling (alternative)](MMC-Best-Practice-GLM-5.1-A2-8-Node-PD-Disaggregation-PCIE-Single-Node-Pooling.md)

#### GLM-4.7
- [GLM-4.7 + two A2 nodes, co-deployment](MMC-Best-Practice-GLM-4.7-A2-Two-Node-Co-Deployment.md)

#### Other
- [Expand usable memory in an A3 HCCS network](Expand-Usable-Memory-on-A3-with-MemCache.md)
- [MemCache + vLLM A3 standalone deployment case study](MemCache-vLLM-A3-Standalone-Deployment-Case-Study.md)
- [DRAM + SSD multi-tier pooling configuration guide](DRAM-SSD-Multi-Tier-Pooling-Configuration-Guide.md)
- [Enable UB and UBoE pooling on 950DT and 950PR](Enable-UB-and-UBoE-Pooling-for-950DT-and-950PR.md)

## Quick Start

- [Deploy a MemCache HA cluster (Kubernetes Lease)](How-to-Deploy-a-MemCache-HA-Cluster.md)
- [Deploy a MemCache HA cluster with etcd (recommended)](How-to-Deploy-a-MemCache-HA-Cluster-etcd.md)
- [Connect Grafana and Prometheus for monitoring](MemCache-Grafana-Prometheus-Integration-Tutorial.md)

## Release Branching

- [Development and release cadence](Development-and-Release-Cadence-Principles.md)
