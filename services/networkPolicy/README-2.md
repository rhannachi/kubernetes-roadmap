## NetworkPolicy Ingress/Egress Kubernetes

Dans cet exemple, nous disposons de trois Pods :
- un **web Pod**
- un **API Pod**
- un **database Pod**

Notre objectif est de **protéger le database Pod** pour qu’il n’accepte les connexions **que du API Pod**, et **uniquement sur le port 3306** (port MySQL).  
Par défaut, Kubernetes autorise **tout le trafic** (Ingress et Egress) entre tous les Pods. Nous allons donc créer une **NetworkPolicy** pour restreindre cet accès.

***

### Étape 1 – Création d’une NetworkPolicy

Nous créons une **NetworkPolicy** nommée `db-policy` afin d’appliquer des règles au database Pod.  
L’association entre la policy et le Pod se fait grâce à un **podSelector** utilisant les labels.  
Exemple :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
```

Cette première étape **bloque par défaut tout le trafic** entrant/sortant du Pod. Il faut maintenant créer une règle Ingress autorisant uniquement le trafic souhaité.

***

### Étape 2 – Autoriser le trafic depuis l’API Pod

Nous voulons permettre au **API Pod** d’envoyer des requêtes MySQL au **database Pod** via le port 3306.  
Cela nécessite une **règle d’Ingress** (trafic entrant vers le Pod). Les réponses au trafic autorisé sont automatiquement permises ; il n’est donc pas nécessaire d’ajouter une règle de sortie.

Exemple complet :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 3306
```

Cette policy autorise uniquement les Pods ayant le label `role: api` à se connecter au `role: db` via le port 3306.

***

### Étape 3 – Limiter l’accès à un Namespace spécifique

Si plusieurs API Pods avec le même label existent dans d’autres Namespaces (ex. `dev`, `test`, `prod`), tous pourront accéder au database Pod. Pour restreindre cet accès uniquement aux Pods du Namespace `prod`, on utilise **namespaceSelector** conjointement à **podSelector** :

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        env: prod
    podSelector:
      matchLabels:
        role: api
  ports:
  - protocol: TCP
    port: 3306
```

Ici, les labels du Namespace doivent être définis lors de sa création :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    env: prod
```

Cette configuration exige que la source du trafic réponde aux deux critères :
- être un Pod avec le label `role: api`
- situé dans un Namespace labellisé `env: prod`  
  → Cela fonctionne comme un **ET logique** (AND).

Si les sélecteurs sont séparés en deux règles différentes, cela devient un **OU logique** (OR), et plus de trafic sera autorisé.

***

### Étape 4 – Autoriser un serveur externe (IPBlock)

Si un serveur de sauvegarde externe (hors cluster) doit accéder au database Pod (adresse IP `192.168.5.1`), ni **podSelector** ni **namespaceSelector** ne conviennent.  
On utilise alors un **ipBlock** :

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 192.168.5.1/32
  ports:
  - protocol: TCP
    port: 3306
```

Cela permet aux adresses IP spécifiées d’accéder au Pod, même si elles ne sont pas dans le cluster.

***

### Étape 5 – Exemple d’Egress (sortie du trafic)

Supposons maintenant que le **database Pod** doive se connecter à un serveur distant pour y envoyer ses sauvegardes.  
Cette fois, le trafic **part du database Pod** : il faut donc une **règle Egress**.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-egress
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.1/32
    ports:
    - protocol: TCP
      port: 80
```

Ainsi, le database Pod peut envoyer du trafic sortant vers `192.168.5.1` sur le port 80 (HTTP).

***

## Résumé essentiel

- **NetworkPolicy** contrôle le trafic réseau entre Pods et vers l’extérieur.
- Par défaut, tout trafic est autorisé si aucune policy n’est définie.
- Types de règles :
  - **Ingress** : entrées vers le Pod
  - **Egress** : sorties depuis le Pod
- **Sélecteurs possibles** :
  - **podSelector** : sélection par labels de Pods
  - **namespaceSelector** : sélection par labels de Namespaces
  - **ipBlock** : sélection par adresse IP ou plage d’adresses
- Les sélecteurs combinés dans une même règle appliquent un **ET logique**, alors que des règles séparées équivalent à un **OU logique**.
- Exemple typique :
  - Autoriser le trafic **Ingress** depuis le **API Pod** (port 3306)
  - Interdire tout autre trafic
  - Optionnellement, autoriser un serveur externe via **ipBlock**
  - Pour une communication sortante (backup, API externe), utiliser **Egress**
