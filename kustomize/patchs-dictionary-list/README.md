## Manipuler des clés et des listes dans des manifestes Kubernetes avec Kustomize

Kustomize permet de modifier, ajouter ou supprimer des clés et des éléments dans les fichiers de configuration Kubernetes grâce aux **patches Json6902** et aux **Strategic Merge Patches (SMP)**.  
Ces deux approches servent à adapter dynamiquement les ressources (Deployment, Service, ConfigMap, etc.) sans modifier directement les fichiers YAML d’origine.

***

### Modifier une clé dans un dictionnaire avec un patch Json6902

Pour remplacer une valeur dans un dictionnaire (par exemple un label), on utilise l’opération `replace`.

**Exemple :** changer le label `component: api` en `component: web`.

**kustomization.yaml**
```yaml
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: api-deployment
    patch: |
      - op: replace
        path: /spec/template/metadata/labels/component
        value: web
```

**Résultat :**
```yaml
labels:
  component: web
```

***

### Modifier une clé avec un Strategic Merge Patch

Même logique, mais dans un fichier séparé :

**kustomization.yaml**
```yaml
patches:
  - path: label-patch.yaml
```

**label-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        component: web
```

Kustomize fusionne ce patch avec la ressource d’origine et met à jour le label correspondant.  
Les autres clés non mentionnées sont conservées.

***

### Ajouter une clé dans un dictionnaire

Pour ajouter un nouveau label `org: kodekloud` :

**Json6902 Patch**
```yaml
- op: add
  path: /spec/template/metadata/labels/org
  value: kodekloud
```

**Strategic Merge Patch**
```yaml
spec:
  template:
    metadata:
      labels:
        org: kodekloud
```

**Résultat :**
```yaml
labels:
  component: api
  org: kodekloud
```

***

### Supprimer une clé d’un dictionnaire

Pour supprimer le label `org: kodekloud` :

**Json6902 Patch**
```yaml
- op: remove
  path: /spec/template/metadata/labels/org
```

**Strategic Merge Patch**
```yaml
spec:
  template:
    metadata:
      labels:
        org: null
```

**Résultat :**
```yaml
labels:
  component: api
```

*Remarque :* La suppression de clé avec `null` dépend du comportement du merge stratégique. La clé est supprimée uniquement si elle existe dans la ressource d’origine.

***

### Modifier un élément d’une liste (containers)

Dans un Deployment, la liste des containers se trouve sous :  
`/spec/template/spec/containers`

#### Exemple : remplacer l’image du container existant

**Json6902 Patch**
```yaml
- op: replace
  path: /spec/template/spec/containers/0
  value:
    name: haproxy
    image: haproxy:2.9
```

**Strategic Merge Patch**
```yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: haproxy:2.9
```

**Résultat :**
```yaml
containers:
  - name: nginx
    image: haproxy:2.9
```

*Important :* Le **Strategic Merge Patch** ne remplace pas tout le container, il met simplement à jour les champs mentionnés et conserve les autres (`ports`, `env`, etc.).

***

### Ajouter un élément dans une liste

Pour ajouter un container supplémentaire à la liste :

**Json6902 Patch**
```yaml
- op: add
  path: /spec/template/spec/containers/-
  value:
    name: haproxy
    image: haproxy:2.9
```

Le tiret `-` à la fin du chemin ajoute l’élément **à la fin** de la liste.

**Strategic Merge Patch**
```yaml
spec:
  template:
    spec:
      containers:
        - name: haproxy
          image: haproxy:2.9
```

**Résultat attendu :**
```yaml
containers:
  - name: nginx
    image: nginx
  - name: haproxy
    image: haproxy:2.9
```

*Remarque :* Pour éviter de remplacer un container existant, assurez-vous que le `name:` est unique.

***

### Supprimer un élément d’une liste

**Json6902 Patch**
```yaml
- op: remove
  path: /spec/template/spec/containers/1
```

Cela supprime le second élément (index 1) de la liste.

**Strategic Merge Patch**
```yaml
spec:
  template:
    spec:
      containers:
        - $patch: delete
          name: database
```

**Résultat :**
```yaml
containers:
  - name: web
    image: nginx
```

***

### Résumé concis

- **Json6902 Patch**
    - Utilise des opérations explicites (`add`, `remove`, `replace`) et des chemins JSON.
    - Chaque opération s’applique à une cible exacte.
- **Strategic Merge Patch**
    - Fusionne les champs YAML sur base de clés (comme `name`).
    - Supprime une clé avec `null` ou un élément de liste avec `$patch: delete`.
- **Cas d’usage typiques**
    - Modifier un label : `replace`
    - Ajouter un label : `add`
    - Supprimer un label : `remove`
    - Modifier un container : `/spec/template/spec/containers/0`
    - Ajouter un container : `/spec/template/spec/containers/-`
    - Supprimer un container : `$patch: delete`

Les deux types de patchs permettent de personnaliser les manifestes Kubernetes sans dupliquer ni altérer les fichiers YAML originaux.  
L’approche Strategic Merge est plus naturelle pour des modifications YAML, tandis que Json6902 convient mieux aux manipulations précises et complexes.
