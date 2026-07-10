# catalog-ci

Shared CI/CD building blocks for the **catalog** project - an open collection of
Helm charts and container images (a Bitnami replacement), with a premium tier of
hardened images and installation/upgrade support.

This repository holds **reusable GitHub Actions workflows** that the other
repositories call. Centralizing the build, sign, test and publish logic here
means every chart and image is released the same way, and improvements land
everywhere at once.

## Repositories in the project

| Repository | Purpose |
| --- | --- |
| [`catalog-charts`](https://github.com/vojtechmares/catalog-charts) | Helm charts, one per directory under `charts/`. |
| [`catalog-images`](https://github.com/vojtechmares/catalog-images) | Custom-built container images. |
| [`catalog-hardened-images`](https://github.com/vojtechmares/catalog-hardened-images) | Premium hardened images (minimal, CVE-reduced). |
| **`catalog-ci`** | This repository - reusable workflows shared by the above. |

## Publishing model

Everything is published as **OCI artifacts** to `registry.mareshq.com`:

| Artifact | Location |
| --- | --- |
| Charts | `oci://registry.mareshq.com/catalog/charts/<chart>` |
| Images | `registry.mareshq.com/catalog/images/<image>` |
| Hardened images | `registry.mareshq.com/catalog/hardened/<image>` |

Charts are pulled with `helm pull oci://...` / referenced directly in
dependencies - there is no `helm repo add` index. All artifacts are signed with
[cosign](https://docs.sigstore.dev/) using **keyless / OIDC** signing, so no
long-lived signing keys exist.

## Reusable workflows

### `reusable-chart-lint-test.yaml`

Lints changed charts with [`ct`](https://github.com/helm/chart-testing) and
installs them on an ephemeral [kind](https://kind.sigs.k8s.io/) cluster. Use on
pull requests.

```yaml
jobs:
  lint-test:
    uses: vojtechmares/catalog-ci/.github/workflows/reusable-chart-lint-test.yaml@main
    with:
      charts-dir: charts
```

### `reusable-chart-release.yaml`

Packages, pushes and signs charts whose version is not yet in the registry.
Releases are **idempotent** and driven by the `version` in `Chart.yaml`.

```yaml
jobs:
  release:
    permissions:
      contents: read
      id-token: write # cosign keyless
    uses: vojtechmares/catalog-ci/.github/workflows/reusable-chart-release.yaml@main
    with:
      registry: registry.mareshq.com
      oci-repo: catalog/charts
    secrets:
      registry-username: ${{ secrets.REGISTRY_USERNAME }}
      registry-password: ${{ secrets.REGISTRY_PASSWORD }}
```

### `reusable-image-build.yaml`

Builds a multi-arch image, pushes it, attaches SBOM + provenance, signs it with
cosign and scans it with Trivy.

```yaml
jobs:
  build:
    permissions:
      contents: read
      id-token: write # cosign keyless
    uses: vojtechmares/catalog-ci/.github/workflows/reusable-image-build.yaml@main
    with:
      image: nginx
      context: nginx
      version: "1.27.3"
    secrets:
      registry-username: ${{ secrets.REGISTRY_USERNAME }}
      registry-password: ${{ secrets.REGISTRY_PASSWORD }}
```

See each workflow file for the full list of inputs and defaults.

## Required configuration in calling repositories

Set these as repository or organization **secrets**:

| Secret | Description |
| --- | --- |
| `REGISTRY_USERNAME` | Username / robot account for `registry.mareshq.com`. |
| `REGISTRY_PASSWORD` | Password or token for that account. |

Calling jobs must grant `id-token: write` so cosign can obtain an OIDC token for
keyless signing.

## Verifying signatures

```sh
cosign verify \
  --certificate-identity-regexp '^https://github.com/vojtechmares/catalog-.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  registry.mareshq.com/catalog/images/<image>:<tag>
```

## Hardening notes / roadmap

- Actions are referenced by major-version tag today; pin to commit SHAs (managed
  by the included Dependabot config) before relying on this in production.
- Consider adding branch protection and required status checks to each repo.
- Consider publishing chart provenance (`helm package --sign`) in addition to
  cosign, for Helm-native `helm verify`.

## License

[Apache-2.0](./LICENSE).
