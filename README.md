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
