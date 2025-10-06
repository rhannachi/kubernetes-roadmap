# Introduction Network Policy Kubernetes

### Introduction
Avant d’aborder les **Network Policies**, il est essentiel de comprendre les notions de base concernant le trafic réseau et la sécurité.\
Nous partirons d’un exemple simple de communication entre une application web, une API backend et une base de données, puis nous verrons comment ces concepts s’appliquent dans un **Cluster Kubernetes**.

***

### Exemple classique de flux réseau
Imaginons une architecture classique composée de trois composants :
- Un **web server** servant le front-end (port 80).
- Un **API server** servant les APIs backend (port 5000).
- Un **database server** recevant les requêtes SQL (port 3306).

Le flux est le suivant :
1. Un utilisateur envoie une requête HTTP au web server (port 80).
2. Le web server appelle l’API server (port 5000).
3. L’API server interroge la base de données (port 3306).
4. La réponse remonte jusqu’à l’utilisateur.

On distingue alors deux types de trafic :
- **Ingress traffic** : trafic entrant (exemple : requête d’un utilisateur vers le web server).
- **Egress traffic** : trafic sortant (exemple : appel du web server vers l’API server).

Important : lors de la définition des règles, seul compte le *trafic initiateur*. 
Les réponses (------>) sont autorisées implicitement.

***

### Mise en place des règles réseau
Dans notre exemple, il faudrait les règles suivantes :
- Ingress : autoriser le **web server** à recevoir sur port 80.
- Egress : autoriser le **web server** à appeler l’**API server** sur port 5000.
- Ingress : autoriser l’**API server** à recevoir sur port 5000.
- Egress : autoriser l’**API server** à atteindre le **database server** sur port 3306.
- Ingress : autoriser le **database server** à recevoir sur port 3306.

Sans ces règles, les composants ne pourraient pas échanger correctement.

```  
Utilisateur
   |
   |  (Ingress: req. HTTP port 80)
   v
+------------+
| Web Server |  --(Egress: req. API port 5000)-->  +---------------+
|  port 80   |                                   | API Server       |
+------------+  <------------------------------  | port 5000       |
      |                                             +---------------+
      |                                              |
      |                                              | (Egress: req. DB port 3306)
      |                                              v
      |                                       +----------------+
      |                                       | Database Server|
      |                                       | port 3306      |
      |                                       +----------------+
      |
```

***

### Réseau dans Kubernetes
Dans un **Cluster Kubernetes** :
- Chaque **Node**, **Pod** et **Service** possède une adresse IP.
- Les **Pods** communiquent librement entre eux par défaut (*All Allow*).
- L’un des prérequis réseaux est que tous les Pods puissent s’atteindre sans configuration spécifique telle que des routes manuelles.

Exemple : dans une installation typique, tous les Pods font partie d’un réseau virtuel couvrant l’ensemble des Nodes.

**👉 Ce qu’il faut retenir :** 
Dans Kubernetes, même si les Pods sont répartis sur plusieurs nodes avec des adresses IP différentes, ils peuvent communiquer entre eux de façon transparente.\
Kubernetes assure que tous les Pods peuvent communiquer entre eux, peu importe le node où ils résident, en utilisant un réseau virtuel.

**Question :**  
Pourquoi utiliser un Service **ClusterIP** dans Kubernetes alors que les pods peuvent communiquer directement entre eux via le réseau interne du cluster ?

**Réponse :**
Car l’adresse IP des pods est éphémère et peut changer à chaque redéploiement, rendant les connexions directes instables.\
Un Service **ClusterIP** fournit une adresse IP virtuelle et un nom DNS stables pour accéder à un groupe de pods, garantissant ainsi une communication fiable,\
un équilibrage de charge et une découverte constante au sein du cluster, même si les pods changent ou sont remplacés.

***

### Application à notre exemple
En déployant :
- Un **Pod web server**,
- Un **Pod API server**,
- Un **Pod database**,
- et leurs **Services** associés,

Par défaut, *tous peuvent communiquer entre eux*.  
Mais que se passe-t-il si l’on souhaite interdire au **web server** d’accéder directement à la **database**, pour obliger le passage par l’API ?

C’est exactement le rôle d’une **NetworkPolicy**.

### Concept de NetworkPolicy
Une **NetworkPolicy** est un objet Kubernetes appliqué à un ou plusieurs Pods via **labels** et **selectors**, et qui définit des règles **Ingress** et/ou **Egress**.

Exemple concret :
- Objectif : autoriser uniquement l’**API server** à accéder au **database server** sur port 3306.
- Étapes :
    1. Ajouter un label au Pod database (ex: `role=db`).
    2. Définir une **NetworkPolicy** qui sélectionne ce Pod via `podSelector`.
    3. Dans `policyTypes`, préciser `Ingress`.
    4. Ajouter une règle ingress autorisant les Pods avec label `role=api` sur port 3306.

Example avec NetworkPolicies

```yaml
apiVersion: networking.k8s.io/v1   # Version de l’API pour les NetworkPolicy
kind: NetworkPolicy                # Type d’objet : une NetworkPolicy
metadata:
  name: allow-api-to-db             # Nom de la policy, ici pour autoriser l’API à accéder à la DB
spec:
  podSelector:                      # Sélectionne les Pods ciblés par cette NetworkPolicy
    matchLabels:
      role: db                      # Ici, la policy s’applique uniquement aux Pods ayant le label "role=db"
  policyTypes:
    - Ingress                       # On spécifie que la policy gère uniquement le trafic entrant (Ingress)
  ingress:                          # Définition des règles d’autorisation en Ingress
    - from:                         # Spécifie les Pods/source autorisés à parler aux Pods "role=db"
        - podSelector:
            matchLabels:
              role: api             # Seuls les Pods étiquetés "role=api" (API server) sont autorisés
      ports:                        # Liste des ports accessibles
        - protocol: TCP             # Protocole autorisé
          port: 3306                # Port MySQL (3306) autorisé, limitant l’accès aux bases de données
```

- Cette configuration applique la **NetworkPolicy** uniquement aux Pods de rôle `db`.
- Elle bloque tout trafic entrant, sauf celui venant des Pods étiquetés `role=api` vers le port 3306.
- Les egress du Pod database restent libres, puisque nous n’avons pas restreint `Egress`.

***

### Points importants
- Une **NetworkPolicy** n’a d’effet que si le CNI (Container Network Interface) supporte cette fonctionnalité.
- Solutions compatibles : **Calico**, **Cilium**, **Kube-Router**, **Romana**, **Weave Net**.
- **Flannel** ne supporte *pas* les Network Policies. Si vous déployez une policy dans ce cas, aucun blocage n’aura lieu et aucun message d’erreur ne sera affiché.

***

## Résumé concis

- Par défaut, dans Kubernetes, tous les Pods communiquent librement.
- Le trafic se distingue en **ingress** (entrant) et **egress** (sortant).
- Une **NetworkPolicy** permet de restreindre ce trafic entre Pods via **labels** et **selectors**.
