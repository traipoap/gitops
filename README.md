# GitOps — Flux CD

> All cluster state lives in Git. Flux CD reconciles it continuously.

Part of [traipoap](https://github.com/traipoap) — the GitOps repo for a K3s cluster. Infrastructure as Code (Terraform + Ansible), CI/CD, and app code live in the main repo.

---

## How It Works

```
Git (main) ──► Flux CD ──► K3s Cluster
```

1. **Flux watches this repo** (every 1 minute).
2. **ArtifactGenerator** packages `infrastructure/` and `apps/` into artifacts.
3. **Kustomizations** apply them in order:

```
flux-system (bootstrap)
  └► infra-controllers   (cert-manager, external-secrets, kyverno, flux-web, namespaces)
       └► infra-configs  (issuers, external secrets, quickwit index job)
            └► infra-services  (vector, quickwit, nfs)
                 └► apps  (frontend + backend deployments)
  └► image-automation  (auto-updates image tags → commit → Flux deploys)
```

4. **Image Automation**: when a new image lands on GHCR, Flux detects it, updates the tag in `apps/staging/kustomization.yaml`, and commits as `fluxcdbot`. No manual `kubectl set image`.

---

## Repository Layout

```
├── apps/
│   ├── base/                  # Shared manifests (deployments, services, routing)
│   ├── staging/               # Staging overlay (image tags, patches)
│   └── production/            # Production overlay
├── clusters/
│   ├── staging/
│   │   ├── flux-system/       # Flux bootstrap (gotk-components + gotk-sync)
│   │   ├── image-automation/  # ImageRepository + ImagePolicy + ImageUpdateAutomation
│   │   ├── apps.yaml
│   │   ├── infrastructure.yaml
│   │   └── artifacts.yaml
│   └── production/            # Same structure, different path
├── infrastructure/
│   ├── controllers/           # Helm-managed: cert-manager, external-secrets, kyverno, flux-web
│   ├── configs/               # Plain YAML: issuers, external secrets, kyverno, gateway, TLS
│   └── services/              # Secret-dependent: vector, quickwit, nfs
└── scripts/validate.sh        # flux-schema validation
```

### What's Where

| Layer | Path | Contains |
|-------|------|----------|
| Base controllers | `infrastructure/controllers/` | cert-manager, external-secrets, kyverno (Helm), flux-web, namespaces, Istio addons (Prometheus, Grafana, Kiali) |
| Configs | `infrastructure/configs/` | ClusterIssuer, ExternalSecrets, Kyverno policy, Quickwit index job, Gateway, TLS cert, Flux Web HTTPRoute |
| Services | `infrastructure/services/` | Vector, Quickwit, NFS provisioner (HelmReleases with `valuesFrom` secrets) |
| App | `apps/base/` | Deployments, Services, HTTPRoute, ContainerLimits, ExternalSecrets for app |
| Per-env | `clusters/<env>/` | Flux bootstrap + Kustomizations + Image Automation |

---

## Flux Resources

### Kustomizations (apply order)

| # | Name | Applies | Depends On |
|---|------|---------|------------|
| 1 | `flux-system` | `clusters/staging` (or `production`) | — |
| 2 | `infra-controllers` | `infrastructure/controllers` | flux-system |
| 3 | `infra-configs` | `infrastructure/configs` | infra-controllers |
| 4 | `infra-services` | `infrastructure/services` | infra-configs |
| 5 | `apps` | `apps/staging` (or `production`) | infra-services |
| 6 | `image-automation` | `clusters/<env>/image-automation` | — |

### HelmReleases

| Release | Namespace | What It Does |
|---------|-----------|--------------|
| `cert-manager` | `security` | TLS cert automation (self-signed CA) |
| `external-secrets` | `security` | Syncs AWS SecretsManager → K8s Secrets |
| `kyverno` | `security` | Policy engine (enforces resource requests) |
| `flux-web` | `flux-system` | GitOps dashboard (server-only, no admin creds) |
| `vector` | `logging` | Ships k8s logs + syslog → Quickwit |
| `quickwit` | `logging` | Log store (S3-backed, via ExternalSecret) |
| `nfs-subdir-external-provisioner` | `networking` | NFS StorageClass (RWX PVCs) |

### External Secrets (AWS SecretsManager → K8s)

| Secret | Used For |
|--------|----------|
| `jwt-secret` | Backend JWT signing |
| `github-registry` | GHCR image pull (dockerconfigjson) |
| `nfs-provisioner-secret-values` | NFS server endpoint (Helm values) |
| `quickwit-s3-secret-values` | Quickwit S3 storage creds (Helm values) |

> Nothing sensitive is stored in Git. All credentials come from AWS SecretsManager at runtime.

### Image Automation

| Image | Policy | Updates |
|-------|--------|---------|
| `ghcr.io/traipoap/backend` | semver `>=0.0.0` | `apps/staging/kustomization.yaml` |
| `ghcr.io/traipoap/frontend` | semver `>=0.0.0` | `apps/staging/kustomization.yaml` |

Strategy: **Setters** — Flux rewrites the `newTag` value in the kustomization, commits, and the next reconcile deploys the new image.

### Security

| Tool | What It Does |
|------|--------------|
| Kyverno (`require-pod-resources`) | Rejects Pods without CPU/memory requests |
| LimitRange (`container-limits.yaml`) | Default CPU limits per namespace |
| Container securityContext | `runAsNonRoot`, `drop: ALL` capabilities, no privilege escalation |

---

## Application

```
frontend (Astro, :4321)  ←  Gateway (:443 TLS)  ←  Cloudflare
backend (Go API, :8080)  ←  HTTPRoute
```

- **TLS**: Istio Gateway terminates TLS. Cert issued by cert-manager (self-signed CA).
- **Routing**: `/*` → frontend, `/api/*` → backend (via HTTPRoute).
- **Storage**: NFS PVCs for backend exports + data (RWX).

---

## Observability

| Tool | Namespace | What |
|------|-----------|------|
| Prometheus | `istio-system` | Metrics (Istio addon) |
| Grafana | `istio-system` | Dashboards + Quickwit log datasource |
| Kiali | `istio-system` | Service mesh topology |
| Quickwit | `logging` | Log search (S3-backed) |
| Vector | `logging` | Log shipper (k8s + syslog → Quickwit) |
| Flux Web | `flux-system` | GitOps dashboard |

**Log flow:** `Pod logs → Vector → Quickwit → Grafana`

---

## Quick Start

```bash
# 1. Bootstrap Flux (per cluster)
  flux bootstrap github \
    --components-extra=image-reflector-controller,image-automation-controller,source-watcher \
    --owner=traipoap --repository=gitops --branch=main \
    --path=./clusters/staging --read-write-key --personal
    
# 2. Check status
flux get all -A
kubectl get pods -A

# 3. Validate manifests locally
./scripts/validate.sh          # YAML + kustomize
./scripts/validate.sh -H       # + Helm rendering
```

---

## Environments

| Env | Bootstrap Path | App Overlay | Image Automation |
|-----|---------------|-------------|------------------|
| staging | `clusters/staging/` | `apps/staging/` | ✅ (auto-updates) |
| production | `clusters/production/` | `apps/production/` | ✅ (auto-updates) |

### Add a New Environment

```bash
cp -r clusters/staging clusters/<new-env>   # update paths in gotk-sync.yaml
cp -r apps/staging apps/<new-env>           # add env-specific patches
# update artifacts.yaml to include apps/<new-env>/**
flux bootstrap github ... --path=./clusters/<new-env>
```

---

## Roadmap

- [ ] Multi-env promotion (staging → production) with gate
- [ ] Alerting rules + SLOs (Prometheus Alertmanager)
- [ ] NetworkPolicies (default-deny)
- [ ] Velero backup + restore runbook
- [ ] Canary deployments (Flagger or Istio VirtualService)
- [ ] cosign image signing + Flux verification
