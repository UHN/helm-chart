# Repository Guidelines

This repository publishes UHN Helm charts through GitHub Pages at `https://uhn.github.io/helm-chart`.

## Layout

- Application charts live under `charts/other/<chart-name>`.
- Keep chart dependencies declared in `Chart.yaml` and locked in `Chart.lock`.
- Do not commit downloaded chart archives under `charts/**/charts/*.tgz`; generate them with `helm dependency build` or `helm package --dependency-update`.

## Validation

Before pushing chart changes, run:

```sh
helm dependency build charts/other/cardiac-uhn
helm lint charts/other/cardiac-uhn
helm template cardiac-uhn charts/other/cardiac-uhn --namespace cardiac-dev
mkdir -p /tmp/uhn-helm-chart-package/other
helm package charts/other/cardiac-uhn --dependency-update --destination /tmp/uhn-helm-chart-package/other
```

For repository publishing checks, verify:

```sh
helm repo index /tmp/uhn-helm-chart-package --url https://uhn.github.io/helm-chart/
```

## Release Discipline

- Bump the chart `version` in `Chart.yaml` for every released chart change.
- Update `artifacthub.io/changes` in `Chart.yaml` when the release behavior changes.
- Keep GitHub Actions and Helm dependencies pinned to explicit versions.
- Prefer small chart defaults and environment-specific overrides over adding team-specific branches to templates.
