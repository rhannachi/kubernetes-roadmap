```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp-container
          # Versions possibles : 1.27 → 1.28 → 1.29
          image: nginx:1.27
          ports:
            - containerPort: 80
```

***

### Explications du fichier

- Ce **Deployment** crée 3 réplicas de Pods utilisant l’image `nginx:1.27`.
- La stratégie de déploiement par défaut est **RollingUpdate**, donc les Pods sont remplacés progressivement.
- Le label `app: webapp` sert à associer les Pods au ReplicaSet et au Deployment.
- Le conteneur expose le port 80.

***

### Commandes utiles

#### 1. Créer le déploiement avec nginx:1.27
```bash
kubectl apply -f deployment.yaml
kubectl get rs
kubectl get pods
```

Exemple de sortie :
```
NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/my-webapp-5476f7878   3         3         3       5s
```
```
NAME                        READY   STATUS    RESTARTS   AGE
my-webapp-5476f7878-4dr4t   1/1     Running   0          21s
my-webapp-5476f7878-r94dj   1/1     Running   0          21s
my-webapp-5476f7878-t67ms   1/1     Running   0          21s
```

Chaque Pod fait partie du ReplicaSet `my-webapp-5476f7878`.

***

#### 2. Vérifier le statut du déploiement
```bash
kubectl rollout status deployment/my-webapp
```

***

#### 3. Vérifier l’historique des révisions
```bash
kubectl rollout history deployment/my-webapp
```

***

#### 4. Mettre à jour l’image vers nginx:1.28
```bash
kubectl set image deployment/my-webapp webapp-container=nginx:1.28
kubectl get rs
kubectl get pods
```

Un nouveau ReplicaSet est créé, par exemple `my-webapp-7957c57db5`, et de nouveaux Pods associés apparaissent.  
On peut constater le lien entre les Pods et leur ReplicaSet via le suffixe du nom.

***

#### 5. Vérifier les révisions détaillées
```bash
kubectl rollout history deployment/my-webapp --revision=2
kubectl rollout history deployment/my-webapp --revision=3
```

Ces commandes affichent les différences de template entre chaque version (notamment le changement de l’image).

***

#### 6. Mettre à jour l’image vers nginx:1.29
```bash
kubectl set image deployment/my-webapp webapp-container=nginx:1.29
kubectl get rs
kubectl get pods
```

Un ReplicaSet supplémentaire est créé (`my-webapp-6d9b676ffb`) et devient actif.

***

#### 7. Faire un rollback vers la révision précédente (nginx:1.28)
```bash
kubectl rollout undo deployment/my-webapp
kubectl get rs
kubectl get pods
```

Le ReplicaSet `my-webapp-7957c57db5` est réactivé avec ses Pods correspondants.

***

#### 8. Rollback vers la première version (nginx:1.27)
```bash
kubectl rollout undo deployment/my-webapp --to-revision=1
kubectl get rs
kubectl get pods
```

Le ReplicaSet `my-webapp-5476f7878` redevient actif.

***
