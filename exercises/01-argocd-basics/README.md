# Exercice 1 -- ArgoCD : deployer via Git

## Objectif

Creer une Application ArgoCD, comprendre le cycle sync/drift/auto-sync et effectuer un rollback.

## Contexte

Dans le Workshop 1, nous avons déployé nos applications manuellement avec `kubectl apply`. Cette approche fonctionne, mais elle présente des limites : pas de trace de qui a déployé quoi, pas de moyen simple de revenir en arrière, et aucune garantie que le cluster reflète l'état souhaité.

Avec ArgoCD, on adopte l'approche GitOps : un dépôt Git devient la **source de vérité unique**. ArgoCD surveille ce dépôt en continu grâce à une **boucle de réconciliation** et synchronise automatiquement les manifestes Kubernetes dans le cluster.

Le pont entre Git et Kubernetes est le CRD **Application**. Cette ressource indique à ArgoCD :
- **Où** trouver les manifestes (repository Git, branche, chemin)
- **Où** les deployer (cluster cible, namespace)
- **Comment** se comporter (sync manuelle ou automatique, self-heal, prune)

Dans cet exercice, nous allons créer une Application ArgoCD pas à pas, observer chaque comportement, puis activer progressivement les fonctionnalités d'automatisation.

## Prerequis

- Exercice 0 terminé (ArgoCD installé et fonctionnel)
- Port-forward vers ArgoCD disponible (`kubectl port-forward svc/argocd-server -n argocd 8080:80`)
- Fork du repo github https://github.com/stafyniaksacha/talk-gitops-workshop dans votre propre compte

## Etapes

### 1. Explorer le manifeste Application

Commencez par lire le fichier `application.yaml` situé dans le répertoire de l'exercice :

```bash
cat application.yaml
```

Prenez le temps de comprendre chaque section :

- **metadata.name** : le nom de l'Application, affiché dans l'interface ArgoCD (`app-blue`).
- **metadata.namespace** : les Applications ArgoCD doivent vivre dans le namespace `argocd`.
- **spec.source.repoURL** : l'URL du depot Git surveillé par ArgoCD. 
  - mettez ce champ à jour pour pointer sur votre fork.
- **spec.source.targetRevision** : la branche Git à suivre (`main`).
- **spec.source.path** : le chemin dans le depot contenant les manifestes Kubernetes (`exercises/01-argocd-basics/manifests`).
- **spec.destination.server** : le cluster ciblé (`https://kubernetes.default.svc` = le cluster local).
- **spec.destination.namespace** : le namespace dans lequel les ressources seront déployées (`workshop`).
- **spec.syncPolicy** : la politique de synchronisation. Ici `{}` signifie sync manuelle.

Le repertoire `manifests/` contient un Deployment et un Service pour `app-blue`, une application podinfo configurée avec une couleur bleue et le message "Deployed with ArgoCD!".

### 2. Appliquer l'Application

Creez l'Application dans le cluster :

```bash
kubectl apply -f application.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/app-blue created
```
</details><br>

Verifiez que l'Application a ete créée dans le namespace `argocd` :

```bash
kubectl get applications -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
NAME       SYNC STATUS   HEALTH STATUS   PROJECT   AGE
app-blue   OutOfSync     Missing         default   10s
```
</details><br>

Le statut `OutOfSync` est normal : ArgoCD a detecté les manifestes dans Git mais ne les a pas encore synchronisés dans le cluster (car la syncPolicy est manuelle).

### 3. Observer dans l'interface ArgoCD

Si ce n'est pas déjà fait, ouvrez un tunnel vers l'interface ArgoCD :

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

L'Application `app-blue` apparait dans le tableau de bord avec le statut **OutOfSync**. Cela signifie que l'etat souhaite (defini dans Git) ne correspond pas a l'etat actuel du cluster (ou les ressources n'existent pas encore).

> **Note** : laissez le port-forward actif dans ce terminal et ouvrez un nouveau terminal pour les etapes suivantes.

### 4. Synchroniser l'application

Dans l'interface ArgoCD, cliquez sur l'Application `app-blue`, puis sur le bouton **Sync**, puis sur **Synchronize**.

Vous pouvez aussi synchroniser via la ligne de commande (impératif) :

```bash
kubectl patch application app-blue \
  -n argocd \
  --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/app-blue patched
```
</details><br>

Attendez quelques secondes, puis vérifiez le statut de l'Application :

```bash
kubectl get applications -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
NAME       SYNC STATUS   HEALTH STATUS   PROJECT   AGE
app-blue   Synced        Healthy         default   2m
```
</details><br>

Le statut est passé à **Synced** (les manifestes Git sont appliques dans le cluster) et **Healthy** (les ressources fonctionnent correctement).

Verifiez que les ressources ont été créées dans le namespace `workshop` :

