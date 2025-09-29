# Monitoring Kubernetes

Dans cette section, nous abordons la surveillance (monitoring) d’un **Kubernetes cluster**.

### Quoi surveiller dans un cluster Kubernetes ?

Pour bien surveiller Kubernetes, il faut collecter des métriques à plusieurs niveaux :
- Au niveau des **nodes** : nombre total de nodes, leur état de santé, et leurs performances en CPU, mémoire, réseau et disque.
- Au niveau des **pods** : nombre de pods actifs, ainsi que leurs consommations de CPU et mémoire.

### Besoin d’une solution complète de monitoring

Il nous faut une solution capable de collecter ces métriques, de les stocker et de fournir des capacités d’analyse.

À ce jour, Kubernetes ne propose pas de monitoring intégré complet.  
Néanmoins, plusieurs solutions open source existent :
- Metrics Server
- Prometheus
- Elastic Stack

Et des solutions propriétaires telles que Datadog ou Dynatrace.

Dans le cadre de ce cours, qui se concentre sur le rôle de développeur Kubernetes, nous allons nous limiter à présenter le **Metrics Server**.  

On peut déployer un seul Metrics Server par cluster Kubernetes. Il collecte les métriques sur chaque node et pod, les agrège et les stocke en mémoire.

Note importante :  
Le Metrics Server ne stocke pas ces données sur disque, donc il ne fournit pas d’historique des performances. Pour cela, il faut utiliser des solutions plus avancées comme Prometheus.

### Comment les métriques sont-elles collectées ?

Sur chaque node, le **kubelet** exécute un composant nommé **cAdvisor** (Container Advisor) qui récupère les métriques de performance des pods.\
Ces métriques sont ensuite exposées via l’API du kubelet pour être consommées par le Metrics Server.

### Comment déployer Metrics Server ?

- Sur Minikube (cluster local), la commande suivante active le Metrics Server :
  ```
  $ minikube addons enable metrics-server
  ```

- Sur d’autres environnements, il faut :
    - Cloner les fichiers de déploiement du Metrics Server depuis leur repository GitHub officiel.
    - Les appliquer avec la commande :
  ```
  $ kubectl create -f <deployment-files>
  ```

Ces fichiers déploient les ressources nécessaires (pods, services, rôles) pour que Metrics Server collecte les métriques.

### Accès aux métriques

Une fois déployé et démarré, le Metrics Server demande un certain temps pour collecter les données.

Pour consulter les performances des nodes, on utilise :
```
$ kubectl top nodes
```

Pour consulter les métriques des pods, on utilise :
```
$ kubectl top pods
```

***

## Résumé concis

- Kubernetes nécessite la surveillance des ressources au niveau des **nodes** et des **pods** (CPU, mémoire, réseau, disque).
- Kubernetes ne fournit pas de monitoring complet intégré ; il existe des solutions open source (Metrics Server, Prometheus, Elastic Stack) et propriétaires (Datadog).
- Le **Metrics Server** est un composant léger de collecte de métriques récents (CPU/mémoire) stockées en mémoire, sans historique.
- Il récupère les métriques via le **kubelet** et son composant **cAdvisor** sur chaque node.
- Sur Minikube, Metrics Server peut être activé via un addon. Sinon, on déploie ses manifests YAML officiels.
- Les commandes `kubectl top nodes` et `kubectl top pods` permettent d’afficher les métriques collectées par Metrics Server.


