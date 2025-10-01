# Les Services Kubernetes

Un **Service** dans Kubernetes est un objet qui permet la communication entre les différents composants d’une application, que ce soit **à l’intérieur** du Cluster (entre Pods et autres Services) ou **depuis l’extérieur** (vers les utilisateurs).  
Il permet de lier des Pods entre eux ou de les exposer aux clients de manière stable, même si les Pods eux-mêmes sont éphémères.

## Pourquoi utiliser un Service ?
Prenons un exemple concret :
- Un groupe de Pods sert le **front-end web** aux utilisateurs.
- Un autre groupe exécute les **processus back-end**.
- Un troisième groupe se connecte à une **base de données externe**.

Sans Service, ces Pods ne pourraient pas communiquer de manière fiable, car leurs adresses IP changent lorsqu’ils sont redéployés.\
Le Service introduit une **couche d’abstraction stable** qui permet :
- aux utilisateurs d’accéder au front-end,
- au front-end de communiquer avec le back-end,
- au back-end de se connecter à une base de données externe,
- et à l’ensemble des Pods de rester faiblement couplés (indépendance entre microservices).

***

## Types de Services Kubernetes

1. **ClusterIP** (par défaut)
    - Crée une IP virtuelle accessible uniquement **à l’intérieur du Cluster**.
    - Exemple d’utilisation : relier un service back-end à un front-end.

2. **NodePort**
    - Ouvre un port sur chaque Node du Cluster.
    - Les utilisateurs accèdent à l’application via l’IP du Node + le port exposé.
    - Exemple : accéder à une application web depuis l’extérieur.

3. **LoadBalancer**
    - Crée un Load Balancer externe (supporté par un Cloud provider).
    - Redirige automatiquement vers les différents Pods du Service.
    - Exemple : répartir les requêtes utilisateurs entre plusieurs serveurs web.

***

## Exemple pratique : exposer une application via un "NodePort"

Imaginons une application web tournant dans un **Pod** sur port `80`. Nous voulons y accéder depuis un ordinateur externe sans SSH dans le Node.

#### Étape 1 : Création du fichier de définition du Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-web-service
spec:
  type: NodePort
  selector:
    app: my-web-app
  ports:
    - port: 80         # Port du Service (interne au Cluster)
      targetPort: 80   # Port des Pods (où l’app écoute)
      nodePort: 30008  # Port exposé sur chaque Node
```

#### Étape 2 : Application de la configuration

```
$ kubectl apply -f my-web-service.yaml
```

#### Étape 3 : Vérification du Service

```
$ kubectl get services
NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
my-web-service   NodePort   10.96.220.130  <none>        80:30008/TCP   1m
```

#### Étape 4 : Accès depuis l’extérieur

Si le Node a pour IP `192.168.1.2`, l’application est accessible à l’adresse :

```
http://192.168.1.2:30008
```

***

### Exemple avec plusieurs Pods

En production, il est courant d’avoir plusieurs Pods identiques pour assurer **haute disponibilité** et **scalabilité**.

Si trois Pods ont le label `app: my-web-app`, le Service sélectionne automatiquement les trois grâce au `selector`.\
le "NodePort" de Kubernetes agit alors comme un **load balancer interne** et distribue les requêtes entre eux de façon aléatoire.

***

### Services multi-Nodes

Lorsque les Pods sont répartis sur plusieurs Nodes dans le Cluster :
- Le **NodePort** est ouvert sur tous les Nodes.
- On peut accéder à l’application via n’importe quelle IP de Node + port `30008`.

Exemple :

```
curl http://192.168.1.2:30008
curl http://192.168.1.3:30008
```

Dans les deux cas, l’accès fonctionne car Kubernetes gère la redirection.

***

## Résumé concis

- Un **Service** est un objet Kubernetes qui permet de stabiliser l’accès aux Pods dont les IP changent.
- Il existe **3 types principaux** de Service :
    - **ClusterIP** : communication interne dans le Cluster.
    - **NodePort** : ouvre un port accessible depuis l’extérieur (via IP du Node).
    - **LoadBalancer** : crée un Load Balancer externe (cloud).
- Un Service utilise un **selector** basé sur les labels pour cibler les Pods.
- Il agit aussi comme un **load balancer** interne, répartissant automatiquement le trafic entre plusieurs Pods.
- Exemple : un Service de type NodePort permet d’accéder à une app web via `http://<NodeIP>:<NodePort>`.

