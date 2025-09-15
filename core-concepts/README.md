```
   Master vs Worker Nodes

   +----------------------------------+         +----------------------------------+
   |             Master               |         |           Worker Node            |
   |----------------------------------|         |----------------------------------|
   |  kube-apiserver                  | <-----> |  kubelet                         |
   |  etcd                            |         |                                  |
   |  controller                      |         |  Container Runtime               |
   |  scheduler                       |         |                                  |
   +----------------------------------+         +----------------------------------+
```

### Nodes et Cluster
- Un **Node** est une machine (physique ou VM) qui exécute Kubernetes et héberge les conteneurs.
- On les appelait autrefois *minions*.
- Un **Cluster** est un ensemble de Nodes. Il permet la haute disponibilité (si un Node tombe, les autres maintiennent l’application) et le partage de charge.

### Rôle du Control Plane (anciennement Master)
- Le **Control Plane** est responsable de la gestion et de l’orchestration des Worker Nodes.
- Il surveille l’état du Cluster, assigne les conteneurs aux Nodes et déplace les charges de travail si un Node échoue.

### Composants principaux de Kubernetes
- **API Server** : point d’entrée principal, expose l’API pour l’administration du Cluster.
- **etcd** : base de données distribuée clé/valeur pour stocker la configuration et l’état du Cluster.
- **Scheduler** : assigne les Pods aux Nodes disponibles.
- **Controller Manager** : supervise l’état du Cluster et déclenche des actions (par ex. redémarrer des Pods).
- **Container Runtime** : moteur qui exécute les conteneurs (ex. Docker, containerd, CRI-O).
- **kubelet** : agent installé sur chaque Worker Node, qui fait tourner les Pods et communique avec l’API Server.

### Différence Control Plane / Worker Nodes
- **Control Plane** : exécute `kube-apiserver`, `etcd`, `controller-manager`, `scheduler`.
- **Worker Nodes** : exécutent `kubelet` et le **Container Runtime** pour héberger les Pods.

### Outil kubectl
- **kubectl** est l’outil CLI officiel pour interagir avec Kubernetes.
- Commandes clés :
  - `kubectl run` : déployer une application.
  - `kubectl cluster-info` : obtenir des informations sur le Cluster.
  - `kubectl get nodes` : lister les Nodes.
