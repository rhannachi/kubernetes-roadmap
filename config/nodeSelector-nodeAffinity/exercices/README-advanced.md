### 1. Comment écrire une règle nodeAffinity qui impose qu’un Pod soit planifié sur un Node avec soit `size=large` ou `size=medium`, **mais jamais** sur un Node avec `size=small` ?

Pour ce scénario, on combine une règle obligatoire qui accepte `size` dans `[large, medium]` et une exclusion explicite du `size=small` n’est pas nécessaire car on ne sélectionne que large ou medium avec `In`, donc la taille small est automatiquement exclue.

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: size
      operator: In
      values:
      - large
      - medium
```

***

### 2. Comment combiner plusieurs clés dans une même règle nodeAffinity pour exiger qu’un Node ait **à la fois** un label `gpu=nvidia` **et** un label `disktype=ssd` ?

Ici, les deux conditions sont combinées dans un même élément `matchExpressions`, ce qui implique un ET logique :

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: gpu
      operator: In
      values:
      - nvidia
    - key: disktype
      operator: In
      values:
      - ssd
```

***

### 3. Comment formuler une règle qui accepte un Pod sur un Node possédant un label `zone` dans ["us-east1", "us-west1"], **et** que ce Node ne doit pas avoir le label `maintenance=true` ?

Utilisation de `In` pour la zone, et `NotIn` pour exclure les Nodes en maintenance :

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: zone
      operator: In
      values:
      - us-east1
      - us-west1
    - key: maintenance
      operator: NotIn
      values:
      - "true"
```

***

### 4. Comment utiliser nodeAffinity pour planifier un Pod sur n’importe quel Node **qui possède** un label `environment` quelle que soit sa valeur ?

On utilise l’opérateur `Exists` sur la clé `environment` :

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: environment
      operator: Exists
```

***

### 5. Comment exprimer une préférence avec nodeAffinity préférentielle (preferredDuringSchedulingIgnoredDuringExecution) pour des Nodes avec le label `disktype=ssd` mais accepter d’autres Nodes si besoin ?

On utilise `preferredDuringSchedulingIgnoredDuringExecution` avec un poids :

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 100
  preference:
    matchExpressions:
    - key: disktype
      operator: In
      values:
      - ssd
```

***

### 6. Comment écrire une règle nodeAffinity qui interdit explicitement la planification du Pod sur des Nodes avec le label `type=spot` (exclure ces Nodes) ?

On utilise `NotIn` pour exclure les Nodes avec `type=spot` :

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: type
      operator: NotIn
      values:
      - spot
```

***

### 7. Comment définir plusieurs termes nodeSelectorTerms avec des règles alternatives (OU) ? Par exemple, planifier un Pod soit sur un Node avec `region=eu` **ou** `region=us`, avec en plus une contrainte facultative sur `disktype=ssd`.

Pour l’OU, on met plusieurs `nodeSelectorTerms` — chacun représente un terme au sens OU :

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: region
      operator: In
      values:
      - eu
  - matchExpressions:
    - key: region
      operator: In
      values:
      - us
```

La contrainte facultative peut être ajoutée sous forme de `preferredDuringScheduling` pour `disktype=ssd`.

***

### 8. Comment combiner nodeAffinity obligatoire pour un label, et en même temps une nodeAffinity préférentielle pour un autre label ?

Dans le même manifeste YAML, on sépare la section `requiredDuringSchedulingIgnoredDuringExecution` pour la contrainte stricte et `preferredDuringSchedulingIgnoredDuringExecution` pour la préférence :

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: gpu
          operator: In
          values:
          - nvidia
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 50
      preference:
        matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

***

### 9. Comment gérer un scénario où un Pod doit préférentiellement être planifié sur un Node avec `gpu=amd` **ou** `gpu=nvidia`, mais absolument ne jamais sur un Node sans GPU ?

On combine une règle obligatoire qui exige que la clé `gpu` existe (excluant les Nodes sans GPU), et une règle préférentielle pour les valeurs `amd` et `nvidia` :

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: gpu
          operator: Exists
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: gpu
          operator: In
          values:
          - amd
          - nvidia
```

***

### 10. Comment écrire un manifeste YAML avec nodeAffinity combinant des expressions `In`, `NotIn` et `Exists` pour un contexte dynamique ?

Exemple combinant toutes ces expressions dans une seule règle obligatoire :

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: environment
      operator: Exists
    - key: region
      operator: In
      values:
      - eu
      - us
    - key: maintenance
      operator: NotIn
      values:
      - "true"
```
