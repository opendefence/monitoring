# OpenDefence monitoring

Prometheus, Grafana, Loki, and Linkerd Viz as a Zarf package named `monitoring`. Deploy installs the [base](https://github.com/opendefence/base) package first. Monitoring uses External Secrets, `ClusterIssuer/public-issuer`, Traefik, and Linkerd from that package.

This repo includes base's kind cluster and registry tasks.

## Development usage

Install Docker or Podman, the POSIX utilities, and the tools in `mise.toml`. Ports 80 and 443 must be free. Set `BASE_VERSION` to a published base package, or set `BASE_PACKAGE` to an archive or another OCI reference.

```sh
mise install
task up BASE_VERSION=<published-base-version>
task status
task down
task reset
```

- `up`, also called `dev`, starts the registries and the kind cluster. It deploys base with `BASE_ISSUER=ca` unless you set `BASE_ISSUER`. Then it deploys this checkout.
- `down` deletes the selected cluster. Registry containers and their data stay.
- `reset` also removes owned registry containers and their anonymous volumes. **Local images and cached data are deleted.**
- Installing base again upgrades its Helm releases.

`CLUSTER_NAME`, `REGISTRY_PORT`, and the other cluster and registry settings are documented in opendefence/base.

## Package

`zarf.yaml` defines the package. Components deploy in this order: namespace, Prometheus Operator CRDs and operator, Grafana Operator, kube-state-metrics, Prometheus and Alertmanager, Loki, Alloy, Grafana, Linkerd Viz, then dashboards. Chart versions and release names are in `zarf.yaml`. Loki values are in `helm-values/loki.yaml`. Linkerd Viz reads the Prometheus from this package and does not start its own Prometheus.

External Secrets generates Grafana's admin password. Grafana's certificate uses `ClusterIssuer/public-issuer` and `grafana.host`. The default host is `grafana.localhost`.

```sh
task build:deploy
task build:lint
task build:inspect
task build:find-images
task build:package ARCH=amd64 BUILD_DIR=.build
```

`build:deploy` deploys in connected mode and does not prompt. Dashboards are a default component, so that command deploys them. `DASHBOARDS=false` adds `--components=-dashboards` and skips them.

```sh
task up BASE_VERSION=<published-base-version> DASHBOARDS=false
zarf package deploy oci://ghcr.io/opendefence/monitoring:<version> --connected --confirm --components=-dashboards
```

Set the host with `--set-values grafana.host=grafana.example.com` or with a values file. `MONITORING_VALUES_FILE` passes that file to `monitoring:install`.

Connected deploys do not push images. The package does not list container images. Run `build:find-images` and add the printed images under `images:` before you build an offline package.

```sh
task monitoring:install BASE_VERSION=<published-base-version> MONITORING_VERSION=<published-monitoring-version>
task monitoring:remove
```

`monitoring:install` deploys base, then `oci://ghcr.io/opendefence/monitoring:<MONITORING_VERSION>`. Set `BASE_PACKAGE` or `MONITORING_PACKAGE` to use a local archive. `monitoring:remove` removes monitoring and leaves base installed. It forwards `BASE_ISSUER` and `BASE_VALUES_FILE` to base. `up` defaults the issuer to `ca`.

| Setting                  | Default                                                                  |
| ------------------------ | ------------------------------------------------------------------------ |
| `BASE_VERSION`           | empty, required unless `BASE_PACKAGE` is set                             |
| `BASE_PACKAGE`           | `oci://ghcr.io/opendefence/base:<BASE_VERSION>`                          |
| `BASE_ISSUER`            | empty on `monitoring:install`. Package default is `acme`. `up` sets `ca` |
| `BASE_VALUES_FILE`       | empty                                                                    |
| `MONITORING_VERSION`     | empty, required unless `MONITORING_PACKAGE` is set                       |
| `MONITORING_PACKAGE`     | `oci://ghcr.io/opendefence/monitoring:<MONITORING_VERSION>`              |
| `MONITORING_VALUES_FILE` | empty                                                                    |
| `DASHBOARDS`             | `true`                                                                   |
| `KUBE_CONTEXT`           | `kind-<CLUSTER_NAME>`                                                    |

Zarf has no kube-context flag. Deploy, remove, and status exit unless the current context is `KUBE_CONTEXT`.

After `taskfiles/monitoring.yaml` is on GitHub, another repo can include that URL. The file includes base's taskfile, so `task monitoring:install` still deploys base first. `.taskrc.yml` trusts `raw.githubusercontent.com`, so Task downloads those files without asking.

## GitHub Actions

A push to `main` lints the package, builds it, signs it, and publishes `ghcr.io/opendefence/monitoring:<version>` and `:latest`. The workflow reads the version from `.bumpversion.toml` and replaces `+` with `-` in the OCI tag.

Do not edit `.github/workflows/actions.lock` by hand. When a workflow adds or removes a `uses` dependency, update the lock:

```sh
gh actions-lock
```

## Syntax checks

```sh
task --list
task -t taskfiles/build.yaml --list
task -t taskfiles/monitoring.yaml --list
zarf dev lint .
```

These commands do not deploy to a cluster.
