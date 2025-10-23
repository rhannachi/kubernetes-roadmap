### Comprendre et utiliser Kustomize

Kustomize est un outil intégré à `kubectl` qui permet de gérer et personnaliser les configurations Kubernetes sans dupliquer le code YAML.\
L’objectif principal est de **réutiliser les manifests existants** tout en adaptant certains paramètres à chaque environnement (dev, staging, production).

### Le problème initial

Imaginons un simple fichier `Deployment` pour Nginx :

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

On souhaite utiliser ce `Deployment` pour trois environnements :
- **development** : 1 replica
- **staging** : 2 replicas
- **production** : 5 replicas

Une première solution consisterait à créer trois répertoires :
```
/dev
/staging
/production
```
et copier le manifest dans chacun, en modifiant simplement la valeur `replicas`.  
Bien que fonctionnelle, cette approche entraîne une **duplication du code**, difficile à maintenir : chaque changement (ajout d’un Service, modification d’une image, etc.) doit être répliqué partout.

***

### La solution apportée par Kustomize

Kustomize résout ce problème grâce à deux concepts fondamentaux : **base** et **overlay**.

#### Base
Le dossier `base` contient les ressources communes à tous les environnements :
```plaintext
kustomize-example/
└── base/
    ├── deployment.yaml
    └── kustomization.yaml
```

Exemple du `kustomization.yaml` dans le dossier base :
```yaml
resources:
  - deployment.yaml
```

Le fichier `deployment.yaml` définit la configuration par défaut (ici, `replicas: 1`).

#### Overlays
Les **overlays** permettent de surcharger la configuration du **base** pour un environnement spécifique sans tout recopier.

Arborescence :
```plaintext
kustomize-example/
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

Exemple pour **staging** :
```yaml
# overlays/staging/kustomization.yaml
resources:
  - ../../base
patches:
  - path: patch.yaml
```

Et le fichier `patch.yaml` :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
```

Pour **production**, on aurait simplement :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 5
```

***

### Génération et application

Kustomize combine automatiquement la base et les overlays pour générer les manifestes finaux.

Pour voir le résultat :
```bash
kubectl kustomize overlays/staging
```

Pour appliquer les changements au cluster :
```bash
kubectl apply -k overlays/production
```

***

### Avantages de Kustomize
- **Zéro duplication** : un seul fichier de base partagé.
- **Maintenance simplifiée** : une modification est automatiquement reflétée dans tous les environnements.
- **Interopérabilité** : tout reste en YAML standard.
- **Pas de langage de template** : apprentissage rapide comparé à Helm.

***

### Résumé essentiel

| Concept | Description | Exemple |
|----------|--------------|----------|
| Base | Contient les ressources communes et valeurs par défaut | `deployment.yaml`, `kustomization.yaml` |
| Overlay | Contient les différences par environnement | `replicas`, `imageTag`, etc. |
| Commande | Génère et applique les manifests fusionnés | `kubectl apply -k overlays/staging` |

***

