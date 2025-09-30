# Strategy Deployment

En Kubernetes, plusieurs stratégies de déploiement permettent de mettre à jour une application tout en maîtrisant les risques liés aux interruptions de service.\
Les plus courantes sont **Recreate**, **RollingUpdate** (par défaut), ainsi que les stratégies alternatives **Blue-Green** et **Canary**.

## Stratégie Recreate
Avec la stratégie **Recreate**, Kubernetes supprime d’abord **tous les Pods existants** avant de créer les nouveaux.  
L’inconvénient majeur est que l’application devient indisponible pendant la transition, car aucun Pod n’est disponible jusqu’au lancement des nouveaux.

**Exemple :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:2.0
```

Dans ce cas, tous les Pods en **version 1.0** sont supprimés avant que la **version 2.0** ne soit déployée.

***

## Stratégie RollingUpdate
La stratégie **RollingUpdate** est la méthode par défaut. Les Pods sont remplacés un par un, ce qui permet d’assurer une continuité de service.  
L’application reste accessible, et le déploiement est progressif.

**Exemple :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:2.0
```

Ici, Kubernetes remplace progressivement les Pods : pour chaque Pod supprimé, un nouveau Pod est créé.

***

## Stratégie Blue-Green
La stratégie **Blue-Green** repose sur l’existence de deux environnements distincts :
- **Blue** : version actuelle de l’application (par ex., `v1`).
- **Green** : nouvelle version déployée en parallèle (par ex., `v2`).

Pendant une phase de tests, seul **Blue** reçoit le trafic. Une fois les validations terminées, on bascule le trafic du **Service** vers **Green**. Cette approche permet un retour rapide en cas de problème (rollback).

**Exemple :**
```yaml
# Déploiement Blue (version v1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# Déploiement Green (version v2)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: myapp:2.0

---
# Service basculant entre v1 et v2
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: v1   # initialement vers Blue
  ports:
  - port: 80
    targetPort: 80
```

Pour basculer tout le trafic vers **Green** :
```yaml
spec:
  selector:
    app: myapp
    version: v2
```

***

## Stratégie Canary
La stratégie **Canary** consiste à déployer une nouvelle version (ex. v2) en parallèle de l’ancienne (v1), mais à recevoir uniquement une **petite portion du trafic**.  
Cela permet de tester l’application avec de vrais utilisateurs tout en limitant les risques. Ensuite, si tout fonctionne, on augmente progressivement le nombre de Pods en v2 jusqu’à faire disparaître v1.

**Exemple simplifié :**
```yaml
# Déploiement v1 (stable)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# Déploiement v2 (canary, peu de Pods)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1   # seulement 1 pod en "canary"
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: myapp:2.0

---
# Service distribuant le trafic entre v1 et v2
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
```

Ici, Kubernetes envoie une petite partie du trafic vers v2 (car il n’y a qu’un seul Pod v2 sur plusieurs Pods v1). Au fil du temps, on augmente le nombre de Pods v2.

***

## Résumé concis

- **Recreate** : supprime tous les Pods existants avant de créer la nouvelle version → indisponibilité temporaire.
- **RollingUpdate (par défaut)** : remplace les Pods un par un → pas d’interruption de service.
- **Blue-Green** : déploie une nouvelle version en parallèle (Green), puis bascule le trafic du Service de Blue vers Green → rollback rapide possible.
- **Canary** : déploie une nouvelle version avec un petit nombre de Pods, reçoit seulement une fraction du trafic → permet des tests progressifs en production.

***
