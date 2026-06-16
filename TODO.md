# TODO

## Near Term

- Add `artifacthub.io/changes` annotations to `charts/other/cardiac-uhn/Chart.yaml` before the next chart version bump.
- Decide whether `vault-auth`, `vault-config`, and `vault-shared` should use Cardiac Drive names.
- Confirm Cardiac Drive Vault paths for NATS, Mongo, and application secrets.
- Add a sample values file for a real Cardiac Drive application.
- Add helm-unittest coverage for rendered ServiceAccount, pull secret, Vault address, and init containers.

## Later

- Add k3d install testing if this repository grows beyond one chart.
- Consider OCI publishing to `ghcr.io/UHN/helm` if consumers want OCI charts in addition to GitHub Pages.
- Add generated docs if chart values expand enough that README tables become useful.
