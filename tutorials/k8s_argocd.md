# ArgoCD and GitOps for Kubernetes

ArgoCD is a continuous delivery tool for Kubernetes applications.

ArgoCD monitors a Git repo where your Kubernetes YAML manifests live, and keeps the cluster in sync with whatever is committed there.
Your Git repo becomes the **single source of truth** for the desired state of your application - this pattern is called **GitOps**.

## How it fits together

```
Developer pushes code
        │
        ▼
① GitHub Actions builds a new Docker image and pushes it to DockerHub
        │
        ▼
② GitHub Actions updates the image tag in `infra/k8s`
        │
        ▼
③ ArgoCD detects the change in the repo and syncs the cluster
        │
        ▼
④ New pods are rolled out with the updated image
```



## Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Access the ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0
```

The username is `admin`. Retrieve the initial password with:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```

## Create the ArgoCD Application

In the ArgoCD UI, click **+ New App** and fill in:

1. In the **General** section, set:
   - For **Application Name**, enter `yolo`.
   - For **Project**, select `default`.
   - For **Sync Policy**, select **Automatic**.
2. In the **Source** section, set:
   - For **Repository URL**, enter the URL of your PolyAI GitHub repo.
   - For **Path**, enter the path to the Kubernetes manifests `infra/k8s`.
3. In the **Destination** section, set:
   - For **Cluster URL**, enter `https://kubernetes.default.svc` (this is the in-cluster API server URL).
   - For **Namespace**, enter `default`.
4. Click **Create**. ArgoCD will immediately sync and deploy the YoloService into your cluster.

## Test manual GitOps

Update one of your Kubernetes manifests in `infra/k8s` (e.g. change the number of replicas of the Yolo service) and commit the change to your PolyAI repo.
Commit & push, watch ArgoCD detect the change and roll out the update automatically.


# Exercises 

### :pencil2: CI/CD pipeline for Argo

Let's assume **dev** and **prod** deployments have distinct manifest directories in your  repo:

```
infra/
└── k8s/
    ├── dev/       ← manifests for the dev environment
    └── prod/      ← manifests for the prod environment
```


In the PolyAI repo, modify the GitHub Actions workflow to build and deploy the Yolo service, as follows:
1. Builds and pushes the yolo Docker image.
2. Updates the image tag in the correct manifest directory (`infra/k8s/dev` or `infra/k8s/prod`).
3. Commits and pushes the change back to the same repo so ArgoCD picks it up.

```yaml
name: PolyAI yolo build-deploy

on:
  push:
    branches:
      - dev    # triggers the dev pipeline
      - main   # triggers the prod pipeline

permissions:
  contents: write

jobs:
  build-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: ...
  
      - name: Update YAML manifests
        run: |
          # Update the image tag in the correct environment directory
          # Adjust the sed pattern to match your deployment manifest
          sed -i "s|image: .*/yolo-service:.*|image: ${{ secrets.DOCKERHUB_USER }}/yolo:${{ env.IMAGE_TAG }}|" \
            infra/k8s/${{ env.ENV }}/deployment.yaml

      - name: Commit and push changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add infra/k8s/${{ env.ENV }}/
          git commit -m "ci: update yolo-service image to ${{ env.IMAGE_TAG }}"
          git push
```

### :pencil2: Define ArgoCD apps declaratively

Create one ArgoCD `Application` manifest per environment. Each one points ArgoCD at the correct sub-directory of your PolyAI repo.

**Dev app** - `infra/k8s/argo/yolo-dev.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: yolo-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL:  # TODO change me
    targetRevision: HEAD
    path: infra/k8s/dev/yolo
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Prod app** - `infra/k8s/argo/yolo-prod.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: yolo-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL:  # TODO change me
    targetRevision: HEAD
    path: infra/k8s/prod/yolo
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    # Prod sync is intentionally manual - auto-sync is disabled so you review before promoting.
```

> **Tip:** Use `automated` sync for dev (fast feedback) but leave prod as manual (safer promotions).

- You can bootstrap both apps at once with an **app of apps** - a single ArgoCD app that deploys all your other ArgoCD apps:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL:  # TODO change me
    targetRevision: HEAD
    path: infra/k8s/argo
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
    syncOptions:
      - CreateNamespace=true
```