# Kubernetes Alerting 

> This project demonstrates a Kubernetes alerting setup using Helm, Prometheus, and Alertmanager.

---

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [What It Includes](#what-it-includes)
- [Alerting Design](#alerting-design)
- [Notification](#notification)
- [Deployment Flow](#deployment-flow)
- [Documentation](#documentation)
- [Notes](#notes)

---

## Overview

This project sets up a Kubernetes monitoring and alerting system using Helm. Prometheus is used to collect and store cluster metrics, while Alertmanager handles and routes alerts. It helps detect issues like high CPU/memory usage, pod failures, and node problems, and sends real-time notifications when something goes wrong.

---

## Architecture

> Kubernetes Cluster → Prometheus → Alert Rules → Alertmanager → Telegram Bot (topic-based routing)

---

## What it includes

- Prometheus for cluster metrics collection  
- Alertmanager for alert routing and notification handling  
- kube-prometheus-stack deployed via Helm  

- Alert coverage:
  - Node health (CPU, memory, disk usage)
  - Pod health (CrashLoopBackOff, Pending, Restarting)

- Label-based routing using `scope` and `alertgroup`  
- Telegram bot integration with topic-based routing  
- Go template formatting for structured alert messages  

---

## Deployment

The monitoring stack is installed using Helm with the **kube-prometheus-stack chart**. This deploys Prometheus, Alertmanager, Grafana, and the Prometheus Operator in the monitoring namespace.

All resources such as alert rules and configurations are managed using Kubernetes manifests for version control and consistency.

---

## Alerting Design

Alerts are organized by scope to make them easier to manage and route.

- Node alerts → infrastructure-level issues (CPU, memory, disk)  
- Pod alerts → application-level issues (crashes, restarts, pending states)  

Alertmanager routes all alerts based on labels like `scope` and `alertgroup`, ensuring each alert goes to the correct destination.

---

## Notification

Alertmanager sends alerts to a Telegram bot for real-time monitoring.

Alerts are routed into a Telegram group using topic-based routing, allowing separation of alerts by category (CPU, RAM, Disk, Pod, etc.).

This improves clarity and makes it easier to track specific system issues in real time.

---

## Summary

> This project helps monitor a Kubernetes cluster in real time. It detects problems early and sends alerts automatically, making it easier to keep the system stable and healthy.

---

## Deployment Flow

1. Install kube-prometheus-stack using Helm  
2. Verify Prometheus and Alertmanager pods are running  
3. Apply PrometheusRule manifests  
4. Configure Alertmanager routing rules  
5. Enable notification channels  

---

## Notes

- All monitoring is managed using Helm for easier upgrades and maintenance
- Alerts are grouped and routed to reduce noise
- System is designed for scalability across multiple namespaces

---

## Documentation
- [Setup Guide](./setup.md)
- [Alert Rules](./rules.md)        
- [Trouble Shooting](./troubleshooting.md)      

---

## Repository

- GitHub Repository: https://github.com/Haihengly/k8s-alerting-platform
