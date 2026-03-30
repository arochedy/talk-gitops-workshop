# Workshop GitOps & Routing

> Workshop hands-on d'une demi-journee (~3h30-4h) pour apprendre ArgoCD, le GitOps et le routage avec Traefik.

Public cible : développeurs ayant suivi le [Workshop 1 (Kubernetes & Helm)](https://github.com/stafyniaksacha/talk-k8s-workshop) ou disposant de connaissances equivalentes en kubectl, Deployments, Services et Helm.

---

## Description

Ce workshop a pour objectif de faire découvrir le GitOps et le routage HTTP aux développeurs qui ont déjà une première expérience avec Kubernetes. À travers des exercices pratiques progressifs, les participants apprendront à :

- Comprendre le principe GitOps et la différence entre déploiement push et pull
- éployer et gérer des applications avec ArgoCD (sync, drift, self-heal, prune)
- gérer plusieurs applications avec le pattern App of Apps
- Comprendre le concept de CRD (Custom Resource Definition) et l'extensibilité de Kubernetes
- Configurer le routage HTTP avec Traefik (IngressRoute, Middleware)

L'application de démonstration utilisée tout au long du workshop est [podinfo](https://github.com/stefanprodan/podinfo), une application web légère conçue pour tester des fonctionnalités Kubernetes.

---

## Prérequis

Assurez-vous d'avoir les outils suivants installés sur votre machine avant le workshop.

| Outil | Installation (macOS) | Installation (autres) |
|-------|----------------------|-----------------------|
| Docker Desktop | [docker.com/get-started](https://www.docker.com/get-started/) | [docker.com/get-started](https://www.docker.com/get-started/) |
| minikube | `brew install minikube` | [minikube.sigs.k8s.io/docs/start](https://minikube.sigs.k8s.io/docs/start/) |
| kubectl | `brew install kubectl` | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools/) |
| Helm | `brew install helm` | [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/) |
| pnpm (slides uniquement) | `brew install pnpm` | [pnpm.io/installation](https://pnpm.io/installation) |

Vérification rapide :

```bash
docker --version
minikube version
kubectl version --client
helm version
```

---

## Structure du projet

```
talk-gitops-workshop/
├── exercises/
│   ├── 00-setup/                  # Mise en place de l'environnement
│   ├── 01-argocd-basics/          # ArgoCD : deployer via Git
│   │   └── manifests/
│   ├── 02-app-of-apps/            # Pattern App of Apps
│   │   ├── demo-apps/
│   │   └── argocd-apps/
│   └── 03-traefik-routing/        # Traefik : routage et middlewares
│       └── manifests/
├── slides/
│   └── public/
├── cheatsheet.md
└── README.md
```

---

## Liens utiles

- [Workshop 1 -- Kubernetes & Helm](https://github.com/stafyniaksacha/talk-k8s-workshop)
- [Documentation ArgoCD](https://argo-cd.readthedocs.io/)
- [Documentation Traefik](https://doc.traefik.io/traefik/)
- [Documentation Kubernetes](https://kubernetes.io/docs/home/)
- [Documentation Helm](https://helm.sh/docs/)
- [podinfo -- Application de démonstration](https://github.com/stefanprodan/podinfo)
  