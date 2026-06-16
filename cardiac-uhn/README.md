# cardiac-uhn

`cardiac-uhn` is a UHN wrapper around the bjw-s `app-template` chart with DHP application defaults:

- image pull secret: `github-uhn-registry`
- Vault address: `https://vault.uhndhp.io`
- Vault Kubernetes service account: `vault-auth`
- Vault agent and Consul Template startup flow from `external-example/generic-dhp-application-helm`

Set the application image and startup command before deploying:

```yaml
controllers:
  main:
    containers:
      main:
        image:
          repository: ghcr.io/uhn/cardiac-uhn
          tag: "0.1.0"

application:
  command: exec /app/start
```
