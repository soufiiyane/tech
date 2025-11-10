# WordPress + Monitoring Infrastructure Overview

This repository contains the GitOps manifests that drive the production WordPress workload and its observability stack on an RKE2 single‑node cluster. The goal of this document is to give future operators (and future‑me) a concise map of how everything fits together so that changes or incident response can happen quickly.

---

## Platform Summary

- **Cluster**: RKE2 (single EC2 instance, 8 GiB RAM / 100 GiB disk).
- **Storage**: Longhorn with the `longhorn-retain` StorageClass (Retain reclaim policy, RWO volumes).
- **Ingress**: NGINX Ingress Controller with cert-manager (ClusterIssuer `letsencrypt-prod`) issuing TLS for public endpoints.
- **GitOps**: Argo CD watches this repo.
  - `manifests/wordpress` → WordPress workload (Application: `wordpress-stack`).
  - `manifests/monitoring` → Prometheus, Alertmanager, Grafana, exporters, reloader, etc. (Application: `monitoring-stack`).

All manifests are plain YAML—no Helm charts—so Git is the source of truth.

---

## WordPress Stack (manifests/wordpress)

- **Namespace**: `wordpress`.
- **Deployments**:
  - `wordpress` (image `soufiiyane/tmt-wordpress:latest`).
  - `mysql` (MySQL 8.0).
  - `phpmyadmin` (phpMyAdmin 5.2.1).
- **Persistent Volumes**:
  - `mysql-data` PVC – 20 GiB.
  - `wordpress-uploads` PVC – 20 GiB.
  - `wordpress-backup` PVC – 10 GiB (cronjob backups).
- **Secrets**:
  - MySQL credentials via SealedSecret (`mysql-sealedsecret.yaml`).
  - SMTP credentials (SealedSecret).
- **Ingress**:
  - Host `usemirai.site` (TLS via cert-manager).
- **Backups**:
  - CronJob dumps database to the backup PVC (see `wordpress-cronjob-backup.yaml`).



---

## Monitoring Stack (manifests/monitoring)

### Namespace

- `monitoring` (created via `namespace.yaml`).

### Core Services

- **Prometheus** (`prometheus` deployment)
  - Image: `prom/prometheus:v2.52.0`.
  - PVC: `prometheus-data` (5 GiB, Longhorn retain).
  - ConfigMap: `prometheus-config` defines scrape jobs:
    - API server, nodes, pods, services.
    - `kube-state-metrics`.
    - `node-exporter` (per-node metrics).
    - `kubernetes-cadvisor`.
    - `wordpress-blackbox` (HTTP probe to `https://usemirai.site/`, default cadence 2 h).
  - Rules (`prometheus-rules` ConfigMap):
    - Deployment availability for `mysql`, `phpmyadmin`, `wordpress`.
    - WordPress public endpoint (blackbox probe).
  - Ingress: `promoth.usemirai.site` (TLS via cert-manager).
  - Deployment strategy: `Recreate` (avoids PVC lock issues).

- **Alertmanager**
  - Image: `quay.io/prometheus/alertmanager:v0.27.0`.
  - PVC: `alertmanager-data` (1 GiB).
  - Config: sealed secret (`alertmanager-config-sealedsecret.yaml`) with Slack webhook.
  - Ingress: `alertmanager.usemirai.site`.

- **Grafana**
  - Image: `grafana/grafana:10.4.3`.
  - PVC: `grafana-data` (2 GiB).
  - Datasource ConfigMap pre-configures Prometheus.
  - Dashboard provisioning:
    - `Kubernetes / Views / Global` (ID 15757).
    - `Kubernetes / Views / Nodes` (ID 15759).
  - Ingress: `grafana.usemirai.site`.
  - Change admin credentials by rotating the sealed secret (`grafana-secret.yaml` locally → `kubeseal`).

### Supporting Components

- **kube-state-metrics**: Deployment + RBAC for Kubernetes object metrics.
- **node-exporter**: DaemonSet that exposes node CPU/memory/disk metrics to Prometheus.
- **Blackbox Exporter**: Deployment probing the public WordPress endpoint.
- **Stakater Reloader** (`manifests/monitoring/reloader`):
  - Watches ConfigMaps/Secrets and restarts Prometheus/Grafana/Alertmanager automatically when configs change (no manual `/-/reload`).
