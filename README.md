# PFE-GitOps

> **Dépôt GitOps — Projet de Fin d'Études Sopra Steria / ESPRIT**
> Ce dépôt contient les **manifests Kubernetes** de l'application `my-app` déployée sur **Azure AKS**.
> Il est géré automatiquement par **ArgoCD** selon le pattern GitOps.

---

## Principe GitOps

**Ce dépôt ne se modifie jamais manuellement pour les déploiements.**

Le flux est entièrement automatisé :

```
1. Développeur pousse du code → PFE-DevSecopsProject
       │
       ▼
2. Pipeline GitHub Actions build l'image Docker
   → Push vers saharacr.azurecr.io/my-app:<SHA>
       │
       ▼
3. Job update-gitops : sed image tag dans k8s/deployment.yaml
   → git commit + push vers CE dépôt (PFE-GitOps)
       │
       ▼
4. ArgoCD détecte le changement (polling ~3 min)
       │
       ▼
5. ArgoCD kubectl apply → AKS mis à jour automatiquement
```

**Séparation des responsabilités** :
- [`PFE-DevSecopsProject`](https://github.com/saharhamraoui/PFE-DevSecopsProject) = code applicatif + pipeline + Terraform
- **Ce dépôt** = état désiré de l'infrastructure Kubernetes (source of truth)

---

## Structure

```
PFE-GitOps/
├── k8s/
│   ├── deployment.yaml          # Déploiement application (2 réplicas)
│   ├── service.yaml             # Service LoadBalancer (port 80 → 8080)
│   ├── hpa.yaml                 # HorizontalPodAutoscaler (2→5 pods)
│   ├── ingress.yaml             # NGINX Ingress Controller
│   └── secret-provider-class.yaml  # Azure Key Vault CSI Driver
└── argocd/
    └── application.yaml         # Définition ArgoCD Application
```

---

## Manifests Kubernetes

### `deployment.yaml`
- Image : `saharacr.azurecr.io/my-app:<SHA>` (mis à jour automatiquement par le pipeline)
- 2 réplicas
- Readiness probe + Liveness probe sur `/` port 8080
- Limites : 500m CPU, 256Mi RAM

### `service.yaml`
- Type : `LoadBalancer`
- Port 80 → containerPort 8080
- Expose l'application sur une IP publique Azure

### `hpa.yaml`
- Scale automatique : **2 → 5 pods**
- Déclenché si CPU > 70% **ou** mémoire > 80%

### `ingress.yaml`
- NGINX Ingress Controller
- Routage HTTP vers le service `my-app`

### `secret-provider-class.yaml`
- Azure Key Vault CSI Secrets Store Driver
- Monte le secret `acr-password` depuis Key Vault `sahar-kv-pfe`
- Converti en Kubernetes Secret natif dans les pods

---

## ArgoCD

### Application (`argocd/application.yaml`)

```yaml
source:
  repoURL: https://github.com/saharhamraoui/PFE-GitOps.git
  path: k8s
destination:
  server: https://kubernetes.default.svc
  namespace: default
syncPolicy:
  automated:
    prune: true      # Supprime les ressources retirées de Git
    selfHeal: true   # Annule les changements kubectl manuels
```

### Appliquer l'application ArgoCD

```bash
kubectl apply -f argocd/application.yaml
```

### Accès UI ArgoCD

| Info | Valeur |
|------|--------|
| URL | https://\<IP LoadBalancer ArgoCD\> |
| Username | `admin` |
| Password | récupéré via `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" \| base64 -d` |

---

## Prérequis

- Cluster AKS `sahar-aks` provisionné (Terraform dans [PFE-DevSecopsProject/azure/](https://github.com/saharhamraoui/PFE-DevSecopsProject/tree/main/azure))
- ArgoCD installé sur AKS :
  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd --server-side --force-conflicts \
    -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```
- Secret `GITOPS_TOKEN` configuré dans GitHub Actions (PFE-DevSecopsProject) avec accès Contents R/W sur ce dépôt

---

## Auteur

**Sahar Hamraoui** — Étudiante ingénieure ESPRIT, stagiaire Sopra Steria
