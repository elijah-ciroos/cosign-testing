# cosign-testing

Test repo for signing and verifying container images with [cosign](https://github.com/sigstore/cosign).
CI builds the image in `Dockerfile`, pushes it to GHCR, signs it with a cosign
key pair, and verifies the signature (see `.github/workflows/test-sign-verify.yaml`).

## Testing locally against a zot registry

You can reproduce the build/push/sign/verify flow locally using
[zot](https://zotregistry.dev/) as a throwaway OCI registry.

### 1. Start zot

Use a mounted config so you can enable extensions (search, UI) and a mounted
data dir so images survive a container restart:

```bash
mkdir -p ~/zot-registry/data
cat > ~/zot-registry/config.json <<'EOF'
{
  "storage": { "rootDirectory": "/var/lib/registry" },
  "http": { "address": "0.0.0.0", "port": "5000", "compat": ["docker2s2"] },
  "log": { "level": "debug" },
  "extensions": {
    "search": { "enable": true },
    "ui": { "enable": true },
    "mgmt": { "enable": true },
    "trust": { "enable": true, "cosign": true }
  }
}
EOF

docker run -d -p 5000:5000 \
  -v ~/zot-registry/config.json:/etc/zot/config.json \
  -v ~/zot-registry/data:/var/lib/registry \
  --name zot \
  ghcr.io/project-zot/zot-linux-amd64:latest
```

The `trust` extension is what would let zot itself verify and report on
cosign signatures (in addition to us shelling out to `cosign verify`).
**As of zot v2.1.20 this still doesn't work for cosign v3's default output.**
zot v2.1.17 stopped crashing when indexing the new Sigstore bundle format,
and v2.1.19 fixed a real digest-binding bug in trust verification — but
neither taught the `trust`/`imagetrust` extension to recognize the new
bundle format in the first place. It's still only wired up for the old
annotation-based "simple signing" layout. Confirmed by hand against
`ghcr.io/project-zot/zot-linux-amd64:latest` (v2.1.20) on 2026-08-05:
`cosign verify` succeeds, but the GraphQL `IsTrusted` field stays `false`
and `Author` stays empty even after uploading the matching public key.
Tracked upstream as
[project-zot/zot#4299](https://github.com/project-zot/zot/issues/4299)
(a maintainer suggested v2.1.19 as a fix; it isn't — see step 7 below for
how to re-check once it's actually resolved).

Docker treats `localhost`/`127.0.0.0/8` registries as insecure by default, so
pushes over plain HTTP to `localhost:5000` work with no extra daemon config.

Open **http://localhost:5000/** in a browser for the zot web UI — it lists
repositories and tags.

### 2. Build and push the test image

```bash
docker build -t localhost:5000/cosign-testing:local .
docker push localhost:5000/cosign-testing:local
```

Note the digest printed in the push output (e.g. `sha256:a513ef...`).

### 3. Generate a local cosign key pair

The `cosign.pub` checked into this repo corresponds to the private key stored
in the `COSIGN_PRIVATE_KEY` GitHub Actions secret — you won't have the
matching private key locally. Generate your own pair for local testing:

```bash
COSIGN_PASSWORD="" cosign generate-key-pair
```

This writes `cosign.key` and `cosign.pub` to the current directory (don't
overwrite the repo's `cosign.pub` with this — use a scratch directory or
rename the output).

### 4. Create a signing config with the transparency log disabled

By default, cosign uploads every signature to the public Sigstore
transparency log (Rekor) — a real, permanent, public record — even for
plain `--key` signing. For local test signing that's not something you want.
Generate a signing config with Rekor stripped out (keeps Fulcio/OIDC/TSA
defaults, only removes the Rekor service):

```bash
cosign signing-config create --with-default-services --no-default-rekor \
  --out no-tlog-signing-config.json
```

### 5. Sign the image

```bash
IMAGE="localhost:5000/cosign-testing:local@sha256:<digest>"
COSIGN_PASSWORD="" cosign sign --yes \
  --signing-config no-tlog-signing-config.json \
  --key cosign.key "$IMAGE"
```

This uses cosign's default (new) bundle format — no compatibility flags
needed — and produces no transparency-log entry.

### 6. Verify the signature

Since there's no transparency-log entry, `cosign verify` needs
`--insecure-ignore-tlog=true` (safe here — this is local test signing, not a
publicly-distributed artifact):

```bash
cosign verify --insecure-ignore-tlog=true --key cosign.pub "$IMAGE"
```

### 7. Check whether zot's `trust` extension agrees (currently: no)

Upload the public key — before or after signing, order doesn't matter, zot
re-evaluates on each query — so zot's `trust` extension can independently
verify the signature:

```bash
curl --data-binary @cosign.pub "http://localhost:5000/v2/_zot/ext/cosign"
```

Then query the GraphQL search endpoint for the image's signature status:

```bash
curl -s "http://localhost:5000/v2/_zot/ext/search" \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ Image(image: \"cosign-testing:local\") { Digest IsSigned SignatureInfo { Tool IsTrusted Author } } }"}' \
  | jq .
```

Today this comes back `IsSigned: true` but `SignatureInfo[].IsTrusted: false`
and `Author: ""`, regardless of whether the right key was uploaded — see the
note in step 1. `cosign verify` in step 6 is the real check for now; treat
this step as a regression probe to rerun after a future zot upgrade.

### Cleanup

```bash
docker rm -f zot
rm -rf ~/zot-registry
```

## Testing Kyverno image-verification policies against zot

Investigated whether [Kyverno](https://kyverno.io/) can block/flag pods
running images that aren't signed, reusing the zot setup above as the
in-cluster registry for a local [kind](https://kind.sigs.k8s.io/) cluster.
Two separate policy mechanisms exist in Kyverno; neither fully worked, for
two independent reasons. Confirmed 2026-08-05 against Kyverno chart 3.8.2
(app v1.18.2) and cosign v3.1.2.

**Registry reachability from kind needs the zot container joined to the
`kind` Docker network**, in addition to the host port mapping — kubelet's
image pull and Kyverno's own registry client both run as processes inside
the cluster's own network namespace, where `localhost` means the pod itself,
not the host:

```bash
docker network connect kind zot
```

Reference the image in-cluster as `zot:5000/...` (zot's real listen port),
not the host-mapped `localhost:5000/...` used for `docker push`/`cosign sign`.
This doesn't survive `docker rm -f zot` — reconnect after recreating it.

**Plain-HTTP zot needs explicit opt-in, twice, at two different layers:**

- containerd on the kind node defaults to HTTPS for any registry hostname
  that isn't literally `localhost`/`127.0.0.0/8`/`*.local`, and will fail
  pulling the pod's own image with `http: server gave HTTP response to
  HTTPS client` unless told otherwise via a per-registry mirror config:
  ```bash
  docker exec kyverno-test-control-plane mkdir -p /etc/containerd/certs.d/zot:5000
  docker exec -i kyverno-test-control-plane tee /etc/containerd/certs.d/zot:5000/hosts.toml <<'EOF'
  server = "http://zot:5000"
  [host."http://zot:5000"]
    capabilities = ["pull", "resolve"]
  EOF
  docker exec -i kyverno-test-control-plane tee -a /etc/containerd/config.toml <<'EOF'

  [plugins."io.containerd.grpc.v1.cri".registry]
    config_path = "/etc/containerd/certs.d"
  EOF
  docker exec kyverno-test-control-plane systemctl restart containerd
  ```
- Kyverno's own (legacy) registry client needs the same opt-in separately,
  via a Helm value: `--set features.registryClient.allowInsecure=true`.

**Finding: the legacy `ClusterPolicy.verifyImages` mechanism can never pass,
because cosign v3 no longer writes the signature format it looks for.**
`ClusterPolicy`'s `verifyImages`/`keys` attestor (the long-standing Kyverno
mechanism) only knows how to find signatures via the old
`sha256-<digest>.sig` tag convention. cosign v3.1.2 writes exclusively via
the OCI 1.1 Referrers API instead — confirmed by querying
`/v2/<repo>/referrers/<digest>` directly and finding real sigstore bundle
entries there, with zero matching `.sig` tags anywhere in the same
repository. Tried both `--registry-referrers-mode=legacy` and
`COSIGN_DOCKER_MEDIA_TYPES=1` on `cosign sign` to force the legacy write path;
neither changed where the signature landed. Every `cosign verify` against
this same image succeeds fine (it's Referrers-aware) — this is specifically
a `ClusterPolicy.verifyImages` limitation. This is the same category of gap
as the zot trust-extension issue above: tooling built against cosign's old
tag convention hasn't caught up to cosign v3's new default. No upstream
Kyverno issue filed yet for this one.

**Newer `ImageValidatingPolicy` (CEL-based) is unconfirmed, not disproven.**
Kyverno also ships a second, newer policy CRD
(`imagevalidatingpolicies.policies.kyverno.io`) with a much richer,
cosign-verify-flag-shaped attestor schema (`cosign.key`, `cosign.keyless`,
`cosign.tuf`) that looks far more likely to understand Referrers-based
signatures — but its `verifyImageSignatures()` CEL function appears to use
its own registry client, separate from the one `allowInsecure` fixes:
```
policy verify-cosign-signature/evaluation error: failed to evaluate policy:
Get "https://zot:5000/v2/": http: server gave HTTP response to HTTPS client
```
Real production registries are HTTPS, so this may simply never matter in
practice — but it meant we couldn't get a real answer locally. Routing
around it via GHCR (real HTTPS, no local-registry transport issues) hit a
different open question instead: none of the CI-pushed images in
`ghcr.io/elijah-ciroos/cosign-testing` have *any* signature artifact visible,
via either convention — querying GHCR's Referrers API for the latest tag's
digest returns `MANIFEST_UNKNOWN`. Unknown whether that's a GHCR limitation,
an artifact of how the CI signing step runs, or something else. Not
investigated further.

**Net result:** proved Kyverno *can* mechanically intercept and evaluate
pods against a local registry (transport-layer problems are all solved and
documented above), but couldn't get a real pass/fail signal on either policy
mechanism against a cosign-v3-signed image in this local setup. Whether
Kyverno can enforce cosign v3 signatures at all remains an open question —
next step would be a registry with working HTTPS *and* confirmed OCI 1.1
Referrers support (self-signed zot, or debugging the GHCR gap above).

### Cleanup

```bash
kind delete cluster --name kyverno-test
docker network disconnect kind zot
```
