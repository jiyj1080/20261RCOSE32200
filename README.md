# 20261RCOSE32200

# Project 2: In-Kernel Virtual-IP Load Balancer via eBPF & TC

A high-performance, in-kernel load balancer implemented using extended Berkeley Packet Filter (eBPF). This project operates entirely within the Linux kernel's Traffic Control (TC) subsystem, performing Network Address Port Translation (NAPT) and session-aware traffic distribution without the overhead of context switching to user-space.

## 🚀 Key Features

* **Transparent VIP Routing (Phase 1):** Intercepts client requests targeting a Virtual IP (VIP) and translates the destination to a physical backend server dynamically. Performs symmetric source address translation on egress to maintain TCP state integrity.
* **TCP Flow-Based Load Balancing (Phase 2):** Distributes incoming traffic across multiple backend servers (`main` and `backup`) using a round-robin scheduling algorithm based on atomic flow counters.
* **Session Affinity via BPF Maps:** Utilizes a `BPF_MAP_TYPE_HASH` to store 5-tuple connection signatures (Source IP/Port, Dest IP/Port, Protocol). This ensures that once a TCP 3-way handshake is initiated, all subsequent packets of that flow are consistently routed to the same backend server.
* **Automated Checksum Recalculation:** Safely modifies L3 (IPv4) and L4 (TCP) headers directly in the kernel datapath, leveraging BPF helper functions to recalculate checksums and prevent packet drops.

## 🛠️ Technology Stack

* **Environment:** Linux Kernel v5.15, Ubuntu 22.04
* **Languages:** C (eBPF programs), Bash (Network setup scripts)
* **Tools:** LLVM/Clang, `bpftool`, `tc` (Traffic Control), `iproute2`, `tcpdump`
* **Core Concepts:** eBPF Maps (Array, Hash), Linux Network Namespaces, TC Ingress/Egress Hooks, Atomic Operations, Low-level Packet Parsing.

## 🏗️ System Architecture

The project simulates a hyper-scale infrastructure using isolated Linux Network Namespaces connected via virtual ethernet (`veth`) pairs and a bridge (`br-db`).

```text
[ ns_client ] (10.0.0.2) 
      |
 (veth-cli) 
      |
 [ TC eBPF Hook @ veth-cli-h ] <--- NAPT & Load Balancing logic applied here
      |
   [ br-db ] (192.168.1.1)
      |
      +---> [ ns_db_main ]   (192.168.1.100:18080)
      +---> [ ns_db_backup ] (192.168.1.101:28080)
