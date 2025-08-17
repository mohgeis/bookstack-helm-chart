# bookstack-helm-chart

A Helm chart to deploy [BookStack](https://www.bookstackapp.com/) using the LinuxServer.io image.

- Image: `linuxserver/bookstack` ([Docker Hub](https://hub.docker.com/r/linuxserver/bookstack))
- Upstream install docs: see [BookStack Docker installation](https://www.bookstackapp.com/docs/admin/installation/#docker)

## TL;DR

```bash
helm repo add my-bookstack https://<your-gh-pages-or-host>/bookstack-helm-chart
helm install my-bstack my-bookstack/bookstack \
  --set app.url=https://docs.example.com \
  --set db.host=mariadb.default.svc.cluster.local \
  --set db.user=bookstack --set db.password=supersecret
```

## Prerequisites

- A reachable MySQL/MariaDB instance and credentials.
- Persistent storage class (if `persistence.enabled=true`).

## Configuration

Key values (see `values.yaml` for full list):

- `image.repository` / `image.tag`
- `app.url` (required)
- `app.timezone`, `app.puid`, `app.pgid`
- `db.host`, `db.port`, `db.name`, `db.user`, `db.password` or `db.existingSecret`
- `service.*`, `ingress.*`
- `persistence.*`

## Using an existing secret for DB

Create a secret with keys: `db-user`, `db-password`, `db-name`, `db-host`, `db-port` and set `db.existingSecret` to its name.

```bash
kubectl create secret generic bookstack-db \
  --from-literal=db-user=bookstack \
  --from-literal=db-password=supersecret \
  --from-literal=db-name=bookstack \
  --from-literal=db-host=mariadb \
  --from-literal=db-port=3306
```

## Ingress Example

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: docs.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docs-tls
      hosts:
        - docs.example.com
```

## Publishing to Artifact Hub (OCI via GHCR)

This chart is configured to publish to GitHub Container Registry (OCI) on merges to `main`. You can then add the OCI repo in Artifact Hub.

Publish flow:

1. On merge to `main`, the GitHub Action logs in to GHCR and pushes the packaged chart to `oci://ghcr.io/<owner>/bookstack`.
2. Artifact Hub can index this OCI repository when you register it.

Register in Artifact Hub:

1. Sign in to Artifact Hub with your GitHub account: https://artifacthub.io/
2. Add a new repository: Type `Helm charts`, Kind `OCI`, URL `oci://ghcr.io/<owner>/bookstack`.
3. Ensure `.artifacthub-repo.yml` exists with owners info (added with your email), and `charts/artifacthub-repo.yml` includes `type: oci`.
4. Save. Artifact Hub will validate and begin indexing releases.

Dry run on PRs: the workflow lints, renders, packages, and uploads the `.tgz` as an artifact without publishing.

If you prefer classic Helm repo indexing instead of OCI, see the section below.

## TLS via cert-manager (optional)

If you already have cert-manager and an Issuer/ClusterIssuer, you can have the chart request a certificate automatically:

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: bookstack.example.com
      paths:
        - path: /
          pathType: Prefix

certificate:
  enabled: true
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-prod
  # Optional overrides
  # secretName: bookstack-tls
  # dnsNames:
  #   - bookstack.example.com
```

The certificate secret will be referenced by the Ingress as usual via `ingress.tls`.

## Packaging & Publishing a Helm repo

- Package:

```bash
helm package .
```

- Generate `index.yaml` for your repo folder:

```bash
mkdir -p repo && mv bookstack-*.tgz repo/
helm repo index repo --url https://<your-gh-pages-or-host>/bookstack-helm-chart
```

- Serve statically (e.g., GitHub Pages). Commit and push `repo/`.

Consumers can add your repo:

```bash
helm repo add my-bookstack https://<your-gh-pages-or-host>/bookstack-helm-chart
helm repo update
helm install my-bstack my-bookstack/bookstack
```

## GitHub Actions: Auto-publish to gh-pages

This repo includes a workflow at `.github/workflows/publish-helm.yaml` that packages the chart and publishes it to the `gh-pages` branch as a Helm repo. It runs on pushes to `main` that modify chart files, and on manual dispatch.

Steps to enable:

1. In repo settings, enable GitHub Pages to serve from the `gh-pages` branch (root).
2. Ensure the workflow has `permissions: contents: write` (already set) so it can push.
3. Push to `main`. The workflow will create/update `gh-pages` with packaged charts and `index.yaml`.

The repo URL used by the workflow is:

```
https://<owner>.github.io/<repo>
```

If using a custom domain, update the `REPO_URL` logic in the workflow and pass a `cname` input to the gh-pages action to publish a CNAME file.

## Notes on the image

This chart defaults to the LinuxServer.io `bookstack` image. If you prefer an official image in future, change `image.repository` and env var names accordingly.

Links:
- BookStack Docker docs: https://www.bookstackapp.com/docs/admin/installation/#docker
- LinuxServer.io image: https://hub.docker.com/r/linuxserver/bookstack
