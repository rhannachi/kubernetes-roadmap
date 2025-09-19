## Exercice

### Partie 1

Crée 2 Pods :
- Le premier pour le front-end avec une image `nginx`
- Le deuxième pour le back-end avec une image `node`

Le Pod front est géré par un ReplicaSet dans le fichier `rs-front.yaml`,  
et le Pod back par un ReplicationController dans le fichier `rc-back.yaml`.

Teste la suppression des pods pour voir s’ils sont recréés automatiquement ou pas.

**Solution :**  
[rc-back.yaml](rc-back.yaml)  
[rs-front.yaml](rs-front.yaml)

```
$ kubectl apply -f rc-back.yaml -f rs-front.yaml 
replicationcontroller/back-replicationcontroller created
replicaset.apps/front-replicaset created
```

```
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
back-replicationcontroller-p7lbr   1/1     Running   0          18s
back-replicationcontroller-sv9vb   1/1     Running   0          18s
front-replicaset-9m6n9             1/1     Running   0          18s
front-replicaset-h724c             1/1     Running   0          18s
```

```
$ kubectl get rs
NAME               DESIRED   CURRENT   READY   AGE
front-replicaset   2         2         2       81s
```

```
$ kubectl get rc
NAME                         DESIRED   CURRENT   READY   AGE
back-replicationcontroller   2         2         2       85s
```

On teste la suppression d’un pod, par exemple `back-replicationcontroller-6k8dr` :
```
$ kubectl delete pod back-replicationcontroller-6k8dr
pod "back-replicationcontroller-6k8dr" deleted
```

On remarque qu’un nouveau pod `back-replicationcontroller-qvk4l` a été recréé automatiquement pour remplacer l’ancien :
```
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
back-replicationcontroller-qvk4l   1/1     Running   0          43s
back-replicationcontroller-sv9vb   1/1     Running   0          29m
front-replicaset-9m6n9             1/1     Running   0          29m
front-replicaset-h724c             1/1     Running   0          29m
```

***

### Partie 2

On veut créer 2 Pods helpers basés sur l’image `busybox` dans des fichiers YAML séparés :
- `busybox-front.yaml` pour le front
- `busybox-back.yaml` pour le back

Chacun de ces Pods helpers est censé être géré par le ReplicaSet ou ReplicationController correspondant.

**Solution :**  
[busybox-back.yaml](busybox-back.yaml)  
[busybox-front.yaml](busybox-front.yaml)

Avant de relancer toute la stack Kubernetes, on modifie le nombre de replicas dans les controllers à 3 :
```yaml
replicas: 3
```

Puis on applique tous les fichiers :
```
$ kubectl apply -f busybox-front.yaml -f busybox-back.yaml -f rc-back.yaml -f rs-front.yaml
pod/busybox-front-helper created
pod/busybox-back-helper created
replicationcontroller/back-replicationcontroller created
replicaset.apps/front-replicaset created
```

On liste les pods :
```
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
back-replicationcontroller-68qtf   1/1     Running   0          21s
back-replicationcontroller-xqrmb   1/1     Running   0          21s
busybox-back-helper                1/1     Running   0          21s
busybox-front-helper               1/1     Running   0          21s
front-replicaset-gjn7l             1/1     Running   0          21s
front-replicaset-ztrfx             1/1     Running   0          21s
```

On supprime un pod backend et on observe :
```
$ kubectl delete pod back-replicationcontroller-68qtf 
pod "back-replicationcontroller-68qtf" deleted

$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
back-replicationcontroller-tshpp   1/1     Running   0          42s
back-replicationcontroller-xqrmb   1/1     Running   0          2m30s
busybox-back-helper               1/1     Running   0          2m30s
busybox-front-helper              1/1     Running   0          2m30s
front-replicaset-gjn7l            1/1     Running   0          2m30s
front-replicaset-ztrfx            1/1     Running   0          2m30s
```

Le pod backend est recréé automatiquement.

***

On supprime ensuite le pod helper backend :
```
$ kubectl delete pod busybox-back-helper  
pod "busybox-back-helper" deleted

$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
back-replicationcontroller-tshpp   1/1     Running   0          3m12s
back-replicationcontroller-w4zp7   1/1     Running   0          32s
back-replicationcontroller-xqrmb   1/1     Running   0          5m
busybox-front-helper              1/1     Running   0          5m
front-replicaset-gjn7l            1/1     Running   0          5m
front-replicaset-ztrfx            1/1     Running   0          5m
```

Le pod helper backend **n’a pas été recréé automatiquement**.

***

### Explication

Dans la partie 2, tu as créé 2 Pods helpers dans des fichiers YAML séparés, sans les intégrer dans le `template` du ReplicaSet (front) ni du ReplicationController (back).

Or, en Kubernetes, **un ReplicaSet ou un ReplicationController ne gère automatiquement que les Pods qu’il crée à partir de son propre `template`** :

- Pour qu’un Pod soit géré automatiquement (créé, recréé, maintenu au nombre voulu), il doit impérativement être défini dans la section `template` du ReplicaSet ou ReplicationController.
- La simple création manuelle d’un Pod avec des labels correspondant **ne suffit pas** à faire en sorte que le ReplicaSet ou ReplicationController le gère automatiquement.
- Ces Pods créés séparément existeront, mais ne seront **pas remplacés ni maintenus automatiquement** par le controller.
- Ce comportement est voulu, il permet à Kubernetes de toujours maintenir la cohérence avec la définition du `template`.

***

### Conclusion

- Les Pods helpers créés séparément ne sont pas gérés automatiquement par les controllers existants.
- Pour avoir une gestion automatique complète, il faudrait soit les intégrer dans le `template` des controllers correspondants,
- soit créer des ReplicaSets ou ReplicationControllers dédiés aux helpers.

Cette distinction est importante pour une bonne compréhension et administration de Kubernetes.
