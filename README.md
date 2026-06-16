# UHN Helm Chart

Helm charts for the UHN Cardiac Drive team.

## Charts

| Chart | Description |
| --- | --- |
| `cardiac-uhn` | App-template based chart for Cardiac UHN applications |

## Helm Repository

This repository is published through GitHub Pages:

```sh
helm repo add uhn https://uhn.github.io/helm-chart
helm repo update
helm search repo uhn
```

Install the `cardiac-uhn` chart:

```sh
helm install cardiac-uhn uhn/cardiac-uhn \
  --namespace cardiac-dev \
  --create-namespace
```

Chart releases are produced by `.github/workflows/release.yml` using Helm chart-releaser. The workflow publishes chart packages and `index.yaml` to the `gh-pages` branch whenever chart versions change on `main`.
