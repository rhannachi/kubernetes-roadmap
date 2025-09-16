# Kubernetes Architecture

***

## Architecture

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

```
$ kubectl config current-context
$ minikube start --driver=docker
```
***

## Pod vs Node

Un **Pod** dans Kubernetes n’est pas une machine virtuelle à l’intérieur d’un Node (comme une VM à l’intérieur d’une VM), mais une unité logique qui groupe un ou plusieurs conteneurs applicatifs qui partagent le même environnement réseau, stockage et ressources du Node.

### Définition du Pod
- Un **Pod** est la plus petite unité déployable dans Kubernetes, servant principalement à héberger un ou plusieurs conteneurs (généralement un seul dans la majorité des cas).
- Les conteneurs d’un même Pod partagent la même adresse IP, le stockage (volumes) et d’autres ressources, ce qui facilite leur communication directe et leur coordination.
- Un Pod fonctionne comme un “hôte logique” : ce n’est pas un système d’exploitation complet comme une VM, mais un regroupement isolé de processus sur le Node, orchestré par Kubernetes.

### Différence avec les machines virtuelles
- Le **Node** correspond à une machine physique ou virtuelle (par exemple une instance EC2 sur AWS).
- Un **Pod** est une abstraction logicielle exécutée sur le Node, qui isole et organise les conteneurs, mais n’est pas une VM à l’intérieur du Node.
- Les Pods ne démarrent pas un OS complet, mais seulement les processus des conteneurs avec un environnement partagé.

***

## Pod

- Kubernetes ne déploie pas de conteneurs directement sur les Nodes: il encapsule chaque conteneur dans un objet appelé **Pod**.
- Un **Pod** est la plus petite unité de déploiement dans Kubernetes. Il représente une seule instance d’application.
- Le cas le plus courant: un pod contient un seul conteneur. Si l’on souhaite augmenter la capacité (scalabilité), il faut créer plusieurs pods, chacun avec une nouvelle instance de l’application.
- Si un Node est trop chargé, des Pods peuvent être répartis sur d’autres Nodes dans le cluster, ce qui permet d’augmenter la capacité globale.
- Pour faire évoluer une application, on ajoute ou supprime des Pods, pas des conteneurs à l’intérieur d’un même Pod.
- Les Pods peuvent parfois contenir plusieurs conteneurs: ces conteneurs doivent être très liés (exemple: une application principale et son conteneur “sidecar” de logging ou traitement de données).
- Dans un même Pod, tous les conteneurs partagent la même adresse IP, le même espace réseau et le même stockage: ils peuvent facilement communiquer et partager des fichiers.
- Cette cohabitation facilite la gestion du cycle de vie: si un Pod est supprimé, tous ses conteneurs le sont automatiquement.
- Dans Docker seul, tout doit être géré manuellement: le lien entre conteneurs, le stockage partagé, la surveillance… Avec Kubernetes, l’objet Pod simplifie et automatise tout cela.
- Même si souvent, on crée des Pods à un seul conteneur, Kubernetes oblige à encapsuler les applications dans des Pods, ce qui reste pérenne et flexible pour l’avenir.
- Pour créer un pod:
  - On utilise par exemple la commande:  
    `kubectl run nginx --image=nginx`
  - L’image du conteneur est téléchargée depuis le registre (Docker Hub par défaut).
- Pour voir les Pods du cluster:
  - Utiliser:  
    `kubectl get pods`
  - Le Pod passe successivement par les états `ContainerCreating` puis `Running` une fois démarré.

***
