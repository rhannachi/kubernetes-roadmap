# Exercice
### Objectif
Comprendre et expérimenter les mécanismes de taints et tolerations dans Kubernetes pour contrôler la planification des Pods sur différents Nodes dans un cluster Minikube à plusieurs nœuds.
### Contexte
Vous disposez d’un cluster Minikube configuré avec au moins 3 nœuds (1 control plane + 2 workers).
Vous devez gérer la répartition des Pods sur ces Nodes en utilisant des taints sur les Nodes et des tolerations dans les Deployments de Pods.

# Solution

```
$ minikube stop

$ minikube status
minikube
type: Control Plane
host: Stopped
kubelet: Stopped
apiserver: Stopped
kubeconfig: Stopped
```

***

## 1. Création d’un cluster Minikube multi-nodes (exemple 3 Nodes)

Minikube supporte la création multi-node avec l’option `--nodes`. Par exemple, pour démarrer un cluster avec 3 Nodes (1 control plane + 2 workers) :

```
$ minikube start --nodes=3 --memory=3g --cpus=3 -p multinode-cluster
```

- `--nodes=3` crée le control plane et 2 worker nodes (total 3 nodes).
- `-p multinode-cluster` crée un profil nommé `multinode-cluster` pour ce cluster.
- Ajustez `--memory` et `--cpus` selon la capacité de votre machine.

***

## 2. Vérification des nodes

Pour voir les nodes disponibles dans le cluster :

```
$ kubectl get nodes
NAME                    STATUS   ROLES           AGE   VERSION
multinode-cluster       Ready    control-plane   47s   v1.33.1
multinode-cluster-m02   Ready    <none>          28s   v1.33.1
multinode-cluster-m03   Ready    <none>          9s    v1.33.1
```

***

## 3. Exemple de Pods avec différentes *tolerations* pour tester *PreferNoSchedule* et *NoSchedule*

### Taint à appliquer sur un Node worker (exemple node `multinode-cluster-m02`)

```
$ kubectl taint nodes multinode-cluster-m02 app=blue:NoSchedule
```

### Deployment tolérant *NoSchedule*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-tolerate-noschedule
spec:
  replicas: 4
  selector:
    matchLabels:
      app: noschedule-app
  template:
    metadata:
      labels:
        app: noschedule-app
    spec:
      containers:
        - name: nginx
          image: nginx
      tolerations:
        - key: "app"
          operator: "Equal"
          value: "blue"
          effect: "NoSchedule"
```

***

### Taint avec effet *PreferNoSchedule* sur un autre Node (exemple node `multinode-cluster-m03`)

```
$ kubectl taint nodes multinode-cluster-m03 app=blue:PreferNoSchedule
```

### Deployment tolérant *PreferNoSchedule*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-tolerate-prefernoschedule
spec:
  replicas: 4
  selector:
    matchLabels:
      app: prefernoschedule-app
  template:
    metadata:
      labels:
        app: prefernoschedule-app
    spec:
      containers:
        - name: nginx
          image: nginx
      tolerations:
        - key: "app"
          operator: "Equal"
          value: "blue"
          effect: "PreferNoSchedule"
```

***

## Result

#### Vérifier précisément les taints appliqués sur chaque Node
```
$ kubectl describe node multinode-cluster | grep -i taint 
Taints:             <none>

$ kubectl describe node multinode-cluster-m02 | grep -i taint 
Taints:             app=blue:NoSchedule

$ kubectl describe node multinode-cluster-m03 | grep -i taint 
Taints:             app=blue:PreferNoSchedule
```

#### Afficher un résumé de plusieurs types d’objets kubernetes (Deployment, Pod, ReplicaSet, Service)
``` 
$ kubectl get all
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/deploy-tolerate-noschedule-67c546cdb7-7krqn         1/1     Running   0          2m9s
pod/deploy-tolerate-noschedule-67c546cdb7-9dgmn         1/1     Running   0          2m9s
pod/deploy-tolerate-noschedule-67c546cdb7-hp4fx         1/1     Running   0          2m9s
pod/deploy-tolerate-noschedule-67c546cdb7-knlfx         1/1     Running   0          2m9s
pod/deploy-tolerate-prefernoschedule-68dbccdff9-6pvxs   1/1     Running   0          2m9s
pod/deploy-tolerate-prefernoschedule-68dbccdff9-8ws29   1/1     Running   0          2m9s
pod/deploy-tolerate-prefernoschedule-68dbccdff9-v6xzt   1/1     Running   0          2m9s
pod/deploy-tolerate-prefernoschedule-68dbccdff9-w4qwz   1/1     Running   0          2m9s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   11m

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deploy-tolerate-noschedule         4/4     4            4           2m9s
deployment.apps/deploy-tolerate-prefernoschedule   4/4     4            4           2m9s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/deploy-tolerate-noschedule-67c546cdb7         4         4         4       2m9s
replicaset.apps/deploy-tolerate-prefernoschedule-68dbccdff9   4         4         4       2m9s

```

