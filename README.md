# Deploy Kyte app to the Kynator orchestrator

A GitHub Action that deploys a [Kyte](https://github.com/kytelang/kyte) web app to a server running the
[Kynator](https://github.com/kytelang/kynator) orchestrator. Point it at your orchestrator and it builds
your app (or takes a prebuilt binary), uploads the binary as a content-addressed blob to `artifactd`, binds
it to `app/version`, and writes the workload manifest into the config store. `kynatord` on the server then
pulls the binary by hash, verifies it, and reconciles the replicas. No SSH, no container registry, no
database in the loop.

## Quick start

In your Kyte app repo, add `.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    tags: ["v*"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Make `kyte` available on PATH (build the toolchain, or install a prebuilt).
      - uses: kytelang/setup-kyte@v1          # if/when published; otherwise build it in a step

      - uses: kytelang/kynator-deploy-action@v1
        with:
          orch-url: ${{ vars.ORCH_URL }}       # e.g. https://orch.example.com:8135
          token:    ${{ secrets.ORCH_TOKEN }}  # the deploy bearer token
          app:      webapp
          app-src:  src/main.ky
          target:   linux-x86_64               # the server's OS/arch
          replicas: "3"
          port:     "8080"
```

Set `ORCH_URL` (a repository **variable**) and `ORCH_TOKEN` (a repository **secret**) under
Settings -> Secrets and variables -> Actions.

Already build the binary yourself? Skip `app-src`/`target` and pass `binary:` instead:

```yaml
      - uses: kytelang/kynator-deploy-action@v1
        with:
          orch-url: ${{ vars.ORCH_URL }}
          token:    ${{ secrets.ORCH_TOKEN }}
          app:      webapp
          binary:   ${{ runner.temp }}/webapp
```

## Reaching the orchestrator

The runner needs network access to `artifactd`'s HTTP endpoint. Pick by your topology:

| orchestrator location | how to reach it | `orch-url` |
|---|---|---|
| public / behind an HTTPS ingress | GitHub-hosted runner pushes directly | `https://orch.example.com:8135` |
| private network / VPC | a **self-hosted runner** on that network | `http://10.0.0.5:8135` |
| locked-down host | a tunnel (Tailscale/WireGuard/SSH `-L`) step, then push over it | `http://127.0.0.1:8135` |

The action is identical in all three; only `orch-url` and the runner's network path change. Always use
`https` and a strong token for a publicly reachable orchestrator: the endpoint accepts a binary that
`kynatord` will execute.

## Getting a deploy token

There is no token to fetch and no login step. `artifactd` guards its routes with a single **shared
secret**: it compares the request's `Authorization: Bearer <token>` header (in constant time) against one
value it reads at startup from the `KYTE_ARTIFACT_TOKEN` environment variable. This is not OAuth, OIDC, or
a JWT, and there is no `/token` endpoint. You choose the secret once, set the same string on the server and
in CI, and that is the whole scheme.

So "getting the token" is three steps.

**1. Generate a strong random secret** (once). Any high-entropy string works; do not hand-pick a
memorable one. For example:

```sh
openssl rand -hex 32
# or
head -c 32 /dev/urandom | base64
```

**2. Give it to the server.** Set `KYTE_ARTIFACT_TOKEN` in the environment of the `artifactd` process (the
same value on every orchestrator host). With the systemd unit from the Kynator install docs:

```ini
# /etc/systemd/system/artifactd.service  (drop-in or the unit's [Service] section)
[Service]
Environment=KYTE_ARTIFACT_TOKEN=<the value from step 1>
```

```sh
sudo systemctl daemon-reload && sudo systemctl restart artifactd
```

On start, `artifactd` logs `auth=on` when a token is set. If the variable is empty it logs `auth=OFF (dev)`
and accepts every request: that is for local development only, never for a reachable host (the endpoint
runs uploaded binaries, so no auth means remote code execution).

**3. Give the same value to CI.** In the app repo, add it as a repository **secret** (not a variable, so it
is masked in logs) under Settings -> Secrets and variables -> Actions:

- `ORCH_TOKEN` (secret) = the value from step 1
- `ORCH_URL` (variable) = your `orch-url`, for example `https://orch.example.com:8135`

The workflow then forwards the secret to the action, which sends it only as a header and never prints it:

```yaml
      - uses: kytelang/kynator-deploy-action@v1
        with:
          orch-url: ${{ vars.ORCH_URL }}
          token:    ${{ secrets.ORCH_TOKEN }}
          app:      webapp
          app-src:  src/main.ky
```

**Verifying it works.** A correct token returns 2xx from every step. A wrong or missing one fails the
upload step with `deploy token rejected (401)`. You can sanity-check the server directly:

```sh
curl -fsS -H "Authorization: Bearer $KYTE_ARTIFACT_TOKEN" https://orch.example.com:8135/artifacts/0/exists
# 404 = reached and authorised (that hash simply is not present); 401 = token wrong
```

**Rotation.** The token is a static secret with no expiry, so rotate it manually and in lock step: set the
new `KYTE_ARTIFACT_TOKEN` on the server, restart `artifactd`, and update the `ORCH_TOKEN` secret in the
same change. Because auth is a single shared secret, the server only ever trusts one token at a time; there
is no per-deploy or short-lived token today. Treat the secret like an SSH deploy key: anyone holding it can
ship a binary the orchestrator will execute, so scope who can read the CI secret accordingly.

## Inputs

| Input | Required | Default | Meaning |
|-------|----------|---------|---------|
| `orch-url` | yes | | artifactd base URL |
| `token` | yes | | deploy bearer token (pass a secret) |
| `app` | yes | | workload name (`workloads/<app>`, `apps/<app>`) |
| `version` | no | git ref name | version label to bind (immutable) |
| `app-src` | no* | | entry `.ky` to build (*required unless `binary` is set) |
| `binary` | no* | | prebuilt binary to deploy (*skips the build) |
| `target` | no | `linux-x86_64` | build target triple for the server |
| `kyte` | no | `kyte` | path to the `kyte` compiler |
| `replicas` | no | `2` | desired replica count |
| `port` | no | `8080` | public listen port (`basePort`) |
| `args` | no | `""` | extra argv per replica, space-separated |
| `spec-template` | no | `""` | JSON spec with `sha256:__ARTIFACT__` to substitute |

## Outputs

- `sha`: the sha256 digest of the deployed binary.
- `revision`: the config-store revision at which `workloads/<app>` was written.

## The manifest is the single source of truth (recommended)

Commit ONE manifest that both runs AND configures your app, and pass it as `spec-template`. It carries
the orchestrator settings, the binary by hash (a `sha256:__ARTIFACT__` placeholder the action fills in),
AND the app's own configuration under `config:` (env-file `KEY=VALUE` lines). kynatord injects each config
entry into every replica as an environment variable, and the app reads it back from the environment, so
there is no second config file. Example `deploy/workload.yaml`:

```yaml
apiVersion: kyte/v1
kind: App
metadata:
  name: webapp
workload:
  artifact: sha256:__ARTIFACT__     # the action substitutes the real digest
  restartPolicy: always
replicas:
  min: 2
  max: 6
network:
  expose: public
  portBase: 8080
  portFlag: --port
config:
  - LOG_LEVEL=info
  - DB_URL=novadb://127.0.0.1:8135?db=shop
```

```yaml
      - uses: kytelang/kynator-deploy-action@v1
        with:
          orch-url:      ${{ vars.ORCH_URL }}
          token:         ${{ secrets.ORCH_TOKEN }}
          app:           webapp
          app-src:       src/main.ky
          spec-template: deploy/workload.yaml
```

Without `spec-template` the action generates a minimal JSON spec from the inputs instead; a JSON spec
uses `"config": [ { "key": "...", "value": "..." } ]` for the same effect.

## What it calls

Against `artifactd`, every request bearing `Authorization: Bearer <token>`:

1. `GET /artifacts/<sha>/exists`, then `PUT /artifacts/<sha>` if absent (verified, idempotent).
2. `PUT /apps/<app>/<version>` with body `sha256:<hex>` (registry bind, immutable).
3. `POST /cfg/put` with header `x-cfg-key: workloads/<app>` and the manifest as the body (desired state).
4. `GET /cfg/get` to read it back and confirm the digest.

Any non-2xx fails the step. The token is only ever sent as a header, never printed.

## Releasing (maintainers)

This is a composite action; `action.yml` at the repo root is what the Marketplace lists. Cut a semver
tag and a moving major tag so consumers can pin `@v1`:

```sh
git tag -a v1.0.0 -m "v1.0.0" && git push origin v1.0.0
git tag -f v1 v1.0.0 && git push -f origin v1
```

Then publish from the repo's Releases page ("Publish this Action to the Marketplace").

## Licence

MIT. See [LICENSE](LICENSE).