```bash
kubectl get all -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
NAME                            READY   STATUS    RESTARTS   AGE
pod/app-blue-6f8b5b7b4d-abc12   1/1     Running   0          30s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/app-blue   ClusterIP   10.xxx.xxx.xxx   <none>        9898/TCP   30s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-blue   1/1     1            1           30s
```
</details><br>

ArgoCD a créé le Deployment (avec 1 replica) et le Service, exactement comme définis dans les manifestes Git.

### 5. Accéder à l'application

Ouvrez un port-forward vers le Service `app-blue` :

```bash
kubectl port-forward svc/app-blue 9898:9898 -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
Forwarding from 127.0.0.1:9898 -> 9898
Forwarding from [::1]:9898 -> 9898
```
</details><br>


Ouvrez votre navigateur a l'adresse [http://localhost:9898](http://localhost:9898).

Vous devriez voir l'interface de podinfo avec une couleur bleue et le message **"Deployed with ArgoCD!"**. Cette application a été déployée par ArgoCD à partir des manifestes présents dans le dépôt Git.

Appuyez sur `Ctrl+C` pour arrêter le port-forward.

### 6. Observer le drift

Dans un autre terminal, modifiez manuellement le nombre de replicas du Deployment :

```bash
kubectl scale deployment app-blue --replicas=3 -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
deployment.apps/app-blue scaled
```
</details><br>

Verifiez que 3 Pods sont en cours d'execution :

```bash
kubectl get pods -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
NAME                        READY   STATUS    RESTARTS   AGE
app-blue-6f8b5b7b4d-abc12   1/1     Running   0          5m
app-blue-6f8b5b7b4d-def34   1/1     Running   0          10s
app-blue-6f8b5b7b4d-ghi56   1/1     Running   0          10s
```
</details><br>

Maintenant, verifiez le statut de l'Application dans ArgoCD :

```bash
kubectl get applications app-blue -n argocd -o jsonpath='{.status.sync.status}'
```

<details>
<summary>Sortie attendue</summary>

```
OutOfSync
```
</details><br>

L'Application est repassée en **OutOfSync**. C'est le **drift** : l'état du cluster (3 replicas) ne correspond plus à l'état défini dans Git (1 replica). ArgoCD détecte cette divergence automatiquement grâce à sa boucle de reconciliation.

Dans l'interface ArgoCD, l'Application apparait désormais en jaune avec le statut OutOfSync.

### 7. Activer l'auto-sync

Editez le fichier `application.yaml` et remplacez la section `syncPolicy` :


```diff
-  syncPolicy: {}
+  syncPolicy:
+    automated: {}
```

Appliquez la modification :

```bash
kubectl apply -f application.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/app-blue configured
```
</details><br>

Observez le comportement : ArgoCD détecte que la syncPolicy est désormais automatique. Puisque l'Application est OutOfSync (3 replicas au lieu de 1), ArgoCD lance automatiquement une synchronisation. Après quelques secondes, le nombre de replicas revient à 1.

Vérifiez :

```bash
kubectl get pods -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
NAME                        READY   STATUS    RESTARTS   AGE
app-blue-6f8b5b7b4d-abc12   1/1     Running   0          8m
```
</details><br>

```bash
kubectl get applications -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
NAME       SYNC STATUS   HEALTH STATUS   PROJECT   AGE
app-blue   Synced        Healthy         default   10m
```
</details><br>

L'auto-sync signifie qu'ArgoCD synchronise automatiquement lorsqu'il détecte une différence entre Git et le cluster. Cependant, il ne surveille pas encore les modifications manuelles en temps réel.

### 8. Activer le self-heal

Editez `application.yaml` et mettez a jour la section `syncPolicy` :


```diff
  syncPolicy:
-   automated: {}
+   automated:
+     selfHeal: true
```

Appliquez la modification :

```bash
kubectl apply -f application.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/app-blue configured
```
</details><br>

Maintenant, tentez de nouveau de modifier manuellement le cluster :

```bash
kubectl scale deployment app-blue --replicas=3 -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
deployment.apps/app-blue scaled
```
</details><br>

Attendez quelques secondes et observez :

```bash
kubectl get pods -n workshop -w
```

<details>
<summary>Sortie attendue</summary>

```
NAME                        READY   STATUS        RESTARTS   AGE
app-blue-6f8b5b7b4d-abc12   1/1     Running       0          12m
app-blue-6f8b5b7b4d-def34   1/1     Running       0          5s
app-blue-6f8b5b7b4d-ghi56   1/1     Terminating   0          5s
```
</details><br>

ArgoCD détecte la modification manuelle et **revert automatiquement** le cluster a l'état défini dans Git (1 replica). C'est le **self-heal** : toute modification manuelle du cluster est annulée par ArgoCD en quelques secondes.

Appuyez sur `Ctrl+C` pour arrêter l'observation.

Vérifiez que l'Application est de nouveau synchronisée :

```bash
kubectl get applications -n argocd
```

<details>
<summary>Sortie attendue</summary>

