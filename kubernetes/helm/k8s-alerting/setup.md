# Setup Guide

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Deployment Flow](#deployment-flow)
- [Install kube-prometheus-stack](#install-kube-prometheus-stack)
- [Apply Secrets](#apply-secrets)
- [Configure Alertmanager](#configure-alertmanager)
- [Apply PrometheusRules](#apply-prometheusrules)
- [Verify Everything](#verify-everything)
- [Test Alerts](#test-alerts)

---

## Prerequisites

- Kubernetes cluster is running
- Helm installed
- `kubectl` configured and connected to cluster
- Telegram bot created with bot token and chat ID ready

---

## Deployment Flow

1. Install kube-prometheus-stack using Helm  
2. Verify Prometheus and Alertmanager pods are running  
3. Apply PrometheusRule manifests  
4. Configure Alertmanager routing rules  
5. Enable notification channels  

---

## Install kube-prometheus-stack

Add the Helm repository:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the chart:
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace
```

Verify pods are running:
```bash
kubectl get pod -n monitoring
```

---

## Apply Secrets

Apply alertmanager bot token secret:
```bash
kubectl apply -f secret/secret.yml -n monitoring
```

Apply alertmanager templates secret:
```bash
kubectl -n monitoring create secret generic alertmanager-templates \
  --from-file=alertmanager/templates \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify secrets exist:
```bash
kubectl get secret -n monitoring | grep alertmanager
```

---

## Configure Alertmanager

Apply alertmanager configuration using Helm upgrade:
```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f alertmanager/routes.yml \
  -f alertmanager/receivers.yml \
  -f alertmanager/templates-mount.yml \
  --set grafana.replicas=1
```

Restart alertmanager to pick up new config:
```bash
kubectl -n monitoring rollout restart sts/alertmanager-monitoring-kube-prometheus-alertmanager
```

---

## Apply PrometheusRules

Apply node rules:
```bash
kubectl apply -f rules/node/
```

Apply pod rules:
```bash
kubectl apply -f rules/pod/
```

Verify rules are loaded:
```bash
kubectl get prometheusrule -n monitoring
```

---

## Verify Everything

Check all pods are running:
```bash
kubectl get pod -n monitoring
```

Check Prometheus targets:
```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```
Open `http://localhost:9090/targets` and verify targets are UP.

Check Alertmanager config:
```bash
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093
```
Open `http://localhost:9093` and verify routes and receivers are correct.

Check templates are mounted:
```bash
kubectl -n monitoring exec -it alertmanager-monitoring-kube-prometheus-alertmanager-0 \
  -c alertmanager -- ls -lah /etc/alertmanager/custom-templates
```

---

## Test Alerts

Test Telegram bot is working:
```bash
curl -X POST "https://api.telegram.org/bot<YOUR_TOKEN>/sendMessage" \
  -d chat_id=<YOUR_CHAT_ID> \
  -d text="Test alert from KubeMonitorBot"
```

Test alert is firing by checking Prometheus rules:
```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```
Open `http://localhost:9090/rules` and verify rules are loaded and firing correctly.

---

## Notes

- Always apply secrets before running helm upgrade
- Run helm upgrade whenever alertmanager config changes
- Run kubectl apply whenever rules change
- Check alertmanager logs if alerts are not received:
```bash
kubectl logs -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0 -c alertmanager
```