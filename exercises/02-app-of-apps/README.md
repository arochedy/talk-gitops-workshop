# Exercice 2 -- App of Apps

## Objectif

Comprendre le pattern App of Apps et déployer plusieurs applications avec une seule Application parente.

## Contexte

Dans l'exercice 1, nous avons créé UNE Application ArgoCD manuellement avec `kubectl apply`. Ça fonctionne, mais imaginez un projet réel avec 10, 20 ou 50 applications. Créer chaque Application une par une avec `kubectl apply` va à l'encontre du principe GitOps : tout doit être dans Git, rien ne doit être fait à la main.

Le pattern **App of Apps** résout ce problème. Le principe est simple :

1. Une **Application parente** surveille un répertoire Git contenant des fichiers YAML d'Applications.
2. ArgoCD synchronise l'application parente, ce qui crée les **Applications enfants** dans le cluster.
3. Chaque application enfant déploie ses propres workloads (Deployments, Services, etc.).

Le résultat : ajouter une nouvelle application au cluster revient à ajouter un fichier YAML dans Git. Zéro `kubectl`, 100% GitOps.

## Prerequis

- Exercice 1 terminé
- ArgoCD installé et fonctionnel
- Port-forward vers ArgoCD disponible (`kubectl port-forward svc/argocd-server -n argocd 8080:80`)

## Étapes

### 1. Explorer les manifestes

Commencez par lister les fichiers de l'exercice :

```bash
ls exercises/02-app-of-apps/
```

<details>
<summary>Sortie attendue</summary>

```
argocd-apps    parent-app.yaml
```
</details><br>

Lisez le manifeste de l'Application parente :

```bash
cat exercises/02-app-of-apps/parent-app.yaml
```

Notez les points importants :
- `path: exercises/02-app-of-apps/argocd-apps` -- le répertoire contenant les Applications enfants.
- `destination.namespace: argocd` (et non `workshop`) car les enfants sont des CRDs Application qui doivent vivre dans le namespace argocd.
- `syncPolicy.automated` avec `prune: true` -- ArgoCD supprimera les enfants qui disparaissent du Git.

Listez maintenant les Applications enfants :

```bash
ls exercises/02-app-of-apps/argocd-apps/
```

Trois Applications enfants. Regardons la premiere :

```bash
cat exercises/02-app-of-apps/argocd-apps/app-blue.yaml
```

Notez la différence clé avec la parente : ici `destination.namespace` est `workshop` car cette Application enfant déploie des workloads (Deployment, Service) dans le namespace workshop. Le champ `path` pointe vers `exercises/02-app-of-apps/demo-apps/app-blue` contenant les manifestes Kubernetes.

Les trois enfants suivent la même structure :

- `app-blue.yaml` → `exercises/02-app-of-apps/demo-apps/app-blue/` (podinfo avec interface bleue)
- `app-green.yaml` → `exercises/02-app-of-apps/demo-apps/app-green/` (podinfo avec interface verte)
- `app-nginx.yaml` → `exercises/02-app-of-apps/demo-apps/app-nginx/` (serveur nginx)

### 2. Appliquer l'Application parente

Appliquez uniquement le manifeste de l'Application parente. ArgoCD se chargera du reste :

```bash
kubectl apply -f exercises/02-app-of-apps/parent-app.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/workshop-apps created
```
</details><br>

C'est la seule commande `kubectl apply` nécessaire. ArgoCD détecte la nouvelle Application parente, synchronise le répertoire `argocd-apps/`, et crée automatiquement les trois Applications enfants. Chaque enfant synchronise ensuite ses propres workloads.

### 3. Observer dans ArgoCD

