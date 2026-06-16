### Description

### Validation

- [ ] `helm dependency build charts/other/cardiac-uhn`
- [ ] `helm lint charts/other/cardiac-uhn`
- [ ] `helm template cardiac-uhn charts/other/cardiac-uhn --namespace cardiac-dev`

### Release Checklist

- [ ] Chart version in `Chart.yaml` was bumped when release behavior changed.
- [ ] `artifacthub.io/changes` was updated when release behavior changed.
- [ ] Values and README examples match the rendered chart behavior.
