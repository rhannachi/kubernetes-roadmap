###  Exercice 1 — Modifier un label avec `Json6902`

**Objectif :**  
Changer le label `tier: backend` en `tier: web` dans un `Deployment` existant.

**Fichier de base (`deployment.yaml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      labels:
        tier: backend
```

**Question :**  
Que faut-il ajouter dans `kustomization.yaml` pour appliquer ce changement sans modifier le YAML original ?

***

** Correction :**

**kustomization.yaml**
```yaml
resources:
  - deployment.yaml

patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/template/metadata/labels/tier
        value: web
```

**Résultat final appliqué par Kustomize :**
```yaml
labels:
  tier: web
```

***

###  Exercice 2 — Ajouter un label avec `Strategic Merge Patch`

**Objectif :**  
Ajouter un label `env: prod` au template du `Deployment` `myapp`.

**Fichier de base (`deployment.yaml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      labels:
        app: demo
```

**Question :**  
Crée un patch SMP pour ajouter ce label.

***

** Correction :**

**kustomization.yaml**
```yaml
resources:
  - deployment.yaml
patches:
  - path: label-patch.yaml
```

**label-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      labels:
        env: prod
```

**Résultat attendu :**
```yaml
labels:
  app: demo
  env: prod
```

***

###  Exercice 3 — Remplacer une image dans un container

**Objectif :**  
Modifier le container nommé `nginx` pour utiliser l’image `nginx:1.27-alpine`.

**Fichier de base (`deployment.yaml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

**Question :**  
Crée un patch pour changer l’image sans toucher le reste.

***

** Correction avec `Strategic Merge Patch` :**

**image-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
```

**kustomization.yaml**
```yaml
resources:
  - deployment.yaml
patches:
  - path: image-patch.yaml
```

**Résultat :**
```yaml
containers:
  - name: nginx
    image: nginx:1.27-alpine
```

***

###  Exercice 4 — Supprimer un container avec `$patch: delete`

**Objectif :**  
Supprimer le container nommé `sidecar` d’un `Deployment`.

**Fichier de base (`app.yaml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-container
spec:
  template:
    spec:
      containers:
        - name: main
          image: busybox
        - name: sidecar
          image: alpine
```

**Question :**  
Comment créer un patch pour ne garder que `main` ?

***

** Correction :**

**delete-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-container
spec:
  template:
    spec:
      containers:
        - $patch: delete
          name: sidecar
```

**kustomization.yaml**
```yaml
resources:
  - app.yaml
patches:
  - path: delete-patch.yaml
```

**Résultat :**
```yaml
containers:
  - name: main
    image: busybox
```

***

###  Exercice 5 — Ajouter un container via `Json6902`

**Objectif :**  
Ajouter un container `debug` à un `Deployment` sans modifier le fichier d’origine.

**Fichier de base (`debug-app.yaml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debug-deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: nginx
```

**Question :**  
Comment écrire un patch JSON pour ajouter ce container ?

***

** Correction :**

**kustomization.yaml**
```yaml
resources:
  - debug-app.yaml

patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: debug-deployment
    patch: |
      - op: add
        path: /spec/template/spec/containers/-
        value:
          name: debug
          image: busybox
          args: ["sleep", "3600"]
```

**Résultat final :**
```yaml
containers:
  - name: app
    image: nginx
  - name: debug
    image: busybox
    args: ["sleep", "3600"]
```


