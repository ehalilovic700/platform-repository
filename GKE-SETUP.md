# Running the platform on GKE (GCP sandbox)

When you're done, you'll have:
- a GKE cluster
- Argo CD
- Crossplane
- the root Argo CD app, which syncs `infrastructure/crossplane` from **your** GitHub repo `ehalilovic700/platform-repository` (it installs provider-kubernetes, the function, the XRD and Composition, and the `livedemo-v5-dev` claim)
- an app-of-apps for the per-app Applications that the Backstage template adds
- Istio, with its ingress gateway as the cluster's ingress controller, exposing Argo CD and Backstage (steps 10–11)
- your own Backstage app image with PostgreSQL, instead of the chart's demo image (step 12)

Argo CD reads from GitHub `main`, not from this folder. Any repo change only takes effect after you push it.

## 0. Where to run the commands

**Cloud Shell** (the `>_` icon in the sandbox console) is already signed in as the sandbox user and has `gcloud`, `kubectl`, `helm` and `git`.

**On your Mac**, your normal gcloud login has no access to the sandbox (you'll get `403 ... container.clusters.get`). Create a separate gcloud configuration for the sandbox user:

```bash
gcloud config configurations create sandbox     # creates the config and switches to it
gcloud auth login                               # sign in as the SANDBOX user in the browser
gcloud config set project playground-s-11-247c2506
gcloud auth list                                # the sandbox user should be marked with *

gcloud components install gke-gcloud-auth-plugin kubectl
brew install helm
```

`get-credentials` (step 2) switches your local `kubectl` context to the new cluster. Afterwards, `gcloud config configurations activate default` switches gcloud back to your own account.

## 1. Variables (run again in every new terminal)

On a Mac (zsh), run `setopt interactivecomments` first. Otherwise zsh passes the `# ...` comments in these commands as arguments, which causes errors like `providers.pkg.crossplane.io "#" not found`.

```bash
export PROJECT_ID=playground-s-11-247c2506   # changes every time the sandbox is recreated
export ZONE=us-central1-a
export CLUSTER=platform
export REPO=https://github.com/ehalilovic700/platform-repository.git

gcloud config set project "$PROJECT_ID"
```

## 2. Create the GKE cluster

```bash
gcloud services enable container.googleapis.com

gcloud container clusters create "$CLUSTER" \
  --zone "$ZONE" \
  --num-nodes 3 \
  --machine-type e2-standard-2 \
  --disk-type pd-standard \
  --disk-size 50

# wait for RUNNING
until [ "$(gcloud container clusters describe "$CLUSTER" --zone "$ZONE" --format='value(status)')" = RUNNING ]; do
  echo "still provisioning..."; sleep 20
done

gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
kubectl get nodes          # 3 nodes, Ready
```

## 3. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd wait --for=condition=available deploy --all --timeout=300s
```

## 4. Install Crossplane

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace --wait

kubectl get pods -n crossplane-system
```

## 5. Publish the platform repo to your GitHub (Mac)

The original `persistentns/platform-repository` isn't accessible, so the manifests in this folder now point to `ehalilovic700/platform-repository`. The course author's inline GHCR token has been replaced with an empty placeholder.

Run these on your Mac. `gh` is already signed in as `ehalilovic700`.

```bash
cd ~/Downloads/platform-repository-main

grep -rn "persistentns" --exclude=CLAUDE.md --exclude=GKE-SETUP.md . || echo "OK: no persistentns refs"
grep -rn "ghp_" --exclude=GKE-SETUP.md . || echo "OK: no GitHub tokens"

git init -b main
git add .
git commit -m "Initial platform repo"
gh repo create ehalilovic700/platform-repository --public --source . --push
```

Check that Argo CD will be able to read it without credentials:

```bash
GIT_TERMINAL_PROMPT=0 git ls-remote "$REPO" HEAD && echo "PUBLIC - OK"
```

## 6. Apply the root Argo CD app (Crossplane layer)

`infrastructure/argocd/applications/application.yaml` already includes three settings:

- `directory.recurse: true`, because every manifest is in a subfolder.
- `SkipDryRunOnMissingResource`, because the ProviderConfig and claim CRDs don't exist until the Provider and XRD are installed.
- `retry`, so the sync converges once those CRDs show up.

From `~/Downloads/platform-repository-main`:

```bash
kubectl apply -f infrastructure/argocd/applications/application.yaml
```

Watch it converge:

```bash
kubectl get applications -n argocd
kubectl get providers,functions          # both should show INSTALLED=True, HEALTHY=True
kubectl get xrd,compositions
kubectl wait provider/provider-kubernetes --for=condition=Healthy --timeout=300s
```

## 7. Give provider-kubernetes access to the cluster

The ProviderConfig uses `InjectedIdentity`, so the provider's own ServiceAccount needs RBAC permissions:

```bash
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes \
     | sed -e 's|serviceaccount/|crossplane-system:|')
echo "$SA"                                       # must not be empty
kubectl create clusterrolebinding provider-kubernetes-admin \
  --clusterrole cluster-admin --serviceaccount="$SA"
```

Check that the `livedemo-v5-dev` claim created its namespace, pull secret and ServiceAccount patch:

```bash
kubectl get localclusters -A                      # SYNCED/READY should become True
kubectl get objects.kubernetes.crossplane.io      # 3 objects, READY True
kubectl get ns livedemo-v5-dev
kubectl -n livedemo-v5-dev get secret ghcr-secret
kubectl -n livedemo-v5-dev get sa default -o jsonpath='{.imagePullSecrets}'; echo
```

## 8. App-of-apps for future apps

This deploys every per-app Application in `infrastructure/argocd/backend/applications/`. The folder is empty for now (the livedemo app pointed to a repo you can't access). Applications get added there when the Backstage template's PRs are merged.

```bash
kubectl apply -f infrastructure/argocd/applications/backend-applications.yaml
kubectl get applications -n argocd        # backend-applications: Synced
```

## 9. Optional: Argo CD UI

**On your Mac:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo   # user: admin
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
Then open https://localhost:8080 and accept the self-signed certificate.

**In Cloud Shell:** Web Preview uses plain HTTP, so first switch argocd-server to insecure mode. Then port-forward `8080:80` and use **Web Preview → Preview on port 8080**:
```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge \
  -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deploy/argocd-server
```

## 10. Istio and the Istio ingress gateway (ingress controller)

Argo CD installs Istio from the official Helm charts (version 1.30.8). The Applications are in `infrastructure/argocd/platform/`. Apply them **in order**: the gateway pods need the istiod sidecar injector to exist already, or they get stuck with `image: auto`.

**10a. Reserve a static IP and write it into the manifests.** Hostnames use `nip.io`, so `argocd.<ip>.nip.io` resolves to the IP without any DNS setup. Argo CD reads from GitHub, so the change has to be pushed.

```bash
setopt interactivecomments
cd ~/Downloads/platform-repository-main
export REGION=us-central1

gcloud compute addresses create istio-ingress --region "$REGION"
export INGRESS_IP=$(gcloud compute addresses describe istio-ingress --region "$REGION" --format='value(address)')
echo "$INGRESS_IP"

grep -rl INGRESS_IP infrastructure | xargs sed -i '' "s/INGRESS_IP/$INGRESS_IP/g"
grep -rn INGRESS_IP infrastructure || echo "OK: IP filled in"

git add . && git commit -m "Add Istio ingress, Backstage and routes" && git push
```

**10b. Install Istio in order.** Each step waits until the previous Argo CD app is Healthy.

It's normal for `istio-base` to stay `Progressing` for about 3 minutes, and for `istiod` to show `Degraded` for under a minute while its autoscaler waits for its first CPU readings.

```bash
wait_app() { until [ "$(kubectl -n argocd get application "$1" -o jsonpath='{.status.health.status}' 2>/dev/null)" = Healthy ]; do echo "waiting for $1..."; sleep 10; done; echo "$1 is Healthy"; }

kubectl apply -f infrastructure/argocd/platform/istio-base.yaml
wait_app istio-base

kubectl apply -f infrastructure/argocd/platform/istiod.yaml
wait_app istiod

kubectl apply -f infrastructure/argocd/platform/istio-ingressgateway.yaml
wait_app istio-ingressgateway

kubectl -n istio-ingress get pods
kubectl -n istio-ingress get svc istio-ingressgateway    # EXTERNAL-IP must equal $INGRESS_IP
```

**10c. Allow only your own IP through the ingress.** Backstage will hold your GitHub token, so don't leave it open to the internet. This is a `kubectl patch` rather than a git change, so your home IP stays out of the public repo. Argo CD won't revert it, because the chart doesn't set the field.

```bash
export MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "$MY_IP"
kubectl -n istio-ingress patch svc istio-ingressgateway --type merge \
  -p "{\"spec\":{\"loadBalancerSourceRanges\":[\"$MY_IP/32\"]}}"
```

## 11. Expose Argo CD and Backstage through the Istio ingress

`infrastructure/istio/` defines one `Gateway`, `platform-gateway`, and two `VirtualService`s:

| URL | Routes to | TLS |
|---|---|---|
| `https://argocd.<INGRESS_IP>.nip.io` | `argocd-server.argocd:443` | Passed through to Argo CD's own self-signed cert. HTTP redirects to HTTPS. |
| `http://backstage.<INGRESS_IP>.nip.io` | `backstage.backstage:7007` | Plain HTTP |

Backstage runs **your own image**, `ghcr.io/ehalilovic700/backstage`, with the chart's bundled PostgreSQL. It uses guest sign-in and loads `local-k8s-cluster/template.yaml` from this repo.

The image is built once in step 12 (12a–12d) and stays in GHCR, so a new sandbox can reuse it. If it doesn't exist yet, do 12a–12d first.

Two Secrets are created by hand and never committed: your GitHub token, and a fixed Postgres password.

```bash
kubectl create namespace backstage
kubectl -n backstage create secret generic backstage-github \
  --from-literal=GITHUB_TOKEN="$(gh auth token)"
kubectl -n backstage create secret generic backstage-postgres \
  --from-literal=admin-password="$(openssl rand -hex 16)" \
  --from-literal=user-password="$(openssl rand -hex 16)"

kubectl apply -f infrastructure/argocd/platform/backstage.yaml
wait_app backstage                 # the first start takes 2-3 minutes

kubectl apply -f infrastructure/argocd/platform/istio-routes.yaml
wait_app istio-routes

curl -skI "https://argocd.$INGRESS_IP.nip.io" | head -1      # expect 200
curl -sI  "http://backstage.$INGRESS_IP.nip.io" | head -1    # expect 200

echo "Argo CD:   https://argocd.$INGRESS_IP.nip.io"
echo "Backstage: http://backstage.$INGRESS_IP.nip.io"
```

- **Argo CD:** accept the self-signed certificate warning and log in as `admin`. Change the initial password right away: **User Info → Update Password**. Once you've done that, remove the initial password Secret: `kubectl -n argocd delete secret argocd-initial-admin-secret`.
- **Backstage:** click **Enter** as Guest, then **Create**. The **Local Cluster + App + CI** template should be listed. It may take a minute to appear while the catalog does its first read.
- The catalog is stored in Postgres (a 2Gi volume), so it survives pod restarts. It's lost when the cluster is deleted.

## 12. Build your own Backstage app (replaces the demo image)

The Helm chart's default image, `ghcr.io/backstage/backstage`, is a demo build with a minimal UI and no sidebar. The [Backstage Kubernetes guide](https://backstage.io/docs/deployment/k8s) instead deploys **your own app**, created with `create-app`, which has the usual sidebar: Home, Catalog, APIs, Docs, Create…

The image is built by GitHub Actions on x86 (amd64). Don't build it on your Mac: Apple Silicon produces an ARM image, and the GKE nodes can't run it.

Steps 12a–12d are done once; the image then stays in GHCR. Step 12e switches a running cluster over to it.

### 12a. Create the app (Mac, about 5 minutes)

This needs Node 22 or 24 (you have 24).

```bash
cd ~/Downloads
npx @backstage/create-app@latest --path backstage    # when asked for the app name, enter: backstage
```

Optionally, preview it locally with `cd ~/Downloads/backstage && yarn start` and open http://localhost:3000. Press Ctrl-C when you're done.

### 12b. Prepare the app for the cluster

There are three changes:

1. Install `mkdocs` into the image, so the **Docs** tab can build docs inside the pod.
2. Change `backend.listen` from a string to object form. The Helm chart injects `APP_CONFIG_backend_listen_port`, and Backstage can't combine that with `listen: ':7007'`, so it crashes at startup.
3. Add a CI workflow that builds the image and pushes it to GHCR.

```bash
cd ~/Downloads/backstage

python3 - <<'EOF'
p = 'packages/backend/Dockerfile'
s = open(p).read()
block = '''# TechDocs: build docs inside the container (techdocs.generator.runIn: local)
RUN apt-get update && \\
    apt-get install -y --no-install-recommends python3 python3-pip python3-venv && \\
    rm -rf /var/lib/apt/lists/*
ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
RUN pip3 install mkdocs-techdocs-core

'''
if 'mkdocs-techdocs-core' in s:
    print('already done')
elif 'USER node' not in s:
    print('USER node not found - paste the block above the line that switches to the node user')
else:
    open(p, 'w').write(s.replace('USER node', block + 'USER node', 1))
    print('Dockerfile updated')
EOF

python3 - <<'EOF'
p = 'app-config.production.yaml'
s = open(p).read()
if "listen: ':7007'" in s:
    open(p, 'w').write(s.replace("listen: ':7007'", "listen:\n    port: 7007", 1))
    print('backend.listen changed to object form')
else:
    print('backend.listen already OK')
EOF

mkdir -p .github/workflows
cat > .github/workflows/build-image.yaml <<'EOF'
name: Build image

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
      - run: corepack enable
      - run: yarn install --immutable
      - run: yarn tsc
      - run: yarn build:backend
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: packages/backend/Dockerfile
          platforms: linux/amd64
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/backstage:latest
            ghcr.io/${{ github.repository_owner }}/backstage:${{ github.sha }}
EOF
```

### 12c. Push the app to GitHub and let CI build it (about 10 minutes)

```bash
cd ~/Downloads/backstage
git init -b main 2>/dev/null       # harmless if create-app already did it
git add -A
git commit -m "Backstage app with CI image build"
git branch -M main
gh repo create ehalilovic700/backstage --public --source . --push

sleep 15
gh run watch "$(gh run list --workflow build-image.yaml --limit 1 --json databaseId -q '.[0].databaseId')"
```

### 12d. Make the image public (one time, in the browser)

New GHCR packages are private, and the cluster has no credentials to pull them.

Open https://github.com/users/ehalilovic700/packages/container/package/backstage, go to **Package settings**, then **Change visibility**, and choose **Public**.

Check that it can be pulled anonymously:

```bash
TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:ehalilovic700/backstage:pull" | sed -E 's/.*"token":"([^"]+)".*/\1/')
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.oci.image.index.v1+json, application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json" \
  https://ghcr.io/v2/ehalilovic700/backstage/manifests/latest     # 200 = public
```

### 12e. Switch the running cluster to your image and Postgres

`infrastructure/argocd/platform/backstage.yaml` already points to your image and enables the chart's PostgreSQL. Like the other files in `platform/`, Argo CD doesn't sync it from git: apply it with `kubectl`, and push it so the repo matches.

```bash
kubectl -n backstage create secret generic backstage-postgres \
  --from-literal=admin-password="$(openssl rand -hex 16)" \
  --from-literal=user-password="$(openssl rand -hex 16)"

cd ~/Downloads/platform-repository-main
kubectl apply -f infrastructure/argocd/platform/backstage.yaml

kubectl -n backstage rollout status statefulset/backstage-postgresql --timeout=300s
kubectl -n backstage rollout status deploy/backstage --timeout=600s
wait_app backstage

git add infrastructure/argocd/platform/backstage.yaml
git commit -m "Backstage: own image + PostgreSQL"
git push
```

Hard-refresh http://backstage.34.28.253.199.nip.io (Cmd+Shift+R), because the browser may have cached the old frontend. You should see the sidebar, and **Create** should list **Local Cluster + App + CI**.

To roll out a new build of your app later, push to `ehalilovic700/backstage`, wait for CI, then run `kubectl -n backstage rollout restart deploy/backstage`.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Ingress `EXTERNAL-IP` stays `<pending>` or doesn't match `$INGRESS_IP` | The reserved address is in a different region or already in use. Check with `gcloud compute addresses list`. |
| `istio-ingress` pods in `ImagePullBackOff` with `image: auto` | The gateway was created before istiod's injector was running. Fix: `kubectl -n istio-ingress rollout restart deploy/istio-ingressgateway` |
| `istio-base` / `istiod` keep going OutOfSync on webhook configurations | istiod rewrites these at runtime. The apps' `ignoreDifferences` already covers the known fields; describe the app to see which other field is changing. |
| The `nip.io` URLs time out in the browser | Your public IP changed, so the source-range restriction blocks you. Re-run step 10c. |
| Backstage pod in `CreateContainerConfigError` | The `backstage-github` Secret is missing. Re-run the first commands of step 11. |
| Backstage page is blank or shows errors | Check `kubectl -n backstage logs deploy/backstage`. Config errors show up at startup. |
| Argo CD URL shows an Istio 404 or a connection reset | The `istio-routes` app isn't Synced, or the `INGRESS_IP` placeholders weren't replaced before the push. Check: `grep -rn INGRESS_IP infrastructure` |
| CI fails in the Backstage repo | Run `gh run view --log-failed -R ehalilovic700/backstage`. Usually it's `yarn tsc` or `yarn build:backend` in the generated app. |
| Backstage crashes with `Invalid type in config for key 'backend.listen' ... got string, wanted object` | Your app's `app-config.production.yaml` has `listen: ':7007'`. Change it to `listen:` / `port: 7007` (step 12b), push, wait for CI, then `kubectl -n backstage rollout restart deploy/backstage`. |
| Backstage pod in `ImagePullBackOff` (step 12) | CI hasn't finished yet, or the package is still private. See step 12d. |
| Backstage logs show `password authentication failed` | `backstage-postgres` was recreated after Postgres had already set up its volume (it only reads the password the first time). Fix: `kubectl -n backstage delete pvc data-backstage-postgresql-0 && kubectl -n backstage delete pod backstage-postgresql-0`. This wipes the catalog. |
| Still no sidebar after step 12e | The browser is using the cached old frontend: hard-refresh. Also check that the pod runs your image: `kubectl -n backstage get pods -o jsonpath='{..image}'` |
| The **Docs** tab says the docs failed to build | The image was built before step 12b's Dockerfile change. Re-run CI, then restart the deployment. |
| `fatal: bad repository ''` | The step 1 variables aren't set in this terminal. Run the `export` lines again. |
| Root app shows `repository not found` / `authentication required` | Step 5 hasn't been pushed yet, or the repo isn't public. |
| Root app is Synced but nothing gets created | An older root app without `directory.recurse` was applied. Re-run step 6. |
| `provider-kubernetes` not healthy, or an `unauthorized` / pull error on the package | `xpkg.upbound.io/upbound/provider-kubernetes:v0.14.0` can't be pulled. Change `infrastructure/crossplane/providers/k8s-provider.yaml` to a current `xpkg.crossplane.io/crossplane-contrib/provider-kubernetes` release, then commit and push. |
| `objects.kubernetes.crossplane.io` show `forbidden` errors | Step 7 hasn't run, or it ran before the provider was installed (so `$SA` was empty). Re-run step 7. |
| App pods in `ImagePullBackOff` for a private GHCR image | `ghcr-secret` only contains an empty placeholder. Make the GHCR package public, or create the pull secret from your own token outside git. |

## Cleanup

```bash
gcloud container clusters delete "$CLUSTER" --zone "$ZONE" --quiet
gcloud compute addresses delete istio-ingress --region "$REGION" --quiet
gcloud config configurations activate default     # Mac: back to your own gcloud account
```
