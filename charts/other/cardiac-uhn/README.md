# cardiac-uhn

`cardiac-uhn` is a UHN wrapper around the bjw-s `app-template` (`common`) chart
that bakes in the Cardiac team's **cluster-native** deployment conventions:

- **Secrets:** External Secrets Operator. An `ExternalSecret` pulls from the
  namespace `SecretStore` (`cardiac-secret`, provisioned by
  `namespace-helm-chart`) into a Secret named `<app>-secret`, consumed via
  `envFrom`. **Stakater Reloader** rolls the pod when that Secret rotates.
- **Exposure:** Gateway API `HTTPRoute` to the shared `envoy` gateway in the
  `network` namespace.
- **Image pulls:** `github-uhn-registry` (GHCR).
- **Dynamic Vault leases:** optional `VaultDynamicSecret` generators for apps
  that need dynamic credentials materialized through ESO.
- **NATS topology:** optional NACK `Stream` and `Consumer` CRs so an app can
  declare its JetStream topology beside the workload that uses it.
- Reference implementation: `repos/cardiac-stage-argo.uhn.io/apps/cohort`.

> This replaces the pre-0.2.0 vault-agent / consul-template in-pod rendering.
> Composed secrets and rendered config files that consul-template's `files:`
> and `MONGO_URL` composition used to produce are now expressed with ESO's
> `externalSecret.template` (see below). Vault **PKI cert issuance** (NATS
> mTLS) has no ESO equivalent — use cert-manager if that is ever revived.

## Consumer contract

Set `global.nameOverride` (or `fullnameOverride`) to your app name so all
resources and the ESO Secret are named after the app, not `cardiac-uhn`.

```yaml
global:
  nameOverride: cohort
  fullnameOverride: cohort

controllers:
  main:
    containers:
      main:
        image: { repository: ghcr.io/uhn/cohort/cohort-server, tag: v0.15.0 }
        command: ["/usr/local/bin/cohort-server"]

service:
  main:
    ports: { http: { port: 10111, targetPort: 10111 } }

route:
  main:
    enabled: true
    hostnames: [cohort.stage.cardiac.uhn.ca]

externalSecret:
  data:                                   # discrete properties -> env keys
    - secretKey: COHORT_CLIENT_SECRET
      remoteRef: { key: cardiac/overlays/stage/app/cohort, property: client_secret }
  template:                               # composed secrets / rendered files
    engineVersion: v2
    data:
      MONGO_URL: "mongodb://{{ .mUser }}:{{ .mPass }}@{{ .mUrl }}"
```

- `externalSecret.data` — one entry per secret key (ESO `spec.data`).
- `externalSecret.template` — ESO `spec.target.template`; compose connection
  strings or emit whole config files. This is the ESO equivalent of the old
  consul-template `files:` / `MONGO_URL` machinery.
- Secret-less apps: set `externalSecret.enabled: false`. The container's
  `envFrom` is marked `optional`, so the pod still starts.

### Dynamic Vault leases

Use `vaultDynamicSecrets.items` when an app needs a Vault dynamic secret, then
reference the generated data from `externalSecret.dataFrom`.

```yaml
vaultDynamicSecrets:
  items:
    - name: cohort-mongo
      path: db/mongo.hpc4healthlocal/creds/cohort

externalSecret:
  dataFrom:
    - sourceRef:
        generatorRef:
          apiVersion: generators.external-secrets.io/v1alpha1
          kind: VaultDynamicSecret
          name: cohort-mongo
  template:
    engineVersion: v2
    data:
      MONGO_URL: "mongodb://{{ .username }}:{{ .password }}@mongo.example:27017"
```

### NACK JetStream topology

Use `nack.streams` for app-owned streams and consumers. The cluster-level NACK
operator supplies its NATS URL and credentials; this chart only renders the CRs.

```yaml
nack:
  enabled: true
  streams:
    - name: wearable
      spec:
        subjects: [stage.wearable.>]
        storage: file
      consumers:
        - name: wearable-data-consumer
          spec:
            filterSubject: stage.wearable.data
```

## Validate

```sh
helm dependency build charts/other/cardiac-uhn
helm lint charts/other/cardiac-uhn
helm template <app> charts/other/cardiac-uhn -n <namespace> -f your-values.yaml
```
