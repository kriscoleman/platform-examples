# Vaultwarden - kURL Platform Example

A Replicated platform example for distributing [Vaultwarden](https://github.com/dani-garcia/vaultwarden), a lightweight Bitwarden-compatible password manager server.

This example demonstrates:

- Helm chart packaging for Replicated distribution
- KOTS Admin Console integration with Config screen
- kURL installer specification for bare-metal/VM installs
- Air-gap image rewriting via `optionalValues`
- Preflight checks and support bundle collection

## Quick Start

### Prerequisites

- [Replicated CLI](https://docs.replicated.com/reference/replicated-cli-installing)
- Helm 3.x
- `REPLICATED_APP` environment variable set to your app slug

### Create a Release

```bash
make release
```

This will:
1. Clean old chart archives
2. Update Helm chart dependencies (pulls the Replicated SDK subchart)
3. Package the vaultwarden Helm chart
4. Sync the chart version into the KOTS HelmChart CR
5. Create a Replicated release from `manifests/` and promote to Unstable

### Other Commands

```bash
make lint       # Lint Helm charts
make template   # Dry-run template rendering
make validate   # Check KOTS Config <-> HelmChart CR contract
make clean      # Remove .tgz archives
make help       # Show all targets
```

## Architecture

```
vaultwarden/
  charts/
    vaultwarden/          # Helm chart
      Chart.yaml          # Chart metadata + Replicated SDK dependency
      values.yaml         # Default values
      templates/          # Kubernetes manifests
  manifests/              # KOTS release directory (yaml-dir pattern)
    kots-app.yaml         # KOTS Application spec
    kots-config.yaml      # KOTS Config screen
    vaultwarden-chart.yaml # HelmChart CR (maps Config -> Helm values)
    kots-preflight.yaml   # Preflight checks
    kots-support-bundle.yaml # Support bundle spec
    kurl-installer.yaml   # kURL installer specification
    k8s-app.yaml          # Kubernetes Application CRD
```

## Configuration

The KOTS Config screen provides:

| Group | Items |
|-------|-------|
| Application Settings | Domain, signups toggle, admin token, log level |
| Database | SQLite (embedded) or external PostgreSQL |
| Storage | Volume size and storage class |
| Ingress | Enable/disable, class, hostname, TLS |
