# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A GitOps platform repo (published as `github.com/ehalilovic700/platform-repository`, originally forked from course material under `persistentns`) made only of Kubernetes/Backstage YAML. It has no build, lint or test tooling. A change takes effect when it is merged to `main` on GitHub and Argo CD syncs it to the cluster, so editing the local copy changes nothing in a cluster.

To sanity-check a manifest against a cluster that has Crossplane and Argo CD installed: `kubectl apply --dry-run=server -f <file>`.

`GKE-SETUP.md` is the step-by-step runbook for setting up the whole platform on a GKE cluster in a GCP sandbox. It covers creating the cluster, installing Argo CD and Crossplane, applying the root apps and the provider RBAC.

## How the pieces connect

There are three layers, and onboarding one app touches all of them:

1. **Backstage template** (`local-k8s-cluster/template.yaml`, takes the parameters `appname` and `env`):
   - renders `local-k8s-cluster/skeleton/`
   - publishes `skeleton/app-repo/` as a new GitHub repo `ehalilovic700/<appname>`
   - moves `skeleton/platform-repo/argocd/application.yaml` to `infrastructure/argocd/backend/applications/<appname>-<env>.yaml`
   - moves `skeleton/platform-repo/crossplane/claim.yaml` to `infrastructure/crossplane/claims/<appname>-<env>.yaml`
   - opens a PR with both files against **this** repo (branch `add-<appname>-<env>`)
   - registers the app's `catalog-info.yaml` in the Backstage catalog

2. **Crossplane** (`infrastructure/crossplane/`):
   - The XRD defines `XLocalCluster`, and its claim kind is `LocalCluster` (`platform.example.org/v1alpha1`, spec `appName` and `environment`).
   - The `localcluster` Composition runs in Pipeline mode with `function-patch-and-transform`. Through `provider-kubernetes` (ProviderConfig `local-k8s`, `InjectedIdentity`), it creates:
     - a Namespace named after `spec.appName`
     - a `ghcr-secret` dockerconfigjson Secret in that namespace
     - a patch to the namespace's `default` ServiceAccount so it uses that pull secret
   - `environment` is required, but no composition step uses it.
   - `InjectedIdentity` needs the provider's ServiceAccount to have RBAC permissions. That binding is **not** in the repo; `GKE-SETUP.md` step 7 creates it imperatively.
   - `claims/livedemo-v5-dev.yaml` is the leftover example claim.

3. **Argo CD** (`infrastructure/argocd/`):
   - Both Applications in `applications/` are applied by hand with `kubectl`; nothing syncs them.
   - `application.yaml` is the root app. It recursively syncs `infrastructure/crossplane` into `crossplane-system`, with `SkipDryRunOnMissingResource` and retry because the ProviderConfig and claim CRDs only appear after the Provider and XRD install.
   - `backend-applications.yaml` is an app-of-apps for `backend/applications/`. That folder is empty except for `.gitkeep` until template PRs add per-app Applications. Each per-app Application deploys the app repo's `deploy/` folder into namespace `<appname>-<env>`.
   - `platform/` holds the platform services, applied one at a time with `kubectl` in the order given in `GKE-SETUP.md` steps 10–11. They're deliberately not an app-of-apps: Argo CD doesn't wait for child-app health, and the ingress gateway must be created after istiod's injector exists.
     - Istio is installed from Helm charts pinned to version 1.30.8: `istio-base`, then `istiod`, then `istio-ingressgateway` (namespace `istio-ingress`, pod label `istio=ingressgateway`).
     - `backstage` runs the user's own app image, `ghcr.io/ehalilovic700/backstage`. That image is built by GitHub Actions in the separate repo `ehalilovic700/backstage` (a `create-app` app) as linux/amd64, because the user's Mac is Apple Silicon.
       - It uses the chart's bundled PostgreSQL in Backstage's default mode: one database per plugin (`backstage_plugin_*`). The bitnami role `backstage` has CREATEDB, which this mode needs.
       - Don't switch to `pluginDivisionMode: schema`: plugins collided creating their migration tables at first start (`knex_migrations_lock already exists`).
       - Hand-made Secrets: `backstage/backstage-github` (`GITHUB_TOKEN`) and `backstage/backstage-postgres` (`admin-password`, `user-password`). The Postgres password must be fixed, because a chart-generated one would change on every Argo CD sync.
       - Config overrides use Backstage `$env:` references rather than `${VAR}`.
       - With tag `latest` + `pullPolicy: Always`, a new build is rolled out with `kubectl -n backstage rollout restart deploy/backstage`.
     - `istio-routes` syncs `infrastructure/istio/`.

