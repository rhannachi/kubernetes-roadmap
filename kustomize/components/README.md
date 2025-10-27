# Les Components dans Kustomize

### Introduction

Dans cette section, nous allons découvrir une fonctionnalité avancée de **Kustomize** : les **components**.\
Un *component* est un bloc réutilisable de configuration Kubernetes, permettant de définir une logique spécifique que l’on peut activer dans plusieurs **overlays** sans dupliquer le code.

Les components sont particulièrement utiles lorsque votre application comporte des **fonctionnalités optionnelles**, activées uniquement dans certaines versions du déploiement (certains overlays).\
Par exemple : une fonctionnalité de **caching** activée uniquement dans les environnements *premium* et *self-hosted*, mais pas dans *development*.

---

## Pourquoi utiliser des Components ?

Si une même configuration doit être appliquée à **tous les overlays**, elle peut simplement être placée dans le dossier **base/**.
Mais si une fonctionnalité ne concerne qu’un **sous-ensemble d’overlays**, alors la placer dans le *base* n’est pas approprié.

Sans les components, il faudrait **copier-coller** la configuration dans chaque overlay concerné.
Cela pose plusieurs problèmes :

* Duplication du code
* Risque d’oublier des modifications dans certains fichiers
* Difficulté de maintenance à long terme

Les **components** résolvent ce problème :
ils regroupent toutes les ressources Kubernetes nécessaires à une fonctionnalité donnée dans un seul dossier, réutilisable dans plusieurs overlays via une simple ligne de configuration.

---

## Exemple de Structure de Projet

Imaginons une application déployée sous trois variantes :

* `development/`
* `premium/`
* `self-hosted/`

Ainsi que la base commune :

* `base/`

Nous allons également ajouter un dossier `components/` pour nos fonctionnalités optionnelles.

### Arborescence complète :

```
my-app/
├── base/
│   ├── api-depl.yaml
│   └── kustomization.yaml
│
├── components/
│   ├── caching/
│   │   ├── redis-depl.yaml
│   │   ├── redis-secret.yaml
│   │   └── kustomization.yaml
│   │
│   └── database/
│       ├── postgres-depl.yaml
│       ├── deployment-patch.yaml
│       └── kustomization.yaml
│
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── premium/
│   │   └── kustomization.yaml
│   └── self-hosted/
│       └── kustomization.yaml
```

---

## Exemple de Component : Database

Prenons la fonctionnalité **“external database”**, utilisée seulement par les environnements `dev` et `premium`.

### Fichier `components/database/postgres-depl.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
```

### Fichier `components/database/deployment-patch.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
        - name: api
          env:
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
```

### Fichier `components/database/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - postgres-depl.yaml

patches:
  - path: deployment-patch.yaml

# Le champ "secretGenerator" dans un fichier kustomization.yaml permet à Kustomize de générer automatiquement un objet Secret Kubernetes à partir de valeurs fournies dans le code
secretGenerator:
  - name: db-secret
    literals:
      - password=mysecurepassword
```

> **Remarque :**
> La différence principale avec une configuration classique est le champ :
>
> ```yaml
> kind: Component
> ```
>
> au lieu de `kind: Kustomization`.

---

## Utilisation du Component dans un Overlay

Prenons l’exemple de l’environnement `dev`, qui a besoin de la base de données externe.

### Fichier `overlays/dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

components:
  - ../../components/database
```

> Ici, nous importons la configuration de base **et** le component `database`.
> Le même principe peut s’appliquer à d’autres overlays comme `premium`.

---

## Autre Exemple : Caching

La fonctionnalité **caching** (avec **Redis**) pourrait être activée uniquement dans `premium` et `self-hosted`.

### Fichier `overlays/premium/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

components:
  - ../../components/caching
  - ../../components/database
```

---

## Avantages des Components

* **Réutilisables** : une seule définition pour plusieurs overlays
* **Faciles à maintenir** : modification centralisée
* **Flexibles** : activation/désactivation selon les besoins
* **Évitent le “config drift”** entre les environnements

---

## Résumé concis

| Concept                    | Description                                                                                     |
| -------------------------- | ----------------------------------------------------------------------------------------------- |
| **Component**              | Bloc réutilisable de configuration Kubernetes (ConfigMaps, Secrets, Deployments, Patches, etc.) |
| **Utilisation**            | Pour des fonctionnalités optionnelles à activer dans certains overlays                          |
| **Fichiers clés**          | `kustomization.yaml` (avec `kind: Component`), fichiers de ressources et patches                |
| **Import dans un overlay** | Ajout via le champ `components:` dans le `kustomization.yaml`                                   |
| **Exemples**               | `caching` (Redis), `database` (Postgres)                                                        |