- **Ingress TLS**: All monitoring ingresses use `letsencrypt-prod` ClusterIssuer.

---

## Alerting & Notifications

- **Alert Scope**:
  - WordPress deployments missing replicas (>2 m).
  - Public WordPress endpoint failing blackbox probe.
  - (Extend by adding new groups to `prometheus-rules.yaml`; Slack receives only explicitly defined alerts.)
- **Testing**:
  - Scale deployment to zero (alerts fire for available replicas).
  - Break image tag or make endpoint unreachable for blackbox alert.

---

## GitOps Workflow

1. Edit YAML in the repo (WordPress or monitoring directories).
2. `kubeseal` any secrets before committing.
3. Commit + push to `main`.
4. Argo CD (`wordpress-stack`, `monitoring-stack`) auto-syncs.
5. Stakater reloader restarts workloads when their configs/secrets change.
6. In case of drift:
   - Argo CD ignore-diff annotations on Prometheus/Grafana/Alertmanager prevent false positives from reloader.
   - Check Argo CD app status; sync if needed.

> **Deploy split in prod**: Ensure `wordpress-stack` points at `manifests/wordpress` and `monitoring-stack` points at `manifests/monitoring`. Apply `manifests/argocd/monitoring-application.yaml` to register the monitoring app.

---

## Operational Tips

- **Secrets**: Regenerate sealed secrets with:
  ```bash
  kubeseal --controller-name=sealed-secrets-controller \
           --controller-namespace=kube-system \
           --format yaml \
           < plaintext.yaml > sealed.yaml
  ```
- **Ingress hosts**: Update certificates or DNS in the ingress manifests if domains change.
- **Blackbox interval**: Adjust `scrape_interval` under `wordpress-blackbox` to tune probe cadence (default 2 h). Remember to revert after testing.
- **Dashboard updates**: Modify JSON under `grafana-dashboards.yaml` (or add new ConfigMaps) and commit.
- **PVC sizing**: Monitor via Grafana PVC table. Increase claims cautiously because node disk is ~100 GiB with ~20 GiB free.
- **Longhorn**: If volumes show degraded replicas, inspect Longhorn UI or add alerts around `longhorn_volume_robustness`.
- **Ceremony for new cluster**:
  - Deploy Longhorn, NGINX ingress, cert-manager, Sealed Secrets controller.
  - Ensure DNS points to the ingress.
  - Reseal secrets (sealed secrets are cluster-specific).
  - Apply Argo CD Applications.

---

## Troubleshooting Checklist

- **Prometheus CrashLoop**:
  - Stale TSDB lock? Deployment strategy is `Recreate`; if issue persists, delete pod to clear lock.
  - Config syntax? Check `/etc/prometheus/prometheus.yml` inside pod (`kubectl exec`).
- **No metrics in Grafana**:
  - Verify `kube-state-metrics` and `node-exporter` pods are running.
  - Check Prometheus targets (`/api/v1/targets`).
- **No Slack alerts**:
  - Confirm SealedSecret decrypted (`kubectl get secret alertmanager-config -n monitoring`).
  - Inspect Alertmanager UI; ensure route matches `alertname`.
- **Ingress issues**:
  - Check cert-manager cert (`kubectl describe certificate ...`).
  - Validate DNS resolves to the ingress endpoint.
- **WordPress downtime**:
  - Inspect pods (`kubectl get pods -n wordpress`).
  - Check MySQL PVC free space via Grafana or `kubectl`.
  - Review Prometheus alerts for deployment availability.

---

## File Map

- `README.md` – this document.
- `manifests/wordpress/` – WordPress workload (deployments, PVCs, ingress, secrets, backups).
- `manifests/monitoring/` – monitoring stack (Prometheus, Grafana, Alertmanager, exporters, reloader, ingresses).
- `manifests/argocd/` – Argo CD applications (WordPress, monitoring).
- `manifests/.../*-sealedsecret.yaml` – cluster-specific sealed secrets (reseal if moving to another cluster).

Keep this README up to date whenever architecture or tooling changes. It’s the runbook for future debugging sessions. Happy monitoring!