4. **Ingress** (`infrastructure/istio/`): one Istio `Gateway`, `istio-ingress/platform-gateway`, plus a `VirtualService` per UI. Hosts are `<name>.<static-ip>.nip.io`.
   - Argo CD uses TLS **passthrough**, so it keeps its own self-signed cert.
   - Backstage's TLS is terminated at the gateway with a self-signed cert from Secret `istio-ingress/backstage-tls`, which is created by hand and never committed. HTTP redirects to HTTPS for both hosts.
   - Backstage must stay on HTTPS: its new-frontend-system UI calls `crypto.randomUUID`, which only exists on secure pages. Over plain HTTP the sidebar extension (`nav-content:app`) crashes.
   - The static IP is hardcoded in these files and in `platform/istio-ingressgateway.yaml` and `platform/backstage.yaml`. In the template state it's the placeholder `INGRESS_IP`, which step 10a replaces with `sed`.
   - The restriction of the gateway Service to the user's own IP (`loadBalancerSourceRanges`) is applied with `kubectl patch`, never committed, so the user's IP stays out of the public repo.

**Naming invariant:** the claim's `appName` is set to `<appname>-<env>`, so the namespace Crossplane creates matches the destination namespace of the Argo CD Application. Keep the two in sync if you change either template.

**App repo CI** (`skeleton/app-repo/.github/workflows/ci.yaml`): on every push to `main` it builds and pushes `ghcr.io/<owner>/<appname>:<sha>`. It then `sed`-rewrites every `image:` line in `deploy/deployment.yaml` and commits that change back to `main`, and Argo CD rolls it out. The Deployment pulls with `imagePullSecrets: ghcr-secret`, so it depends on the Crossplane composition having created that secret.

## Known gaps

- The template's output text says `infrastructure/argocd/applications`, but per-app files are actually written to `infrastructure/argocd/backend/applications`.
- The skeleton's per-app Argo CD Application has a sync `retry`, because its namespace comes from the Crossplane claim in the same PR and may not exist yet at first sync. Apps generated before that change have no retry and need a manual sync if they hit `namespace not found`.
- Template parameters `appname` and `env` are restricted to lowercase DNS-style names, because they become repo, GHCR image, namespace and Kubernetes object names.
- `infrastructure/argocd/backend/application/testapp.yaml` (singular `application`) is an empty stray file.
- The GitHub owner `ehalilovic700` is hardcoded in the template, the skeleton and the Argo CD apps.
- The provider config file is named `k8s-provide-config.yaml` (sic).

## Editing skeleton files

Files under `local-k8s-cluster/skeleton/` are Backstage Nunjucks templates that use `${{ values.appname }}` and `${{ values.env }}`. To output a literal `${{ ... }}`, for example a GitHub Actions expression, escape it the way `ci.yaml` does: `${{ '${{ secrets.GITHUB_TOKEN }}' }}`. A bare GitHub Actions expression in a skeleton file gets evaluated, and broken, at scaffold time.

## Secrets

The repo is public. In `infrastructure/crossplane/compositions/local-cluster-composition.yaml`, `ghcr-secret` holds the placeholder `{"auths":{}}`, which only works for public images. Never commit real registry credentials or tokens here. Private-image pull credentials have to come from outside git.
