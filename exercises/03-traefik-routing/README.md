# Exercice 3 -- Traefik & Routage

## Objectif

Installer Traefik via ArgoCD, configurer des IngressRoutes pour le routage par host et appliquer des Middlewares.

## Contexte

Jusqu'à présent, nous avons accédé à nos applications via `kubectl port-forward`. En production, un **reverse proxy** (ou ingress controller) route le trafic externe vers le bon service en fonction de l'URL.

**Traefik** est un reverse proxy cloud-native qui s'intègre avec Kubernetes via des **CRDs** (Custom Resource Definitions). Nous avons déjà utilisé des CRDs : l'`Application` d'ArgoCD en était une ! Traefik ajoute ses propres CRDs :

- **IngressRoute** : définit les règles de routage (quel host / path mène à quel service).
- **Middleware** : transforme les requêtes (rate limiting, ajout de headers, redirections, etc.).

Plutôt que d'utiliser la ressource native `Ingress` de Kubernetes (limitée en fonctionnalités), les `IngressRoute` de Traefik offrent plus de flexibilité et de contrôle.

## Prérequis

- Exercice 02 terminé
- Les applications app-blue et app-green déployées dans le namespace `workshop` (si ce n'est pas le cas, relancez `kubectl apply -f ../02-app-of-apps/parent-app.yaml`)

> **Note sur `*.localhost` :** sur macOS et la plupart des distributions Linux, `*.localhost` résout vers `127.0.0.1` par défaut (RFC 6761). Si cela ne fonctionne pas sur votre système, ajoutez dans `/etc/hosts` :
> ```
> 127.0.0.1 blue.localhost green.localhost traefik.localhost
> ```

## Étapes

### 1. Installer Traefik via ArgoCD

Appliquez l'Application ArgoCD qui deploie le chart Helm de Traefik :

```bash
kubectl apply -f manifests/traefik-app.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io/traefik created
```
</details><br>

ArgoCD va créer le namespace `traefik` et déployer Traefik. Attendez que le Pod soit prêt :

```bash
kubectl get pods -n traefik -w
```

<details>
<summary>Sortie attendue</summary>

```
NAME                       READY   STATUS    RESTARTS   AGE
traefik-xxxxxxxxxx-xxxxx   1/1     Running   0          45s
```
</details><br>

Appuyez sur `Ctrl+C` une fois le Pod `Running`. Vérifiez le Service :

```bash
kubectl get svc -n traefik
```

<details>
<summary>Sortie attendue</summary>

```
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
traefik   LoadBalancer   10.96.xxx.xxx   <pending>     80:3xxxx/TCP,443:3xxxx/TCP   60s
```
</details><br>

### 2. Démarrer minikube tunnel

Dans un **terminal séparé**, lancez :

```bash
minikube tunnel
```

Laissez ce terminal ouvert. Sur certains systèmes Linux, `sudo` est nécessaire. Vérifiez que l'IP externe est attribuée :

```bash
kubectl get svc -n traefik
```

<details>
<summary>Sortie attendue</summary>

```
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
traefik   LoadBalancer   10.96.xxx.xxx   127.0.0.1     80:3xxxx/TCP,443:3xxxx/TCP   2m
```
</details><br>

L'EXTERNAL-IP affiche `127.0.0.1` au lieu de `<pending>`.

### 3. Creer l'IngressRoute pour app-blue

Appliquez l'IngressRoute qui route `blue.localhost` vers app-blue :

```bash
kubectl apply -f manifests/ingressroute-blue.yaml
```

<details>
<summary>Sortie attendue</summary>

```
ingressroute.traefik.io/app-blue created
```
</details><br>

Testez la route :

```bash
curl http://blue.localhost
```

<details>
<summary>Sortie attendue</summary>

```json
{
  "hostname": "app-blue-xxxxxxxxxx-xxxxx",
  "version": "6.11.2",
  "color": "#1E90FF",
  "message": "App Blue",
  ...
}
```
</details><br>

Vous pouvez aussi ouvrir [http://blue.localhost](http://blue.localhost) dans votre navigateur pour voir l'interface bleue.

### 4. Creer l'IngressRoute pour app-green

```bash
kubectl apply -f manifests/ingressroute-green.yaml
```

<details>
<summary>Sortie attendue</summary>

```
ingressroute.traefik.io/app-green created
```
</details><br>

Testez :

```bash
curl http://green.localhost
```

<details>
<summary>Sortie attendue</summary>

```json
{
  "hostname": "app-green-xxxxxxxxxx-xxxxx",
  "version": "6.11.2",
  "color": "#2E8B57",
  "message": "App Green",
  ...
}
```
</details><br>

Deux applications, deux sous-domaines, une seule instance Traefik.

### 6. Ajouter un Middleware de rate-limiting

Appliquez le Middleware :

```bash
kubectl apply -f manifests/middleware-ratelimit.yaml
```

<details>
<summary>Sortie attendue</summary>

```
middleware.traefik.io/rate-limit created
```
</details><br>

Éditez `manifests/ingressroute-green.yaml` et décommentez la section middlewares (supprimez les `#` devant `middlewares:` et `- name: rate-limit`). La section `routes` doit ressembler a :

```yaml
  routes:
    - match: Host(`green.localhost`)
      kind: Rule
      middlewares:
        - name: rate-limit
      services:
        - name: app-green
          port: 9898
```

Appliquez la modification :

```bash
kubectl apply -f manifests/ingressroute-green.yaml
```

<details>
<summary>Sortie attendue</summary>

```
ingressroute.traefik.io/app-green configured
```
</details><br>

Testez le rate limiting en envoyant des requetes rapides :

```bash
for i in $(seq 1 30); do
  curl -s -o /dev/null -w "%{http_code}\n" http://green.localhost
done
```

<details>
<summary>Sortie attendue</summary>

```
200
200
... (les ~20 premieres requetes retournent 200)
429
429
... (les requetes suivantes retournent 429)
```
</details><br>

### 7. Acceder au dashboard Traefik

Ouvrez [http://traefik.localhost/dashboard/](http://traefik.localhost/dashboard/) dans votre navigateur, ou testez via curl :

```bash
curl -s http://traefik.localhost/dashboard/ | head -3
```

> **Attention** : n'oubliez pas le `/` final dans l'URL (`/dashboard/` et non `/dashboard`).

Le dashboard affiche les **Routers** (nos IngressRoutes), les **Services** backends et les **Middlewares** appliques.

### 8. Lister les CRDs Traefik

```bash
kubectl get crds | grep traefik
```

<details>
<summary>Sortie attendue</summary>

```
ingressroutes.traefik.io       ...
middlewares.traefik.io          ...
ingressroutetcps.traefik.io    ...
tlsoptions.traefik.io          ...
traefikservices.traefik.io     ...
...
```
</details><br>

Traefik étend Kubernetes avec de nombreux types de ressources personnalisées (routage HTTP/TCP/UDP, middlewares, TLS, etc.).

## Vérification

- [ ] Traefik est déployé via ArgoCD dans le namespace `traefik`
- [ ] `minikube tunnel` est actif et le service Traefik a une EXTERNAL-IP
- [ ] `curl http://blue.localhost` affiche l'application bleue
- [ ] `curl http://green.localhost` affiche l'application verte
- [ ] Le rate limiting fonctionne (code 429 après dépassement du seuil)
- [ ] Le dashboard Traefik est accessible sur [http://traefik.localhost/dashboard/](http://traefik.localhost/dashboard/)

## Bonus

- Créez une IngressRoute pour app-nginx sur `nginx.localhost` pointant vers le service `app-nginx` sur le port `80`. Inspirez-vous de `manifests/ingressroute-blue.yaml` en changeant le host, le service et le port.

- Combinez plusieurs middlewares sur une même route. Créez un middleware `custom-headers` (type `headers`, `customResponseHeaders`) qui ajoute `X-Workshop: gitops-routing`. Testez avec `curl -v http://blue.localhost`.

- Testez le routage par path prefix au lieu du host avec la règle :
  ```yaml
  match: Host(`apps.localhost`) && PathPrefix(`/blue`)
  ```

- Au lieu d'appliquer les manifests un par un, mettez à jour l'Application ArgoCD `workshop-apps` de l'exercice 02 pour inclure tous les manifests de ce dossier. Appliquez ensuite la modification en relançant `kubectl apply -f ../02-app-of-apps/parent-app.yaml`.

## Nettoyer

```bash
kubectl delete -f manifests/ -n workshop
```

<details>
<summary>Sortie attendue</summary>

```
deployment.apps "app-blue" deleted
service "app-blue" deleted
...
middleware.traefik.io "rate-limit" deleted
```
</details><br>

```bash
kubectl delete -f manifests/traefik-app.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io "traefik" deleted
```
</details><br>

```bash
kubectl delete -f ../02-app-of-apps/parent-app.yaml
```

<details>
<summary>Sortie attendue</summary>

```
application.argoproj.io "workshop-apps" deleted
```
</details><br>

Verifiez qu'il ne reste plus de ressources :

```bash
kubectl get all -n workshop && kubectl get all -n traefik
```

<details>
<summary>Sortie attendue</summary>

```
No resources found in workshop namespace.
No resources found in traefik namespace.
```
</details><br>

Arretez `minikube tunnel` avec `Ctrl+C` dans le terminal correspondant.
