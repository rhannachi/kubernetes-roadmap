# Les API Groups dans Kubernetes

Avant d’aborder les mécanismes d’autorisation, il est essentiel de comprendre la structure des **API Groups** dans Kubernetes.

## Qu’est-ce que l’API Kubernetes ?

L’**API Kubernetes** est le point d’entrée central pour toute interaction avec le **Cluster**.\
Toutes les opérations que vous effectuez — que ce soit via la commande `kubectl` ou directement avec une requête **REST** — passent par le **kube-apiserver**.

### Exemple : accéder à la version du Cluster

L’**API Server** écoute généralement sur le port `6443`.\
Vous pouvez vérifier la version de votre Cluster avec une requête HTTP :
```
$ curl -k https://<master-node-ip>:6443/version
```
ou via `kubectl` :
```
$ kubectl version --short
```

### Exemple : récupérer la liste des Pods

L’API suit une structure hiérarchique :
```
/api/v1/pods
```
Vous pouvez exécuter :
```
$ curl -k https://<master-node-ip>:6443/api/v1/pods
```
ou tout simplement :
```
$ kubectl get pods
```

---

## Structure des API Groups

L’API Kubernetes est organisée en plusieurs **API Groups**, chacun correspondant à une catégorie fonctionnelle.

### 1. **Core API Group**

Ce groupe contient les ressources de base essentielles au fonctionnement du Cluster. Son chemin est simplement `/api/v1`.

**Exemples de ressources du Core Group :**

* **Pod**
* **Service**
* **Node**
* **Namespace**
* **ConfigMap**
* **Secret**
* **PersistentVolume (PV)** et **PersistentVolumeClaim (PVC)**
* **ReplicationController**
* **Event**

➡️ Exemple :

```
$ kubectl get pods
$ kubectl get nodes
$ kubectl get configmaps
```

---

### 2. **Named API Groups**

Les **Named Groups** sont des regroupements logiques pour les fonctionnalités plus récentes ou spécialisées.
Leur chemin commence par `/apis/` suivi du nom du groupe (ex. `/apis/apps/v1`, `/apis/networking.k8s.io/v1`, etc.).

**Exemples de Named Groups et leurs ressources :**

| API Group                        | Exemples de Ressources                 |
| -------------------------------- | -------------------------------------- |
| **apps/v1**                      | Deployments, ReplicaSets, StatefulSets |
| **networking.k8s.io/v1**         | NetworkPolicies, Ingress               |
| **certificates.k8s.io/v1**       | CertificateSigningRequests             |
| **storage.k8s.io/v1**            | StorageClasses, VolumeAttachments      |
| **rbac.authorization.k8s.io/v1** | Roles, RoleBindings, ClusterRoles      |

Exemple :

``` 
$ kubectl get pods         → /api/v1/pods
$ kubectl get services     → /api/v1/services
$ kubectl get deployments  → /apis/apps/v1/deployments
$ kubectl get replicasets  → /apis/apps/v1/replicasets
```

---

## Les Verbs de l’API Kubernetes

Chaque **Resource** peut être manipulée par un ensemble d’actions appelées **verbs**.

**Principaux verbs :**

* `get` → Récupérer une ressource
* `list` → Lister les ressources
* `create` → Créer une ressource
* `delete` → Supprimer une ressource
* `update` → Mettre à jour une ressource
* `watch` → Surveiller les changements en temps réel

Exemple :

```
$ kubectl get deployment my-app
$ kubectl create -f deployment.yaml
$ kubectl delete pod my-pod
```

---

## Explorer les API disponibles

Vous pouvez lister toutes les API disponibles sur le Cluster en interrogeant directement le **kube-apiserver** :

```
$ curl -k https://<master-node-ip>:6443 -H "Authorization: Bearer <token>"
```

ou plus simplement via un **proxy local** :

```
$ kubectl proxy
```

Cela démarre un proxy sur le port `8001`, vous permettant d’explorer les API sans gérer manuellement les certificats.
Par exemple :

```
$ curl http://localhost:8001/
```

---

## kube-proxy vs kubectl proxy

Bien qu’ils portent des noms similaires, ces deux composants remplissent des rôles très différents :

| Composant         | Rôle principal                                                                                                         |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **kube-proxy**    | Gère la communication réseau entre les Pods et les Services dans le Cluster.                                           |
| **kubectl proxy** | Crée un proxy HTTP local pour accéder au kube-apiserver en utilisant les identifiants de votre fichier **kubeconfig**. |

---

## Points essentiels à retenir

* Toute interaction avec Kubernetes passe par le **kube-apiserver**.
* Les ressources sont organisées en **API Groups** :

    * Le **Core Group** (`v1`) contient les composants de base.
    * Les **Named Groups** contiennent les fonctionnalités avancées (apps, networking, storage…).
* Chaque ressource possède un ensemble d’actions appelées **verbs** (`get`, `list`, `create`, etc.).
* Vous pouvez interagir directement avec l’API via `curl` ou plus simplement via `kubectl proxy`.
* Ne pas confondre **kube-proxy** (réseau) et **kubectl proxy** (accès API).

---

### Résumé concis

* Le **kube-apiserver** est le point d’entrée pour toutes les opérations Kubernetes.
* Les **API Groups** structurent les ressources du Cluster :

    * **Core Group (`/api/v1`)** → Pods, Services, Nodes, etc.
    * **Named Groups (`/apis/...`)** → Apps, Networking, Storage, RBAC, etc.
* Chaque **Resource** possède des **verbs** (get, list, create, delete, update, watch).
* L’accès direct à l’API nécessite une **authentification**, sinon utilisez **kubectl proxy**.
* **kube-proxy ≠ kubectl proxy** : le premier gère le trafic réseau interne, le second facilite l’accès à l’API.

