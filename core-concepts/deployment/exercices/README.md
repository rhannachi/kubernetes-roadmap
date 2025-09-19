# Exercices :

**1. Créer un Deployment basique**\
Rédige un manifeste YAML pour déployer une image nginx avec 3 réplicas et applique-le avec `kubectl apply -f`.

**2. Vérifier l’état du Deployment**\
Utilise `kubectl get deployments`, `kubectl get pods` et `kubectl describe deployment <nom>` pour observer l’état du Deployment et des Pods.

**3. Mettre à l’échelle un Deployment**\
Modifie le nombre de réplicas à 5 dans le manifeste, puis réapplique. Vérifie que 5 Pods sont en Running.

**4. Mettre à jour l’image d’un Deployment**\
Change la version de l’image (par exemple `nginx:1.23` à `nginx:1.24`) dans le manifeste, applique la modification et observe le déroulement d’un Rolling Update.

**5. Revenir à la version précédente (Rollback)**\
Utilise `kubectl rollout undo deployment <nom>` pour revenir à la version précédente de l’application.

**6. Observer l’historique des rollouts**\
Exécute `kubectl rollout history deployment <nom>` pour afficher la liste des révisions du Deployment.

**7. Pauser et reprendre un Deployment**\
Avec `kubectl rollout pause deployment <nom>`, suspends le rollout, puis modifie le manifeste (ex : ressources), observe qu’aucune modification n’a lieu, puis reprends avec `kubectl rollout resume deployment <nom>`.

**8. Ajouter des labels et sélectionner les Pods**\
Ajoute des labels à ton manifeste de Deployment. Utilise ensuite `kubectl get pods --selector=<label>=<valeur>` pour filtrer les Pods.

# Solution :

**1. Créer un Deployment basique**\
[nginx-deployment.yaml](nginx-deployment.yaml)

**2. Vérifier l’état du Deployment**
```
$ kubectl apply -f k8s.yaml
$ kubectl get deployments
$ kubectl get pods
$ kubectl describe deployment nginx-deployment
```

**3. Mettre à l’échelle un Deployment**
```
$ kubectl scale deployment nginx-deployment --replicas=5
$ kubectl get pods
```

**4. Mettre à jour l’image d’un Deployment**
```
$ kubectl set image deployment/nginx-deployment nginx-container=nginx:1.25-alpine

$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-6cf9ff54d-76kkp   1/1     Running   0          4m16s
...

$ kubectl describe nginx-deployment-6cf9ff54d-76kkp
...
Containers:
  nginx-container:
    Container ID:   docker://2b246fc28ab5cfa18bd8a82092a56136c148ba5abde307d861bdffa3b1432c84
    Image:          nginx:1.25-alpine
...
```

**5. Revenir à la version précédente (Rollback)**
``` 
$ kubectl rollout undo deployment/nginx-deployment 
deployment.apps/nginx-deployment rolled back

$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6d6d898675-9kvkj   1/1     Running   0          3m35s
...

$ kubectl describe pod nginx-deployment-6d6d898675-9kvkj
...
Containers:
  nginx-container:
    Container ID:   docker://2061dc81fcfc08944f819677ffc5e705242c629c1f2d1abf27f093541169cac3
    Image:          nginx:latest
...
```

**6. Observer l’historique des rollouts**
```
$ kubectl rollout history deployment nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

**7. Pauser et reprendre un Deployment**
```
# Mettre en pause le Deployment : les modifications apportées ne seront pas appliquées aux Pods tant que le Deployment reste en pause.
kubectl rollout pause deployment nginx-deployment
# Résultat attendu :
deployment.apps/nginx-deployment paused

# Vérifier que le Deployment est bien en pause :
kubectl describe deployment nginx-deployment
...
  Progressing    Unknown  DeploymentPaused
...

# Modifier les ressources (exemple : version de l'image nginx) dans le fichier nginx-deployment.yaml :
...
      containers:
        - name: nginx-container
          image: nginx:1.25-alpine
...

# Appliquer les changements apportés au fichier YAML
kubectl apply -f k8s.yaml
# Résultat attendu :
deployment.apps/nginx-deployment configured

# Vérifier la version de l'image utilisée dans les Pods, les changements ne sont pas encore pris en compte car le Deployment est toujours en pause :
kubectl describe pod nginx-deployment-6d6d898675-9kvkj
...
Containers:
  nginx-container:
    Container ID:   docker://2061dc81fcfc08944f819677ffc5e705242c629c1f2d1abf27f093541169cac3
    Image:          nginx:latest
...

# Reprendre le Deployment pour appliquer tous les changements en attente, les Pods sont alors mis à jour selon la nouvelle configuration :
kubectl rollout resume deployment nginx-deployment
# Résultat attendu :
deployment.apps/nginx-deployment resumed

# Vérifier la version de l'image dans les nouveaux Pods, les changements ont bien été appliqués :
kubectl describe pod nginx-deployment-6cf9ff54d-58zcd
...
Containers:
  nginx-container:
    Container ID:   docker://32882795d04f10e0dd9f2ac56b76d4a088bcb9a8aaff45710ac21ac77b732359
    Image:          nginx:1.25-alpine
...
```

**8. Ajouter des labels et sélectionner les Pods**\
Exemple d’ajout de label dans le YAML :
```yaml
labels:
  app: nginx-pod
  environment: dev
```
```
# Commandes pour filtrer :
$ kubectl get pods -l environment=dev
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-f866cfb56-hxw62   1/1     Running   0          26s
nginx-deployment-f866cfb56-q4hns   1/1     Running   0          28s
nginx-deployment-f866cfb56-v8wpt   1/1     Running   0          29s
```