#### Afficher la liste des Pods avec le Node associé
``` 
$ kubectl get pods -o wide
NAME                                                READY   STATUS    RESTARTS   AGE    IP           NODE                    NOMINATED NODE   READINESS GATES
deploy-tolerate-noschedule-67c546cdb7-7krqn         1/1     Running   0          3m6s   10.244.0.3   multinode-cluster       <none>           <none>
deploy-tolerate-noschedule-67c546cdb7-9dgmn         1/1     Running   0          3m6s   10.244.0.5   multinode-cluster       <none>           <none>
deploy-tolerate-noschedule-67c546cdb7-hp4fx         1/1     Running   0          3m6s   10.244.1.2   multinode-cluster-m02   <none>           <none>
deploy-tolerate-noschedule-67c546cdb7-knlfx         1/1     Running   0          3m6s   10.244.1.3   multinode-cluster-m02   <none>           <none>
deploy-tolerate-prefernoschedule-68dbccdff9-6pvxs   1/1     Running   0          3m6s   10.244.2.2   multinode-cluster-m03   <none>           <none>
deploy-tolerate-prefernoschedule-68dbccdff9-8ws29   1/1     Running   0          3m6s   10.244.2.3   multinode-cluster-m03   <none>           <none>
deploy-tolerate-prefernoschedule-68dbccdff9-v6xzt   1/1     Running   0          3m6s   10.244.0.6   multinode-cluster       <none>           <none>
deploy-tolerate-prefernoschedule-68dbccdff9-w4qwz   1/1     Running   0          3m6s   10.244.0.4   multinode-cluster       <none>           <none>
```

***

## Analyse des Pods et Nodes

### Pods `deploy-tolerate-noschedule`

| Pod Name                                   | Node                   |
|--------------------------------------------|------------------------|
| deploy-tolerate-noschedule-67c546cdb7-7krqn | multinode-cluster       |
| deploy-tolerate-noschedule-67c546cdb7-9dgmn | multinode-cluster       |
| deploy-tolerate-noschedule-67c546cdb7-hp4fx | multinode-cluster-m02   |
| deploy-tolerate-noschedule-67c546cdb7-knlfx | multinode-cluster-m02   |

**Interprétation** :  
Les Pods tolérant le taint avec effet **NoSchedule** sont sur deux Nodes. Deux Pods sont sur le Node `multinode-cluster` (control plane), et deux autres sur `multinode-cluster-m02` (worker avec probablement le taint NoSchedule appliqué).\
Cela indique que les Pods avec tolérance au taint sont correctement planifiés sur les Nodes qui ont possiblement ce taint.\
Note : Par défaut, le control plane Node peut ne pas avoir de taint pour NoSchedule, donc certains Pods peuvent s’y placer.

***

### Pods `deploy-tolerate-prefernoschedule`

| Pod Name                                     | Node                   |
|----------------------------------------------|------------------------|
| deploy-tolerate-prefernoschedule-68dbccdff9-6pvxs | multinode-cluster-m03   |
| deploy-tolerate-prefernoschedule-68dbccdff9-8ws29 | multinode-cluster-m03   |
| deploy-tolerate-prefernoschedule-68dbccdff9-v6xzt | multinode-cluster       |
| deploy-tolerate-prefernoschedule-68dbccdff9-w4qwz | multinode-cluster       |

**Interprétation** :  
Les Pods tolérant le taint **PreferNoSchedule** sont également dispersés entre le Node control plane `multinode-cluster` et le worker Node `multinode-cluster-m03` qui a probablement le taint PreferNoSchedule.\
Cela correspond bien à l’effet de ce taint qui "préférerait" éviter le placement, mais ne l’interdit pas strictement.\
Le scheduler a réparti les Pods sur les deux nodes, comme attendu.

***

## Conclusions générales

- Les pods ayant la **toleration NoSchedule** sont bien planifiés sur le Node tainté `multinode-cluster-m02` et aussi sur le control plane. La présence sur le control plane peut indiquer l'absence d’un taint NoSchedule dessus ou que celui-ci tolère les pods actuellement déployés.
- Les pods avec la **toleration PreferNoSchedule** se répartissent entre les Nodes avec et sans taint PreferNoSchedule, ce qui est cohérent avec la nature "préférentielle" de ce taint.
- La répartition est cohérente avec les effets attendus des taints et tolerations appliqués dans votre cluster Minikube multi-node.