***

## NodePort joue-t-il le rôle de Load Balancer interne ?

Oui, quand un Service de type **NodePort** expose plusieurs Pods (ayant le même label dans son selector), il agit effectivement comme un **load balancer interne simplifié**.\
Concrètement, le Service :
- écoute un port fixe (NodePort) sur chaque Node du Cluster,
- reçoit les requêtes entrantes sur ce port,
- répartit ces requêtes **aléatoirement** entre les différents Pods qui correspondent au selector.

Ainsi, le Service NodePort assure la **distribution du trafic** vers plusieurs Pods de manière transparente, sans configuration supplémentaire.

***

### Pourquoi utiliser un Service de type LoadBalancer alors ?

Un Service **LoadBalancer** fait tout ce que fait NodePort **et en plus** :

- Il **provisionne automatiquement un Load Balancer externe** (fourni par le Cloud provider, par exemple un AWS Elastic Load Balancer, Azure Load Balancer, etc.).
- Ce Load Balancer externe distribue le trafic **avant même qu’il n’atteigne les Nodes**, ce qui permet :
    - une meilleure résilience : si un Node est indisponible, il est exclu automatiquement, ce qui évite que le trafic soit perdu.
    - une vraie gestion de la montée en charge (scalabilité), avec répartition plus sophistiquée (round-robin, moins de connexions, etc.) parfois basée sur couche 4 ou couche 7 selon le type de Load Balancer.
    - une IP publique dédiée, simplifiant la gestion des accès externes.


### Illustration simple en comparaison

| Aspect                    | NodePort                              | LoadBalancer                             |
|--------------------------|-------------------------------------|----------------------------------------|
| Exposition externe       | Oui, sur un port fixe de chaque Node | Oui, via un Load Balancer Cloud externe |
| Charge répartie sur Pods | Oui, répartie aléatoirement par kube-proxy | Oui, répartie par le Load Balancer externe |
| Haute disponibilité      | Non automatique (le client doit gérer le Node IP) | Oui, Load Balancer gère la santé des Nodes |
| IP publique dédiée       | Non                                | Oui                                    |
| Complexité               | Simple, sans dépendance cloud       | Plus complexe, dépend du cloud provider |
| Cas d'usage typique      | Développement, test, petits clusters | Production, haute disponibilité, grande échelle |

***

### Exemple

Imaginons une application web avec 3 Pods derrière un Service NodePort exposé sur le port 30008.

1. Le client envoie une requête vers `192.168.1.2:30008` (Node A).
2. kube-proxy sur Node A reçoit la requête, choisit aléatoirement un des 3 Pods (sur Node A, B ou C) et y redirige la requête.
3. Si Node A tombe en panne, le client doit manuellement changer d’IP pour un autre Node (pas automatique).

Avec un Service LoadBalancer :

1. Le client envoie une requête vers l’IP publique du Load Balancer.
2. Le Load Balancer évalue la santé des Nodes et répartit la requête vers un Node sain.
3. Le Load Balancer masque la complexité du Cluster et garantit la haute disponibilité sans intervention client.

***

## Conclusion synthétique

- Le Service NodePort fournit un load balancing **basique interne** entre les Pods, suffisant pour de petites configurations sans cloud provider.
- Le Service LoadBalancer ajoute un **load balancing externe et avancé**, automatique, avec IP publique, idéal en production sur cloud.
- Le choix dépend surtout du contexte et de la complexité attendue : simplicité vs robustesse et scalabilité.


