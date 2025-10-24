# YAOOK OpenStack Deployment

## Description

**YAOOK Version:** 2024.1 (Caracal)  
**Kubernetes:** v1.31.13 (Kubespray)  
**Storage:** Longhorn + External Ceph Integration (Squid 19.2.3)  
**Networking:** Calico CNI + OVN  
**Load Balancer:** MetalLB  

### Services Deployed
- Core OpenStack Services
- OVN Networking
- Horizon
- Octavia (in progress)

### Not Currently Supported
> As of October 25, 2025

- Masakari
- Watcher
- Skyline

## Environment

### Machine Inventory

| Name    | vCPU | RAM (GB) | HDD (GB) | IPv4                        |
|---------|------|----------|----------|-----------------------------|
| k8s-ctl | 2    | 4.00     | 60.00    | 192.168.1.24                |
| k8s-n1  | 8    | 16.00    | 310.00   | 192.168.1.26, 10.233.97.192 |
| k8s-n2  | 8    | 16.00    | 310.00   | 192.168.1.27, 10.233.122.64 |
| k8s-n3  | 8    | 16.00    | 310.00   | 192.168.1.28, 10.233.121.192|

## Key Findings

- **YAOOK** stands for "Yet Another OpenStack on Kubernetes"
- No official image repository exists for operators - images must be built locally
- YAOOK Operators for each major service must be deployed first from compiled images as prerequisites
- YAOOK currently does not support Masakari, Watcher, or Skyline
- YAOOK uses custom Kubernetes operators with Custom Resources (CRs) instead of Helm charts for OpenStack services
  - Each service (Keystone, Nova, Neutron, etc.) has a dedicated operator managing its lifecycle through a state machine pattern
  - Operators are installed via Helm, but OpenStack services are deployed via CRs
- Professional user base of ~50 members from companies like Cloud&Heat and STACKIT, primarily communicating through GitLab issues
- **Known Issue:** Bitnami image locations are broken due to recent manifest changes
  - **Workaround:** Use `docker pull bitnamilegacy/IMAGE`
- Octavia module exists but is not yet in a working state