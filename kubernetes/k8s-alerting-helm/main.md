# Kubernetes Alerting Setup

> This project demonstrates a Kubernetes alerting setup using Helm, Prometheus, and Alertmanager.

---

## Table of Contents
- [Overview](#overview)
- [Requirements](#requirements)
- [Architecture](#architecture)
- [Alerting Design](#alerting-design)
- [Notification](#notification)
- [Deployment Flow](#deployment-flow)
- [Documentation](#documentation)
- [Notes](#notes)

---

## Overview

This project sets up a Kubernetes monitoring and alerting system using Helm. Prometheus is used to collect and store cluster metrics, while Alertmanager handles and routes alerts. It helps detect issues like high CPU/memory usage, pod failures, and node problems, and sends real-time notifications when something goes wrong.

## ⚙️ Requirements

- Kubernetes cluster is running
- Helm installed
- Namespace monitoring exists (used for Prometheus, Alertmanager, rules)
- kube-prometheus-stack chart
- Basic Prometheus & Alertmanager knowledge
- Email or Telegram or Slack for alerts notifications

---

## Architecture

> Kubernetes Cluster → Prometheus (metrics collection) → Alert Rules → Alertmanager (routing) → Notifications (Telegram / Slack / Email)

---

## Deployment

The monitoring stack is installed using Helm with the **kube-prometheus-stack chart**. This deploys Prometheus, Alertmanager, Grafana, and the Prometheus Operator in the monitoring namespace.

All resources such as alert rules and configurations are managed using Kubernetes manifests for version control and consistency.

---

## Alerting Design

Alerting is organized by scope to separate responsibilities and improve clarity.

- Cluster-level alerts → Monitor node and infrastructure health (e.g. node failure, resource pressure)
- Application-level alerts → Monitor workloads and pods across namespaces (e.g. pod crash, restart loops)

Prometheus evaluates rules and sends alerts to Alertmanager when conditions are met.

---

## Notification

Alertmanager sends alerts to external channels such as Telegram, Slack, or email.

This ensures real-time visibility when issues occur in the cluster.

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
