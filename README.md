# Setup Highly Available Kubernetes Cluster with HAProxy, Keepalived & kubeadm

This repository provides a reference architecture for deploying a **highly available (HA) Kubernetes cluster** using:
- **HAProxy** for TCP load balancing across master nodes
- **Keepalived** for a floating Virtual IP (VIP) supporting failover
- **kubeadm** for rapid and standardized Kubernetes cluster bootstrapping

---

## Architecture Overview

- Two or more **load balancer nodes** (running HAProxy + Keepalived) provide a single highly available VIP for Kubernetes API traffic.
- Three or more **Kubernetes master nodes**, each running stacked etcd, form the control plane.
- One or more **worker nodes** join the cluster for running application workloads.

The load balancer VIP (`<KUBE_VIP>`) is used by all clients, worker nodes, and as the `controlPlaneEndpoint` for the cluster API.

![Architecture Overview](/img/k8s-ha-architecture.png)

---

## Setup Guides

This repository contains **two detailed README guides** for modular step-by-step deployment:

- [HAProxy & Keepalived Setup](/docs/haproxy-keepalived-setup.md)
  Learn how to install and configure HAProxy for TCP load balancing of the API Server and Keepalived for Virtual IP failover.

- [Kubeadm Cluster Setup](/docs/kubeadm-ha-cluster-setup.md)  
  Step-by-step instructions for installing Kubernetes using kubeadm, configuring certificates, and bootstrapping a multi-master cluster.

**Please refer to these files for fully detailed procedures.**

---

## High-Level Setup Flow

1. **Provision VMs/Servers**
    - Ensure all nodes (load balancer, control planes, workers) are running Ubuntu 24.04 and have network connectivity.
2. **Configure HAProxy + Keepalived**
    - Follow the [HAProxy & Keepalived Setup](/docs/haproxy-keepalived-setup.md) to create the HA load balancer pair with a floating VIP.
    - Validate VIP failover and proxying for port `6443` to all master nodes.
3. **Generate TLS Certificates**
    - Use a secure CA and create certificates with all VIP/master addresses in SANs.
    - Distribute the CA and relevant certs securely to cluster nodes.
4. **Deploy Control Plane**
    - Follow the [Kubeadm Cluster Setup](/docs/kubeadm-ha-cluster-setup.md) to initialize the first master with `kubeadm init --control-plane-endpoint <KUBE_VIP>:6443`.
    - Join additional masters using the provided join command with `--control-plane`.
5. **Join Worker Nodes**
    - Run the kubeadm join command as instructed to add workers to the cluster.
6. **Deploy a CNI Plugin**
    - Choose and apply a network add-on like Calico, Flannel, or Cilium for pod networking.
7. **Validate**
    - Use `kubectl get nodes` and `kubectl get pods -n kube-system` to verify the healthy state of the HA cluster.

---

## Useful References

- [Kubernetes Official High-Availability Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [HAProxy Documentation](https://www.haproxy.org/)
- [Keepalived Documentation](https://keepalived.org/)

---

## License

This project is distributed under the Apache-2.0 License.

---
