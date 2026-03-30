# Exercice 0 -- Mise en place de l'environnement

## Objectif

Installer et vérifier tous les outils nécessaires pour le workshop GitOps & Routing.

## Contexte

Ce workshop est la suite directe du **Workshop 1 (Kubernetes & Helm)**. Nous allons réutiliser minikube et kubectl, et y ajouter **ArgoCD** (outil de déploiement GitOps) et **Traefik** (reverse proxy / ingress controller).

ArgoCD sera installé via Helm dans le cluster minikube. Il surveillera un dépôt Git contenant nos manifestes Kubernetes et les déploiera automatiquement dans le cluster.

## Prérequis

- Workshop 1 terminé (ou connaissances équivalentes en kubectl, Deployments, Services, Helm)
- macOS ou Linux
- Docker Desktop (ou OrbStack) installé et démarré

## Étapes

### 1. Démarrer minikube

Si minikube n'est pas déjà en cours d'exécution :

```bash
minikube start --driver=docker
```

Vérifiez que le cluster est prêt :

```bash
kubectl get nodes
```

### 2. Créer le namespace du workshop

Si le namespace `workshop` n'existe pas encore (depuis le Workshop 1) :

```bash
kubectl create namespace workshop
```

<details>
<summary>Sortie attendue</summary>

```
namespace/workshop created
```
</details><br>

Configurez-le comme namespace par défaut :

```bash
kubectl config set-context --current --namespace=workshop
```

<details>
<summary>Sortie attendue</summary>

```
Context "minikube" modified.
```
</details><br>

### 3. Installer ArgoCD

Ajoutez le depot Helm d'ArgoCD :

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

<details>
<summary>Sortie attendue</summary>

```
"argo" has been added to your repositories
...Successfully got an update from the "argo" chart repository
```
</details><br>


Installez ArgoCD avec le fichier de valeurs fourni :

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f argocd-values.yaml \
  --wait
```

<details>
<summary>Sortie attendue</summary>

```
NAME: argocd
LAST DEPLOYED: ...
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
...
```
</details><br>

L'option `--wait` attend que tous les pods soient prêts avant de rendre la main. Cette etape peut prendre 1 a 2 minutes.

Verifiez que tous les pods sont en cours d'execution :

```bash
kubectl get pods -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          60s
argocd-applicationset-controller-xxxxxxxxxx-xxxxx  1/1     Running   0          60s
argocd-redis-xxxxxxxxxx-xxxxx                      1/1     Running   0          60s
argocd-repo-server-xxxxxxxxxx-xxxxx                1/1     Running   0          60s
argocd-server-xxxxxxxxxx-xxxxx                     1/1     Running   0          60s
```
</details><br>

Tous les pods doivent être en statut `Running` avec `1/1` Ready.

### 4. Récupérer le mot de passe admin

ArgoCD génère un mot de passe administrateur initial stocké dans un Secret :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

<details>
<summary>Sortie attendue</summary>

```
<mot-de-passe-aleatoire>
```
</details><br>

Notez ce mot de passe, vous en aurez besoin pour vous connecter à l'interface ArgoCD.

### 5. Accéder à l'interface ArgoCD

Ouvrez un tunnel vers le serveur ArgoCD :

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

<details>
<summary>Sortie attendue</summary>

```
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```
</details><br>

Ouvrez votre navigateur a l'adresse [http://localhost:8080](http://localhost:8080).

Connectez-vous avec :

- **Utilisateur** : `admin`
- **Mot de passe** : celui récupéré à l'étape précédente

Vous devriez voir le tableau de bord ArgoCD, sans aucune application pour l'instant. C'est normal, nous en créerons dans les exercices suivants.

Appuyez sur `Ctrl+C` dans le terminal pour arrêter le port-forward (vous le relancerez quand nécessaire).

## Vérification

Avant de passer à l'exercice suivant, assurez-vous que tous les points ci-dessous sont valides :

- [ ] `minikube status` affiche "Running"
- [ ] `kubectl get nodes` affiche un noeud "Ready"
- [ ] `kubectl get pods -n argocd` affiche tous les pods en statut "Running"
- [ ] L'interface ArgoCD est accessible sur [http://localhost:8080](http://localhost:8080)
- [ ] Le namespace `workshop` existe

## Bonus

- Explorez l'interface ArgoCD : Settings, Clusters, Repositories, Projects.

- Listez les CRDs installées par ArgoCD pour decouvrir les types de ressources personnalisees :

  ```bash
  kubectl get crds | grep argo
  ```

  <details>
  <summary>Sortie attendue</summary>

  ```
  applications.argoproj.io                    ...
  applicationsets.argoproj.io                 ...
  appprojects.argoproj.io                     ...
  ```
  </details><br>

  Ce sont les **Custom Resource Definitions** ajoutees par ArgoCD. Nous les utiliserons dans les exercices suivants.

- Consultez les projets ArgoCD existants :

  ```bash
  kubectl get appprojects -n argocd
  ```

  <details>
  <summary>Sortie attendue</summary>

  ```
  NAME      AGE
  default   ...
  ```
  </details><br>

  Le projet `default` est créé automatiquement à l'installation. Toutes nos Applications utiliseront ce projet.
