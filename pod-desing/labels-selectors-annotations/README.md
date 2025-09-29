# Labels, Selectors and Annotations

## Labels & Selectors

Les labels sont des propriétés associées à chaque objet Kubernetes, sous forme de paires clé/valeur.\
Ils permettent de classifier et d’organiser les objets selon différents critères, comme l’application, la fonctionnalité ou tout autre attribut pertinent.

Les selectors, eux, servent à filtrer ces objets en fonction des labels définis. Par exemple :
- `class=mammal` permet d’identifier tous les objets portant ce label.
- `color=green` permet de sélectionner les objets dont la couleur est verte.  
  En combinant plusieurs conditions, il devient possible de cibler précisément un sous-ensemble d’objets.

Ce principe est similaire aux “tags” utilisés dans d’autres contextes (articles, vidéos, boutiques en ligne), qui permettent de regrouper et de retrouver facilement les éléments.

***

### Exemple 1 : Labels sur un Pod
Dans un fichier de définition de Pod, les labels se définissent sous `metadata.labels` comme suit :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend
    tier: web
spec:
  containers:
  - name: nginx-container
    image: nginx:1.21
```

Après création, on peut sélectionner ce Pod avec :
```
$ kubectl get pods --selector app=frontend
$ kubectl get pods --selector app=frontend,tier=web
```

On peut également utiliser les **labels** et **selectors** pour filtrer des objets Kubernetes (Pod, ReplicaSet, Deployment, Service, etc.) en fonction de leurs labels.\
Par exemple, pour lister tous les objets Kubernetes dans le **Cluster** ayant le label `env=prod` dans le **namespace** `default`:
```
$ kubectl get all -n default --selector env=prod
```

***

### Exemple 2 : ReplicaSet utilisant des selectors
Lorsqu’un **ReplicaSet** est créé, il utilise un selector pour gérer des Pods qui possèdent certains labels.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend-replicaset
  labels:
    app: frontend-rep   # Labels du ReplicaSet lui-même
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend   # Sélection des Pods à gérer
      tier: web
  template:
    metadata:
      labels:
        app: frontend   # Labels appliqués aux Pods
        tier: web
        bla: bla
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.21
```

Ici :
- Les Pods créés auront les labels `app=frontend`, `bla=bla` et `tier=web`.
- Le ReplicaSet utilise `spec.selector.matchLabels` pour découvrir et gérer automatiquement ces Pods.

***

### Exemple 3 : Service filtrant les Pods par label
Un **Service** peut utiliser un selector pour envoyer le trafic vers un certain groupe de Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
    tier: web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Ce Service enverra le trafic vers tous les Pods qui possèdent :
```yaml
labels:
  app: frontend
  tier: web
```

***
## Annotations

### Exemple 1 : Utilisation des annotations
Contrairement aux labels, les **annotations** ajoutent des métadonnées sans impact sur la sélection ou le filtrage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotated-pod
  labels:
    app: backend
  annotations:
    maintainer: "dev-team@example.com"
    git-commit-hash: "a1b2c3d4"
    build-version: "1.0.5"
spec:
  containers:
  - name: backend-container
    image: busybox
    command: ["sh", "-c", "echo Hello Kubernetes && sleep 3600"]
```

Ici, les annotations fournissent des informations utiles (contact, version, build) mais ne servent pas à filtrer ou grouper les objets.

***

## Résumé concis

- **Labels** : paires clé/valeur (ex. `app=frontend`) utilisées pour classifier les objets.
- **Selectors** : permettent de filtrer les objets (`kubectl get pods --selector app=frontend`).
- **ReplicaSet** : utilise les selectors pour gérer les Pods correspondant aux labels définis.
- **Service** : utilise les selectors pour router le trafic vers les Pods correspondants.
- **Annotations** : ajoutent des métadonnées informatives (versions, contact, outils), sans rôle dans la sélection.


