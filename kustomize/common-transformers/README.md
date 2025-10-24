# Kustomize – Common Transformations dans Kubernetes

## Introduction

**Kustomize** est un outil intégré à Kubernetes (inclus dans `kubectl`) qui permet de personnaliser et transformer des fichiers YAML **sans les modifier directement**.  
Son principe repose sur des **Transformers**, qui appliquent des transformations globales sur un ensemble de ressources (labels, annotations, noms, etc.).

Kustomize inclut plusieurs Transformers intégrés, dont les **Common Transformations**, permettant d’ajouter des labels, annotations, préfixes, suffixes ou un namespace commun à toutes vos ressources Kubernetes.

***

## Le problème à résoudre

Imaginons que vous ayez plusieurs fichiers :

* `deployment.yaml`
* `service.yaml`

et que vous vouliez leur appliquer une configuration commune (labels, suffixes, namespace...).  
Faire cela manuellement dans chaque fichier est fastidieux et source d’erreurs. Les **Common Transformations** de Kustomize automatisent ce processus.

***

## Principe de fonctionnement

Toutes les transformations sont définies dans un seul fichier : `kustomization.yaml`.  
Lorsqu’on exécute :

```bash
kubectl apply -k .
```

Kustomize lit ce fichier, applique les transformations à toutes les ressources référencées, puis les déploie.

***

## Exemple de base : fichiers initiaux

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: nginx:latest
```

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

***

## 1. CommonLabels

### Objectif

Ajouter un ou plusieurs labels communs à **toutes les ressources**.

### `kustomization.yaml`

```yaml
resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  org: KodeKloud
  environment: dev
```

### Résultat après `kubectl kustomize .`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
    org: KodeKloud
    environment: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      org: KodeKloud
      environment: dev
  template:
    metadata:
      labels:
        app: webapp
        org: KodeKloud
        environment: dev
    spec:
      containers:
        - name: webapp
          image: nginx:latest
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  labels:
    app: webapp
    org: KodeKloud
    environment: dev
spec:
  selector:
    app: webapp
    org: KodeKloud
    environment: dev
  ports:
    - port: 80
      targetPort: 80
```

👉 Tous les objets reçoivent automatiquement les labels globaux, y compris dans les sélecteurs et templates.

***

## 2. NamePrefix et NameSuffix

### Objectif

Ajouter un **préfixe** et/ou un **suffixe** à tous les noms de ressources.

### `kustomization.yaml`

```yaml
resources:
  - deployment.yaml
  - service.yaml

namePrefix: demo-
nameSuffix: -v1
```

### Résultat après `kubectl kustomize .`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-webapp-v1
  labels:
    app: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
---
apiVersion: v1
kind: Service
metadata:
  name: demo-webapp-v1
  labels:
    app: webapp
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

👉 Très utile pour distinguer plusieurs environnements sans modifier les fichiers YAML source.

***

## 3. Namespace

### Objectif

Placer toutes les ressources dans un **namespace commun**.

### `kustomization.yaml`

```yaml
resources:
  - deployment.yaml
  - service.yaml

namespace: demo-namespace
```

### Résultat après `kubectl kustomize .`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: demo-namespace
  labels:
    app: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: demo-namespace
  labels:
    app: webapp
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

👉 Le namespace est ajouté à chaque ressource lors du rendu.

***

## 4. CommonAnnotations

### Objectif

Ajouter des **annotations globales**.

### `kustomization.yaml`

```yaml
resources:
  - deployment.yaml
  - service.yaml

commonAnnotations:
  owner: team-devops
  contact: devops@kodekloud.com
```

### Résultat après `kubectl kustomize .`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    owner: team-devops
    contact: devops@kodekloud.com
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  annotations:
    owner: team-devops
    contact: devops@kodekloud.com
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

***

## 5. Exemple complet combiné

### `kustomization.yaml`

```yaml
resources:
  - deployment.yaml
  - service.yaml

namePrefix: demo-
nameSuffix: -v1
namespace: demo-namespace

commonLabels:
  org: KodeKloud
  environment: dev

commonAnnotations:
  owner: team-devops
  contact: devops@kodekloud.com
```

### Résultat final après `kubectl kustomize .`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-webapp-v1
  namespace: demo-namespace
  labels:
    app: webapp
    org: KodeKloud
    environment: dev
  annotations:
    owner: team-devops
    contact: devops@kodekloud.com
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      org: KodeKloud
      environment: dev
  template:
    metadata:
      labels:
        app: webapp
        org: KodeKloud
        environment: dev
    spec:
      containers:
        - name: webapp
          image: nginx:latest
---
apiVersion: v1
kind: Service
metadata:
  name: demo-webapp-v1
  namespace: demo-namespace
  labels:
    app: webapp
    org: KodeKloud
    environment: dev
  annotations:
    owner: team-devops
    contact: devops@kodekloud.com
spec:
  selector:
    app: webapp
    org: KodeKloud
    environment: dev
  ports:
    - port: 80
      targetPort: 80
```

👉 Tous les labels communs se propagent automatiquement dans les sélecteurs, templates, et métadonnées, garantissant un rendu cohérent et fonctionnel.

***

## Résumé du cours

| Transformation              | Objectif                                            | Exemple de clé       |
| --------------------------- | --------------------------------------------------- | -------------------- |
| `commonLabels`              | Ajoute des labels communs à toutes les ressources   | `org: KodeKloud`     |
| `namePrefix` / `nameSuffix` | Ajoute un préfixe/suffixe à chaque nom de ressource | `demo-`, `-v1`       |
| `namespace`                 | Définit un namespace global                         | `demo-namespace`     |
| `commonAnnotations`         | Ajoute des annotations globales                     | `owner: team-devops` |

***

## Commandes utiles

Pour appliquer les transformations :

```bash
kubectl apply -k .
```

Pour visualiser le YAML transformé sans le déployer :

```bash
kubectl kustomize .
```

