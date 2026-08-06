# Repository Guidelines

This repository publishes UHN Helm charts through GitHub Pages at `https://uhn.github.io/helm-chart`.

## Layout

- Application charts live under `charts/other/<chart-name>`.
- Keep chart dependencies declared in `Chart.yaml` and locked in `Chart.lock`.
- Do not commit downloaded chart archives under `charts/**/charts/*.tgz`; generate them with `helm dependency build` or `helm package --dependency-update`.

## Validation

Before pushing chart changes, run — for each chart you touched:

```sh
yamllint .
yamale -s .ci/ct/chart_schema.yaml charts/other/<chart>/Chart.yaml
helm dependency build charts/other/<chart>
helm lint charts/other/<chart>
helm template <chart> charts/other/<chart> --namespace cardiac-dev
mkdir -p /tmp/uhn-helm-chart-package/other
helm package charts/other/<chart> --dependency-update --destination /tmp/uhn-helm-chart-package/other
```

A chart may ship `examples/*.yaml`. Those are render fixtures, not just
documentation — CI renders the chart against every one of them, so keep them
working. Render locally the same way, adding `--api-versions
gateway.networking.k8s.io/v1/HTTPRoute` for charts that emit an `HTTPRoute`.

For repository publishing checks, verify:

```sh
helm repo index /tmp/uhn-helm-chart-package --url https://uhn.github.io/helm-chart/
```

## Release Discipline

- Bump the chart `version` in `Chart.yaml` for every released chart change.
- Update `artifacthub.io/changes` in `Chart.yaml` when the release behavior changes.
- Keep GitHub Actions and Helm dependencies pinned to explicit versions.
- Prefer small chart defaults and environment-specific overrides over adding team-specific branches to templates.
