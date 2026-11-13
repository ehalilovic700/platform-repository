# Running the platform on GKE (GCP sandbox)

When you're done, you'll have:
- a GKE cluster
- Argo CD
- Crossplane
- the root Argo CD app, which syncs `infrastructure/crossplane` from **your** GitHub repo `ehalilovic700/platform-repository` (it installs provider-kubernetes, the function, the XRD and Composition, and the `livedemo-v5-dev` claim)
- an app-of-apps for the per-app Applications that the Backstage template adds

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

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `fatal: bad repository ''` | The step 1 variables aren't set in this terminal. Run the `export` lines again. |
| Root app shows `repository not found` / `authentication required` | Step 5 hasn't been pushed yet, or the repo isn't public. |
| Root app is Synced but nothing gets created | An older root app without `directory.recurse` was applied. Re-run step 6. |
| `provider-kubernetes` not healthy, or an `unauthorized` / pull error on the package | `xpkg.upbound.io/upbound/provider-kubernetes:v0.14.0` can't be pulled. Change `infrastructure/crossplane/providers/k8s-provider.yaml` to a current `xpkg.crossplane.io/crossplane-contrib/provider-kubernetes` release, then commit and push. |
| `objects.kubernetes.crossplane.io` show `forbidden` errors | Step 7 hasn't run, or it ran before the provider was installed (so `$SA` was empty). Re-run step 7. |
| App pods in `ImagePullBackOff` for a private GHCR image | `ghcr-secret` only contains an empty placeholder. Make the GHCR package public, or create the pull secret from your own token outside git. |

## Cleanup

```bash
gcloud container clusters delete "$CLUSTER" --zone "$ZONE" --quiet
gcloud config configurations activate default     # Mac: back to your own gcloud account
```
