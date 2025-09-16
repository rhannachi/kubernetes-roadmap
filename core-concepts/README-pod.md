# Pod

***

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

