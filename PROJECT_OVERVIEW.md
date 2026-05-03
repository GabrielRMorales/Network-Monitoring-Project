# Network Monitoring Homelab

A multi-node infrastructure monitoring and alerting system built using Prometheus, Grafana, and Nagios to simulate real-world observability across Linux virtual machines.

---

# Project Goal

To design and implement a full-stack monitoring environment capable of:

- Collecting system-level metrics from multiple Linux machines
- Visualizing performance data in real time
- Detecting host and service failures
- Simulating enterprise-style observability architecture

---

# Architecture Overview

Prometheus Stack:

Prometheus → Node Exporter → Grafana dashboards

Nagios Stack:

Nagios → plugin-based health checks (PING, SSH, system status)

---

# Environment

- 2 Windows host machines running VirtualBox
- 2 Ubuntu Server VMs:
  - Monitoring VM (Prometheus, Grafana, Nagios)
  - Target VM (Node Exporter)

---

# Key Features

- Multi-node metrics collection (CPU, memory, disk, network)
- Real-time dashboards in Grafana
- Host/service monitoring via Nagios
- Distributed observability across VMs
- Persistent service management using systemd

---

# Tools & Technologies

- Prometheus
- Grafana
- Nagios Core
- Node Exporter
- Ubuntu Server
- VirtualBox
- systemd

---

# Key Outcomes

- Built a working observability stack from scratch
- Integrated metrics and alerting systems across multiple VMs
- Configured system services for persistence and reliability
- Diagnosed and resolved real-world deployment issues (service discovery, plugin paths, dashboards, networking)

---

# Future Improvements

- SNMP-based network device monitoring
- Centralized logging (ELK or Loki stack)
- Alerting integrations (email/Discord)
- Containerized deployment (Docker)