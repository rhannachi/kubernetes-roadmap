### Overlay - kustomize

Dans cette section, nous allons regrouper tout ce que nous avons appris sur les fichiers `kustomization.yaml` afin de comprendre le principal cas d’usage de **Kustomize** : adapter une configuration Kubernetes de base pour différents environnements (développement, staging, production, etc.).

Kustomize permet de **personnaliser une configuration commune** sans dupliquer les fichiers YAML.\
Cette personnalisation se fait à l’aide de **overlays**, c’est-à-dire des couches spécifiques à chaque environnement.

---

### Structure d’un projet Kustomize

Un projet Kustomize typique est organisé en deux parties principales :

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

* **base/** : contient la configuration **commune** à tous les environnements (fichiers Deployment, Service, ConfigMap, etc.).
* **overlays/** : contient un dossier par environnement (`dev`, `staging`, `production`), chacun ayant ses propres ajustements (patches ou ressources supplémentaires).

---

### Exemple pratique : modification du nombre de replicas

Supposons que notre fichier de base `nginx-deployment.yaml` contienne :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

Le fichier `base/kustomization.yaml` importe simplement les ressources communes :

```yaml
resources:
  - nginx-deployment.yaml
```

---

### Overlay pour l’environnement **Dev**

Le fichier `overlays/dev/kustomization.yaml` va référencer la base et appliquer un patch :

```yaml
bases:
  - ../../base

patches:
  - patch-replicas.yaml
```

Et voici le patch :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
```

Ce patch indique à **Kustomize** de modifier le nombre de replicas uniquement pour l’environnement *Dev*.
De la même manière, on peut définir un patch spécifique pour *staging* ou *production*.

---

### Overlay pour **Production**

```yaml
bases:
  - ../../base

patches:
  - patch-replicas.yaml
```

Avec le fichier `patch-replicas.yaml` suivant :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
```

Ainsi, chaque environnement peut modifier ses paramètres sans dupliquer la configuration de base.

---

### Ajouter des ressources propres à un environnement

Les overlays peuvent aussi inclure de **nouvelles ressources** non présentes dans la base.
Par exemple, on peut ajouter un déploiement **Grafana** uniquement pour la production :

```
overlays/production/
├── grafana-depl.yaml
└── kustomization.yaml
```

`kustomization.yaml` :

```yaml
bases:
  - ../../base

resources:
  - grafana-depl.yaml

patches:
  - patch-replicas.yaml
```

Ainsi, **Grafana** sera déployé uniquement en production.

---

### Flexibilité de la structure

Kustomize offre une grande **flexibilité dans la structure des répertoires** :

* Vous pouvez organiser le dossier `base/` par fonctionnalité (`auth/`, `frontend/`, `database/`, etc.).
* Les overlays peuvent suivre une autre logique, sans nécessairement refléter la structure exacte de `base/`.
* L’important est simplement que chaque `kustomization.yaml` référence correctement les ressources dont il a besoin.

---

## Résumé concis

* **Kustomize** permet de gérer plusieurs environnements à partir d’une **base commune**.
* Les **overlays** contiennent les ajustements propres à chaque environnement (patches, nouvelles ressources).
* Le fichier `kustomization.yaml` dans chaque overlay :
    * référence la base via `bases: ../../base`
    * applique des **patches** pour modifier certains champs (ex : `replicas`)
    * peut ajouter des **ressources supplémentaires** spécifiques à l’environnement.
* La **structure du projet** est flexible, tant que les références aux bases et ressources sont correctes.