Ouvrez l'interface ArgoCD dans votre navigateur à l'adresse [http://localhost:8080](http://localhost:8080) (relancez le port-forward si nécessaire).

Vous devriez voir 4 Applications :

- `workshop-apps` (app of apps parente)
- `app-blue` (enfant)
- `app-green` (enfant)
- `app-nginx` (enfant)

Toutes doivent afficher le statut **Synced** et **Healthy** après quelques instants.

Vérifiez également via la CLI :

```bash
kubectl get applications -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
NAME             SYNC STATUS   HEALTH STATUS
app-blue         Synced        Healthy
app-green        Synced        Healthy
app-nginx        Synced        Healthy
workshop-apps    Synced        Healthy
```
</details><br>

### 4. Vérifier les ressources déployées

Vérifiez que les workloads ont bien été déployés dans le namespace `workshop` :

```bash
kubectl get all -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
NAME                             READY   STATUS    RESTARTS   AGE
pod/app-blue-xxxxxxxxxx-xxxxx    1/1     Running   0          60s
pod/app-green-xxxxxxxxxx-xxxxx   1/1     Running   0          60s
pod/app-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          60s

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/app-blue    ClusterIP   10.96.xxx.xxx    <none>        9898/TCP   60s
service/app-green   ClusterIP   10.96.xxx.xxx    <none>        9898/TCP   60s
service/app-nginx   ClusterIP   10.96.xxx.xxx    <none>        80/TCP     60s
```
</details><br>

Accédez à chaque application via port-forward (ouvrez un terminal par application) :

```bash
kubectl port-forward svc/app-blue 9898:9898 -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
Forwarding from 127.0.0.1:9898 -> 9898
```
</details><br>

Ouvrez [http://localhost:9898](http://localhost:9898) -- interface podinfo en bleu avec le message "App Blue".

```bash
kubectl port-forward svc/app-green 9899:9898 -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
Forwarding from 127.0.0.1:9899 -> 9898
```
</details><br>

Ouvrez [http://localhost:9899](http://localhost:9899) -- interface podinfo en vert avec le message "App Green".

```bash
kubectl port-forward svc/app-nginx 8081:80 -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
Forwarding from 127.0.0.1:8081 -> 80
```
</details><br>

Ouvrez [http://localhost:8081](http://localhost:8081) -- page d'accueil nginx par defaut.

Appuyez sur `Ctrl+C` dans chaque terminal pour arreter les port-forwards.

### 5. Observer la suppression en cascade

Supprimez l'Application parente :

```bash
kubectl delete -f exercises/02-app-of-apps/parent-app.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io "workshop-apps" deleted
```
</details><br>

ArgoCD déclenche une **suppression en cascade** :

1. L'Application parente `workshop-apps` est supprimée.
2. Comme la parente avait `prune: true`, ArgoCD supprime les enfants (`app-blue`, `app-green`, `app-nginx`).
3. Comme chaque enfant avait aussi `prune: true`, ArgoCD supprime les workloads (Deployments, Services, Pods).

Vérifiez que toutes les Applications ont disparu :

```bash
kubectl get applications -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
No resources found in argocd namespace.
```
</details><br>

Verifiez que les workloads ont egalement ete supprimes :

```bash
kubectl get all -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
No resources found in workshop namespace.
```
</details><br>

### 6. Re-appliquer pour l'exercice suivant

L'exercice 03 s'appuie sur les applications déployées par le pattern App of Apps. Re-appliquez l'Application parente :

```bash
kubectl apply -f exercises/02-app-of-apps/parent-app.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/workshop-apps created
```
</details><br>

Attendez que toutes les Applications soient synchronisees :

```bash
kubectl get applications -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
NAME             SYNC STATUS   HEALTH STATUS
app-blue         Synced        Healthy
app-green        Synced        Healthy
app-nginx        Synced        Healthy
workshop-apps    Synced        Healthy
```
</details><br>

Tout est en place pour l'exercice suivant.

## Vérification

Avant de passer à l'exercice suivant, assurez-vous que tous les points ci-dessous sont valides :

- [ ] 4 Applications visibles dans ArgoCD (1 parent + 3 enfants)
- [ ] 3 Deployments et 3 Services dans le namespace `workshop`
- [ ] `app-blue` affiche une interface bleue sur [http://localhost:9898](http://localhost:9898)
- [ ] `app-green` affiche une interface verte sur [http://localhost:9899](http://localhost:9899)
- [ ] `app-nginx` affiche la page d'accueil nginx sur [http://localhost:8081](http://localhost:8081)
- [ ] La suppression du parent supprime tout en cascade

## Bonus

- Créez une 4ème application. Dans le répertoire `exercises/02-app-of-apps/demo-apps/`, créez un dossier `app-red/` contenant un Deployment et un Service. Le Deployment utilise l'image `ghcr.io/stefanprodan/podinfo:6.11.2` avec les variables d'environnement `PODINFO_UI_COLOR="#DC143C"` et `PODINFO_UI_MESSAGE="App Red"`. Le Service expose le port 9898.

  Puis créez `exercises/02-app-of-apps/argocd-apps/app-red.yaml` pointant vers `exercises/02-app-of-apps/demo-apps/app-red/`. Si vous avez forké le repo, commitez et poussez vos modifications -- ArgoCD détectera le changement automatiquement. Sinon, appliquez manuellement :

  ```bash
  kubectl apply -f exercises/02-app-of-apps/argocd-apps/app-red.yaml
  ```

  Vérifiez que `app-red` apparaît dans ArgoCD avec `kubectl get applications -n argocd`.

- Remplacez les applications `app-blue`, `app-green` et `app-red` par un chart Helm. Définissez des variables pour la couleur et le message, et utilisez le même chart pour les trois applications en passant des valeurs différentes. Cela démontre la puissance de réutilisation du pattern App of Apps combiné à Helm.

- Explorez la vue "Tree" dans le dashboard ArgoCD : cliquez sur l'Application `workshop-apps` pour voir la hiérarchie complète parent → enfants → workloads.

## Nettoyer

Nous gardons l'Application parente déployée pour l'exercice 03. Si vous souhaitez nettoyer plus tard :

```bash
kubectl delete -f exercises/02-app-of-apps/parent-app.yaml
```

Cela supprimera l'Application parente, les enfants et tous les workloads en cascade.