```
NAME       SYNC STATUS   HEALTH STATUS   PROJECT   AGE
app-blue   Synced        Healthy         default   13m
```
</details><br>

### 9. Activer le prune

Editez `application.yaml` et ajoutez l'option `prune` :


```diff
  syncPolicy:
    automated:
      selfHeal: true
+     prune: true
```

Appliquez la modification :

```bash
kubectl apply -f application.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/app-blue configured
```
</details><br>

Le **prune** indique à ArgoCD de supprimer du cluster les ressources qui ne sont plus présentes dans Git. Sans cette option, si vous supprimez un manifeste du dépôt Git, la ressource correspondante resterait orpheline dans le cluster.

Avec `selfHeal: true` et `prune: true`, ArgoCD assure une correspondance exacte et permanente entre Git et le cluster :
- **selfHeal** : revert les modifications manuelles sur les ressources existantes.
- **prune** : supprime les ressources qui n'existent plus dans Git.

Vérifiez la configuration finale :

```bash
kubectl get applications app-blue -n argocd -o jsonpath='{.spec.syncPolicy}'
```

<details>
<summary>Sortie attendue</summary>

```json
{
  "automated": {
    "selfHeal": true,
    "prune": true
  }
}
```
</details><br>

### 10. Nettoyer

Supprimez l'Application ArgoCD :

```bash
kubectl delete -f application.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io "app-blue" deleted
```
</details><br>

Vérifiez que les ressources gérées ont aussi été supprimées du namespace `workshop` :

```bash
kubectl get all -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
No resources found in workshop namespace.
```
</details><br>

La suppression de l'Application a entraîné la suppression de toutes les ressources qu'elle gérait (Deployment et Service). C'est la **suppression en cascade** : puisque `prune: true` était active, ArgoCD supprime les ressources du cluster lorsque l'Application elle-même est supprimée.

> **Note** : si vous aviez `prune: false`, les ressources resteraient dans le cluster même après suppression de l'Application. Il faudrait alors les supprimer manuellement.

## Vérification

- [ ] L'Application apparait dans l'interface ArgoCD
- [ ] L'application app-blue est déployée dans le namespace workshop
- [ ] L'interface podinfo affiche le message "Deployed with ArgoCD!"
- [ ] ArgoCD détecte le drift quand on modifie manuellement le cluster
- [ ] Le self-heal revert les modifications manuelles
- [ ] La suppression de l'Application supprime les ressources gérées

## Bonus

- Consultez l'historique des synchronisations de l'Application :

  ```bash
  kubectl get applications app-blue \
    -n argocd \
    -o jsonpath='{.status.history}'
  ```

  <details>
  <summary>Sortie attendue</summary>

  ```json
  [
    {
      "deployStartedAt": "2025-01-01T10:00:00Z",
      "deployedAt": "2025-01-01T10:00:05Z",
      "id": 0,
      "revision": "abc1234...",
      "source": {
        "path": "exercises/01-argocd-basics/manifests",
        "repoURL": "https://github.com/stafyniaksacha/talk-gitops-workshop.git",
        "targetRevision": "main"
      }
    }
  ]
  ```
  </details><br>

- Forcez un refresh manuel de l'Application :

  ```bash
  kubectl annotate application app-blue -n argocd argocd.argoproj.io/refresh=hard --overwrite
  ```

  <details>
  <summary>Sortie attendue</summary>

  ```
  application.argoproj.io/app-blue annotated
  ```
  </details><br>

  Cela force ArgoCD à re-lire le dépôt Git immédiatement, sans attendre le prochain cycle de reconciliation (par défaut toutes les 3 minutes).

- Dans votre fork du dépôt, ajoutez un nouveau fichier dans le répertoire `manifests/` (par exemple un ConfigMap) :

  ```yaml
  # manifests/configmap.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-blue-config
    namespace: workshop
  data:
    ENV: "production"
  ```

  Poussez la modification sur votre branche `main` et observez ArgoCD détecter et synchroniser automatiquement la nouvelle ressource dans le cluster. C'est la puissance du GitOps : un simple `git push` suffit à déployer.

- Installez la CLI `argocd` et essayez de synchroniser l'Application avec :

  ```bash
  argocd app sync app-blue
  ```

  <details>
  <summary>Sortie attendue</summary>

  ```
  Application 'app-blue' synchronized
  ```
  </details><br>

  La CLI `argocd` offre une expérience plus riche pour interagir avec ArgoCD, notamment pour les synchronisations, rollbacks, et visualisation des ressources.

## Nettoyer

Si ce n'est pas déjà fait, supprimez l'Application :

```bash
kubectl delete -f application.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io "app-blue" deleted
```
</details><br>

Verifiez qu'il ne reste plus de ressources :

```bash
kubectl get all -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
No resources found in workshop namespace.
```
</details><br>

Pensez également à arrêter les port-forward actifs avec `Ctrl+C` dans les terminaux correspondants.
