# uhn-website

`uhn-website` deploys a **static website** — nginx serving content from a shared
volume — on top of the bjw-s `app-template` (`common`) chart. It generalizes the
pattern built for the heart-transplant dashboard
(`hf-transplant`, GitLab `uhn/infrastructure/k8s` → `helm-transplant-dash`).

Two properties define it:

- **The image is never rebuilt.** Stock `nginx` runs with its server block
  mounted from a ConfigMap this chart renders. Nothing about your site is baked
  into a container.
- **Content lives on a volume, not in the image.** Publishing a new build never
  rolls the pod. Changing the *nginx config* does — `common` hashes
  `configMaps.*.data` into a `checksum/configMaps` pod annotation automatically.

## Two ways to publish content

### 1. Push-to-deploy (the receiver sidecar)

Enable `controllers.main.containers.receiver`. It mounts the same volume as
nginx — but **read-write**, where nginx gets **read-only** — and turns an
authenticated upload into an atomic release swap:

```
POST /__deploy   (Basic auth, gzipped tar body)
  → extract into  <mountPath>/releases/<utc-ts>/
  → reject if no index.html at the tar root  (422)
  → atomically flip  <mountPath>/current  → releases/<ts>
  → prune to KEEP releases
```

nginx serves `root <mountPath>/current` and resolves that symlink per request,
so the flip is picked up live with no reload and no downtime.

```fish
cd web; and npm run build
tar -C out -czf - . | curl --fail -u deployer:'<pass>' \
  -H "Content-Type: application/octet-stream" --data-binary @- \
  https://<site-host>/__deploy
# → deployed 20260610T140231.356346000Z
```

Run the `tar` from the directory whose **root** holds `index.html` — a
wrong-directory tar ships an empty body and is rejected `422` with no release
left behind.

**Rollback** — repoint `current` at an older release from any read-write client
of the volume:

```sh
ln -sfn releases/<older-ts> <mountPath>/current
```

### 2. Write to the volume yourself

Leave the receiver disabled — a CI job, an NFS client, or a sync sidecar puts
files on the volume. Set `site.currentDir: ""` to serve the volume root
directly instead of a `current` symlink.

## Consumer contract

Set `global.nameOverride` (or `fullnameOverride`) to your site's name so
resources are named after the site, not `uhn-website`. Consumed from an ArgoCD
repo the same way as `cardiac-uhn`:

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: uhn-website
    repo: https://uhn.github.io/helm-chart
    version: 0.1.0
    releaseName: mysite
    namespace: cardiac-stage
    valuesFile: values.yaml
    # Force Gateway API v1. Without cluster capabilities, bjw-s common falls
    # back to v1alpha2; the gateway serves v1.
    apiVersions:
      - gateway.networking.k8s.io/v1/HTTPRoute
```

Three worked examples ship with the chart, and CI renders every one of them:

| File | Shape |
|---|---|
| `examples/values.nfs-receiver.yaml` | NFS export + push-to-deploy — the transplant-dash setup |
| `examples/values.pvc-eso.yaml` | The default `nfs-retain` claim + credential from Vault via ESO |
| `examples/values.existing-claim.yaml` | Pre-provisioned claim, no receiver, volume root served directly |

## The `site` block

| Value | Default | Meaning |
|---|---|---|
| `site.mountPath` | `/srv/site` | Where the content volume mounts, in **both** containers |
| `site.currentDir` | `current` | Subdirectory of `mountPath` that nginx serves. `""` serves `mountPath` itself |
| `site.index` | `index.html` | nginx `index` directive |
| `site.maxBodySize` | `250m` | nginx `client_max_body_size` for `/__deploy` |
| `site.deployAuthSecret` | `<fullname>-secret` | Secret holding keys `user` and `pass`. Rendered through `tpl` |
| `site.extraConfig` | `""` | Appended verbatim inside the server block, rendered through `tpl` |

## Storage

`persistence.site` defaults to a dynamically provisioned **`nfs-retain`** claim
— 1Gi, `ReadWriteMany`, `retain: true`. Just set `size` and go.

`nfs-retain` rather than the cluster's default StorageClass, deliberately:

| | `fastio-geo` (cluster default) | `nfs-retain` |
|---|---|---|
| Backing | vSphere CSI block | `nfs.csi.k8s.io` |
| Access modes | ReadWriteOnce | ReadWriteMany |
| Reclaim policy | Delete | **Retain** |
| Replicas possible | exactly 1, forever | as many as you like |

A published site is content you do not want to lose to a `kubectl delete pvc`,
and a static site is the easiest thing in the world to scale out — RWO would
foreclose that permanently.

Two guards with confusingly similar names protect the content, and they are
independent:

- `retain: true` — a Helm `resource-policy: keep` annotation. Stops
  `helm uninstall` from deleting the **PVC**.
- `nfs-retain`'s reclaim policy — stops the **PV and its data** from being
  deleted when the PVC goes away.

### Other shapes

Each replaces the default claim wholesale. Because `common`'s values schema
rejects PVC-only keys on an inline NFS volume (and vice versa), and Helm values
*merge* rather than replace, you must null out the keys you are dropping:

```yaml
persistence:
  site:
    type: nfs                              # hand-managed export, outside the CSI driver
    server: uhn-cvdmc.stg.ccm.sickkids.ca
    path: /ifs/H4H/uhn-cvdmc/mysite
    storageClass: null
    accessMode: null
    size: null
    retain: null
