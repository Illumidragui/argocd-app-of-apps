# CLAUDE.md — argocd-app-of-apps

GitHub repo: `Illumidragui/argocd-app-of-apps`

Root ArgoCD Application that manages all cluster workloads using the App of Apps pattern.
This chart is bootstrapped once by `devsecops-infra/scripts/bootstrap-argocd.sh`.
After that, ArgoCD self-manages it from Git.

## How it works

```
bootstrap-argocd.sh
  └── helm install argocd-apps (this chart)
        └── ArgoCD Application: app-of-apps
              └── ArgoCD Applications (one per entry in values.yaml):
                    ├── cert-manager-clusterissuer  → devsecops-helm/cert-manager-clusterissuer
                    ├── ingress-nginx               → devsecops-helm/ingress-nginx
                    ├── tailscale-operator          → devsecops-helm/tailscale-operator
                    ├── syesite                     → devsecops-helm/syesite-chart
                    ├── hello-world                 → devsecops-helm/hello-world
                    └── kuberflow                   → devsecops-helm/kuberflow
```

All applications are configured with `automated: prune + selfHeal`.
This means **Git is truth** — any manual `kubectl apply` will be reverted within ~3 minutes.

## Adding a new application

1. Create the Helm chart in `devsecops-helm/<new-chart>/`
2. Add an entry to `values.yaml` in this repo following the pattern below:
   ```yaml
   my-app:
     namespace: argo-cd
     project: default
     source:
       repoURL: https://github.com/Illumidragui/devsecops-helm.git
       targetRevision: HEAD
       path: my-app
     destination:
       server: https://kubernetes.default.svc
       namespace: my-app
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```
3. Push to `main` — ArgoCD picks it up automatically

## Removing an application

1. Delete its entry from `values.yaml`
2. Push to `main` — ArgoCD prunes the Application and all its resources (because `prune: true`)

## Lint commands

```bash
helm dependency update .
helm lint .
```

## Key invariants

- `selfHeal: true` — manual cluster changes are reverted; always change via Git
- `prune: true` — removing an entry here removes all cluster resources for that app
- `targetRevision: HEAD` — always tracks the latest commit on `main` in devsecops-helm
- The bootstrap script must be re-run only when starting a fresh cluster; ArgoCD self-manages after that
