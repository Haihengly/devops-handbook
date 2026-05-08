# Troubleshooting - Kubernetes Alerting

This document lists common issues and how to identify them in the Kubernetes monitoring setup using Helm, Prometheus, and Alertmanager.

---

## Alert Not Appearing

- Missing `metadata.labels.release=monitoring`
- PrometheusRule not applied in correct namespace
- Rule not picked up by Prometheus
- Incorrect label format in rule definition

---

## Alert Not Triggering

- Expression is incorrect or too strict
- Metrics are not available from Prometheus
- `for:` duration not met yet
- Target service not exposing metrics

---

## Alertmanager Not Receiving Alerts

- Prometheus not connected to Alertmanager
- Incorrect Alertmanager service configuration
- Alert not matching routing rules
- Alert suppressed due to grouping rules

---

## Notification Not Sent (Telegram / Slack / Email)

- Webhook URL is incorrect
- Missing `/alerts` endpoint in configuration
- Receiver not properly defined in AlertmanagerConfig
- Network/DNS issue inside cluster

---

## Application Alerts Not Working

- Missing `scope=namespace` label
- Alert routing does not match scope rules
- PrometheusRule missing namespace selector
- AlertmanagerConfig not matching labels

---

## General Debug Checklist

- Check Prometheus targets:
  ```bash
  kubectl -n monitoring get pods