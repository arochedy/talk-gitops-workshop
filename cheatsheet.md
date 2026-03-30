# Aide-memoire ArgoCD, Traefik & GitOps

Referentiel rapide des commandes pour le workshop GitOps & Routing.

---

## ArgoCD -- Gestion des Applications

Lister les Applications ArgoCD :

```bash
kubectl get applications -n argocd
```

Afficher les details d'une Application :

```bash
kubectl describe application <nom> -n argocd
```

Voir le statut de sync d'une Application :

```bash
kubectl get application <nom> -n argocd -o jsonpath='{.status.sync.status}'
```

Voir l'etat de sante d'une Application :

```bash
kubectl get application <nom> -n argocd -o jsonpath='{.status.health.status}'
```

Forcer un refresh (re-lecture du depot Git) :

```bash
kubectl annotate application <nom> -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

Consulter l'historique d'une Application :

```bash
kubectl get application <nom> -n argocd -o jsonpath='{.status.history}' | python3 -m json.tool
```

Supprimer une Application (et ses ressources gerees) :

```bash
kubectl delete application <nom> -n argocd
```

---

## ArgoCD -- Sync Policy

Activer l'auto-sync :

```yaml
syncPolicy:
  automated: {}
```

Activer l'auto-sync avec self-heal et prune :

```yaml
syncPolicy:
  automated:
    selfHeal: true
    prune: true
```

Creer le namespace de destination automatiquement :

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

---

## ArgoCD -- Application CRD

Structure minimale d'une Application (source Git) :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/...
    targetRevision: main
    path: manifests/
  destination:
    server: https://kubernetes.default.svc
    namespace: workshop
```

Structure d'une Application (source Helm) :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-chart
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.example.com
    chart: mon-chart
    targetRevision: "1.0.0"
    helm:
      valuesObject:
        key: value
  destination:
    server: https://kubernetes.default.svc
    namespace: mon-namespace
```

---

## Traefik -- IngressRoute

Lister les IngressRoutes :

```bash
kubectl get ingressroutes -n workshop
```

Afficher les details d'une IngressRoute :

```bash
kubectl describe ingressroute <nom> -n workshop
```

Structure d'une IngressRoute :

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: mon-app
  namespace: workshop
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`mon-app.localhost`)
      kind: Rule
      middlewares:
        - name: mon-middleware
      services:
        - name: mon-service
          port: 9898
```

---

## Traefik -- Middlewares

Lister les Middlewares :

```bash
kubectl get middlewares -n workshop
```

Afficher les details d'un Middleware :

```bash
kubectl describe middleware <nom> -n workshop
```

Rate Limiting :

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    average: 10
    burst: 20
```

Headers personnalises :

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: custom-headers
spec:
  headers:
    customResponseHeaders:
      X-Custom-Header: "valeur"
```

---

## CRDs -- Custom Resource Definitions

Lister toutes les CRDs du cluster :

```bash
kubectl get crds
```

Lister les CRDs d'ArgoCD :

```bash
kubectl get crds | grep argo
```

Lister les CRDs de Traefik :

```bash
kubectl get crds | grep traefik
```

Afficher la definition d'un CRD :

```bash
kubectl describe crd ingressroutes.traefik.io
```

Lister les instances d'une Custom Resource :

```bash
kubectl get <nom-du-crd> -A
```

---

## kubectl -- Commandes utiles

Lister les ressources dans un namespace :

```bash
kubectl get all -n workshop
```

Suivre les changements en temps reel :

```bash
kubectl get pods -n workshop -w
```

Acceder a un service via port-forward :

```bash
kubectl port-forward svc/<nom> <port-local>:<port-service> -n workshop
```

Tester un endpoint HTTP depuis le terminal :

```bash
curl http://blue.localhost
curl -v http://blue.localhost    # avec les headers
```

Tester le rate limiting :

```bash
for i in $(seq 1 30); do
  curl -s -o /dev/null -w "%{http_code}\n" http://green.localhost
done
```

---

## Raccourcis utiles

| Option | Description |
|---|---|
| `-n argocd` | Namespace ArgoCD |
| `-n traefik` | Namespace Traefik |
| `-n workshop` | Namespace du workshop |
| `-o yaml` | Sortie au format YAML |
| `-o jsonpath='{...}'` | Extraire un champ specifique |
| `-w` | Mode watch (mise a jour en temps reel) |
| `-A` | Tous les namespaces |