```

```yaml
persistence:
  site:
    existingClaim: mysite-content          # you provisioned it
    storageClass: null
    accessMode: null
    size: null
    retain: null
```

### Replicas and rollout

The controller defaults to one replica with `strategy: Recreate`, which is safe
on *any* backing store. On the default RWX claim you can raise `replicas` and
switch to `RollingUpdate` — every nginx sees the same volume, and the receiver's
symlink flip is visible to all of them at once. Do **not** do that on a
ReadWriteOnce claim: the new pod cannot attach the volume while the old one
holds it, and the rollout wedges.

Note that with more than one replica each pod carries its own receiver sidecar.
A given `/__deploy` request is served by exactly one of them, so that is safe —
but two *concurrent* deploys could interleave. Deploy serially.

### Write permission

`nfs-retain` mounts NFSv3 with `sec=sys`, so the server still authorizes writes
by uid/gid — dynamic provisioning does not make that go away. In practice
`csi-driver-nfs` creates each volume's subdirectory world-writable, so the
receiver writes fine with no `runAsUser` override. If a deploy comes back with a
permission error, that assumption did not hold on this share: set
`controllers.main.containers.receiver.securityContext.runAsUser`/`runAsGroup`
to the uid that owns it, exactly as the hand-managed-export path requires.

## Prerequisites when the receiver is enabled

1. **The auth Secret must exist** — keys `user` and `pass`. The receiver refuses
   to start without both, so a missing Secret fails the deploy closed rather
   than exposing an unauthenticated upload endpoint. **The chart defaults to
   neither option**: `site.deployAuthSecret` points at the ESO-produced
   `<fullname>-secret`, which does not exist until you enable `externalSecret`.
   So enabling the receiver means also doing one of these. Either let ESO
   materialize it (`externalSecret.enabled: true`) or create it out-of-band and
   point `site.deployAuthSecret` at it:

   ```sh
   kubectl -n <ns> create secret generic mysite-deploy-auth \
     --from-literal=user=deployer --from-literal=pass='<strong-pass>'
   ```

2. **Write access to the volume.** This is the one setting with no safe default:

   - **NFS** — `sec=sys` authorizes writes by uid/gid, so the receiver must run
     as the uid that owns the export. Set
     `controllers.main.containers.receiver.securityContext.runAsUser` and
     `runAsGroup`. The export must permit the cluster's nodes and must not
     `all_squash` (`root_squash` is fine). Confirm with storage.
   - **PVC** — set `defaultPodOptions.securityContext.fsGroup` instead.

3. **The first deploy bootstraps `current`.** Until the first upload the site
   returns 404 — but `/healthz` keeps the pod Ready and `/__deploy` reachable,
   so just push once.

## Receiver image

`ghcr.io/uhn/website` (multi-arch: `linux/amd64`, `linux/arm64`, org-`internal`
visibility). The binary is site-agnostic — it reads only `WEBROOT`, `KEEP`,
`MAX_BYTES`, `AUTH_USER` and `AUTH_PASS`. Pin `image.tag` to a released tag,
never `latest`.

Source and Dockerfile live in the
[HTDashboard](https://github.com/jduhamel/HTDashboard) repo under
`tools/receiver`, built by that repo's `make docker-push`, which still publishes
to `ghcr.io/uhn/ht-receiver`. `v0.5.0` was copied across index-digest-identical
(`sha256:3726092a…`), so the two tags are the same image today — but a future
`make docker-push` lands only on `ht-receiver`. Until that repo's `Makefile`
targets `ghcr.io/uhn/website` directly, new versions must be copied over:

```sh
docker buildx imagetools create \
  --tag ghcr.io/uhn/website:<tag> ghcr.io/uhn/ht-receiver:<tag>
```

Copy with `imagetools create` (or `crane`/`skopeo --all`), not
`docker pull && docker tag && docker push` — the latter flattens the multi-arch
index to the pushing host's architecture.
