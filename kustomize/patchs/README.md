### Les *patches* dans Kustomize (Kubernetes)

Les *patches* dans Kustomize permettent de modifier de manière ciblée la configuration d’un ou plusieurs objets Kubernetes.\
Contrairement aux *common transformers* qui appliquent des changements globaux (par exemple ajouter un label à tous les objets du Cluster), les *patches* offrent un contrôle précis, limité à certaines ressources spécifiques.

---

### Types d’opérations dans un patch

Un patch définit l’opération à effectuer via le champ `op`. Les trois opérations les plus courantes sont :

* **add** : ajouter un nouvel élément (ex. ajouter un container à une liste de containers dans un *Deployment*).
* **remove** : supprimer un élément existant (ex. retirer un label ou un container d’un *Pod*).
* **replace** : remplacer la valeur actuelle par une nouvelle. Exemple : changer le nombre de *replicas* d’une valeur de `1` à `5`.

Chaque patch précise aussi une **cible (target)**, c’est-à-dire le ou les objets Kubernetes concernés, grâce à des critères tels que :

* `kind` (ex. *Deployment*)
* `name` (nom exact de l’objet)
* `namespace`, `labelSelector` ou `annotationSelector`

Enfin, la propriété **value** décrit la valeur à ajouter ou à remplacer. Pour une opération de type *remove*, aucune valeur n’est nécessaire.

---

### Exemple 1 – Modifier le champ *name* d’un Deployment

Voici un extrait du fichier `kustomization.yaml` permettant de changer le nom d’un *Deployment* :

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |
      - op: replace
        path: /metadata/name
        value: web-deployment
```

Dans cet exemple :

* Le patch cible le *Deployment* nommé `api-deployment`.
* L’opération *replace* modifie le champ `/metadata/name` pour devenir `web-deployment`.

---

### Exemple 2 – Modifier le nombre de *replicas* d’un Deployment

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |
      - op: replace
        path: /spec/replicas
        value: 5
```

Ce patch met simplement à jour la valeur du champ `replicas` (passant de 1 à 5).

---

### Types de patches supportés dans Kustomize

Kustomize supporte deux formats de patch principaux :

#### 1. *JSON 6902 Patch*

C’est le format utilisé dans les exemples précédents. Il définit de manière explicite :

* la cible (target)
* la liste des opérations (`op`, `path`, `value`)

Ce format suit la spécification RFC 6902 et permet un contrôle précis des modifications.

#### 2. *Strategic Merge Patch*

Cette approche utilise directement la syntaxe native des fichiers Kubernetes YAML.
Elle consiste à copier la ressource originale et à ne conserver que les parties à modifier.

**Exemple :**

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: api-deployment
      spec:
        replicas: 5
```

Kustomize fusionne cette configuration avec la ressource d’origine, mettant à jour uniquement les propriétés modifiées (`replicas` ici).

---

### Patches dans un fichier séparé

Dans les projets plus complexes, il est souvent préférable de stocker les *patches* dans des fichiers distincts plutôt que directement dans le `kustomization.yaml`.\
Cela permet de **mieux organiser** les modifications et de **réutiliser** certains patches dans plusieurs environnements (*dev*, *staging*, *prod*…).

#### Exemple : *patches*

Fichier `patch-replicas.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 5
```

Fichier `kustomization.yaml` :

```yaml
resources:
  - deployment.yaml

patches:
  - path: patch-replicas.yaml
```

Ici, `kustomize` appliquera le patch défini dans `patch-replicas.yaml` à la ressource `api-deployment`.\
Cette approche est plus claire, favorise la modularité du code et reste compatible avec la syntaxe YAML standard de Kubernetes.

---

### Bonnes pratiques

* Utiliser un *common transformer* pour appliquer des changements globaux.
* Utiliser un *patch* pour ajuster un objet spécifique.
* Préférer le *Strategic Merge Patch* pour sa lisibilité, notamment lors de larges déploiements.
* Stocker les *patches* dans des fichiers séparés pour favoriser la réutilisation et la maintenance.
* Toujours tester les *patches* localement avec `kubectl kustomize` avant de les appliquer via `kubectl apply -k`.

---

