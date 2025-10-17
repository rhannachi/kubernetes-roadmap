# Autorisation dans Kubernetes

## Introduction
L'**authentification** permet de vérifier l'identité des utilisateurs ou des machines cherchant à accéder au **Cluster** Kubernetes.\
Une fois l'accès obtenu, la question suivante est : *Qu'est-ce que ces utilisateurs peuvent faire sur le Cluster ?* C'est ici que l'**autorisation** intervient.

## Pourquoi l'autorisation est-elle nécessaire ?
En tant qu'administrateur, vous avez des droits complets sur le Cluster, comme consulter ou modifier des objets (**Pod**, **Node**, **Deployment**), en créer ou supprimer.\
Cependant, lorsque d'autres utilisateurs (administrateurs, développeurs, testeurs, applications comme Jenkins ou outils de monitoring) accèdent au Cluster, il est essentiel de limiter leurs permissions selon leurs besoins.\
Par exemple :
- Un développeur peut déployer des applications (**Pod**), mais ne doit pas modifier la configuration du Cluster.
- Une application tierce doit disposer uniquement des droits strictement nécessaires.
  De plus, lorsque le Cluster est partagé entre plusieurs équipes ou organisations via des **Namespaces**, il faut restreindre l'accès aux ressources pour chaque groupe.

## Mécanismes d'autorisation Kubernetes
Kubernetes propose plusieurs mécanismes pour gérer l'autorisation :
- **Node Authorizer**
- **Attribute-Based Access Control** (ABAC)
- **Role-Based Access Control** (RBAC)
- **Webhook Authorizer**
- **AlwaysAllow** et **AlwaysDeny**

***

### 1. Node Authorizer
> "Rappel: Rôle du kubelet et lien avec le Node".\
> Le kubelet est un agent (un processus) qui s'exécute sur chaque Node du cluster Kubernetes.\
> C'est lui qui est responsable de la gestion des Pods sur ce Node : il reçoit les manifestes (les instructions) depuis le kube-apiserver, lance et surveille les conteneurs, et remonte l'état du Node et des Pods.
> Ainsi, le kubelet est l'intermédiaire local qui fait fonctionner réellement les charges de travail sur un Node donné.

Le **Node Authorizer** gère l'accès des **Node** (via le processus **kubelet**) au **kube-apiserver**.\
3 étapes clés qu’effectue le Node Authorizer lorsqu’il traite une requête d’un kubelet

**1. Vérification de l’identité** :\
Le **Node Authorizer** commence par confirmer que la requête provient bien d’un **kubelet légitime** :
- Son nom d’utilisateur doit suivre la forme `system:node:<node-name>`.
- Il doit appartenir au groupe `system:nodes`.
- Le nom `<node-name>` doit correspondre à un **Node réel du cluster**.

**2. Vérification du type de ressource et de l’action** :\
Il autorise uniquement les opérations nécessaires au fonctionnement du kubelet:
- **Lecture** : Pods, Services, Endpoints, Secrets, ConfigMaps liés à ses Pods.
- **Écriture** : son propre objet Node, le statut de ses Pods, Event.
- **Authentification** : requêtes sur **CertificateSigningRequest**, **TokenReview**, **SubjectAccessReview**.

**3. Vérification du lien entre la ressource et le Node** :\
Le **Node Authorizer** s’assure que la ressource demandée concerne **le Node lui-même ou ses Pods**.  
Exemples :
- Un kubelet ne peut lire que les **Pods** dont le `spec.nodeName` correspond à son nom.
- Il ne peut accéder à un **Secret** ou **ConfigMap** que si ceux-ci sont **montés dans ses Pods**.

**Résumé :**  
«Le Node Authorizer vérifie **qui fait la requête** (identité du kubelet), **ce qu’il veut faire** (type d’opération), et **sur quelle ressource** (uniquement celles liées à son Node). »

**schéma** montrant **comment le Node Authorizer gère l’accès des Nodes (via le Kubelet)** au **kube-apiserver**, ainsi que la **circulation des requêtes et informations** dans le Cluster.

```
┌───────────────────────────┐
│         Kubelet           │
│ (Agent sur le Node)       │
└─────────────┬─────────────┘
              │
              │ ① Le Kubelet envoie une requête
              │    au kube-apiserver
              │    (ex: lecture / écriture de ressources)
              │
┌─────────────▼─────────────┐
│    Kubernetes API Server  │
│        (kube-apiserver)   │
└─────────────┬─────────────┘
              │
              │ ② Le kube-apiserver transmet la requête
              │    au Node Authorizer pour vérification
              │
      ┌───────▼────────────────────────────┐
      │         Node Authorizer            │
      │ (Module d’autorisation interne)    │
      └───────┬────────────────────────────┘
              │
   ┌──────────┼──────────────────────────────────────────┐
   │          │                                          │
③ Vérifie l’identité du Kubelet                         │
   - Nom commence par `system:node:`                     │
   - Appartient au groupe `system:nodes`                 │
              │                                          │
              │                                          │
     ┌────────▼──────────────┐
     │ Autorisation accordée ? │
     └────────┬──────────────┘
              │
     ┌────────┴──────────────┬────────────────────────────┐
     │                       │                            │
Oui (Authorized)       Non (Denied)                 │
     │                       │                            │
     │                       │                            │
┌────▼──────────────────┐     ┌────────────────────────────▼──────┐
│ kube-apiserver exécute│     │  Requête rejetée (403 Forbidden)  │
│ la requête du Node     │     │  par le kube-apiserver            │
│ selon ses permissions  │     └───────────────────────────────────┘
└────────┬───────────────┘
         │
         │
④ Accès aux objets internes du Cluster
─────────────────────────────────────────────────────────────────────
   │
   ▼
┌────────────────────────────────────────────────────────────────────┐
│   Ressources du Cluster :                                          │
│   - Services                                                       │
│   - Endpoints                                                      │
│   - Nodes                                                          │
│   - Pods                                                           │
│   - Configurations, Status, Metrics, etc.                          │
└────────────────────────────────────────────────────────────────────┘
```

#### Explication du flux
1. **Le Kubelet envoie une requête** au **kube-apiserver**
   → pour lire ou mettre à jour des ressources du cluster (Pods, Nodes, etc.).
2. **Le kube-apiserver reçoit la requête**
   → il identifie l’origine (le Node) et vérifie son identité.
3. **Le kube-apiserver transmet la requête** au **Node Authorizer**
   → pour déterminer si ce Node a le droit d’effectuer l’action demandée.
4. **Le Node Authorizer vérifie** :
    * Que la requête provient bien d’un Node du cluster.
    * Que ce Node n’accède qu’à ses propres ressources ou à celles liées à ses Pods.
5. **Décision d’autorisation :**
    * ✅ **Si autorisée :** le kube-apiserver exécute la requête.
    * ❌ **Si refusée :** la requête est rejetée (erreur 403).
6. **En cas d’autorisation**, le kube-apiserver effectue l’action demandée
   → accès ou mise à jour des objets internes du cluster (Pods, Services, etc.).

***

### 2. Attribute-Based Access Control (ABAC)

**ABAC (Attribute-Based Access Control)** signifie *Contrôle d’accès basé sur les attributs*.

➡️ Le principe est simple :
On définit des **règles basées sur des attributs** (informations) concernant :

* l’utilisateur (ex : son nom, son groupe),
* la ressource (ex : Pods, Services…),
* le namespace,
* l’action (lecture, écriture, suppression...).

Ces règles sont définies dans un **fichier JSON de politique** lu par le **kube-apiserver**.

#### Comment ça fonctionne
1. Tu écris un ou plusieurs fichiers de politique JSON.
   Chaque fichier contient une règle du type :
    * *tel utilisateur* peut *faire telle action* sur *telle ressource* dans *tel namespace*.
2. Le **kube-apiserver** lit ces fichiers au démarrage.
   Il vérifie les attributs de chaque requête API (ex. : qui fait quoi, sur quelle ressource ?)
   et décide si la requête est autorisée ou refusée.
3. Si tu veux modifier une règle, tu dois :
    * éditer le fichier JSON à la main,
    * puis redémarrer le **kube-apiserver**.

C’est pour cela qu’ABAC est rarement utilisé aujourd’hui : c’est rigide et peu pratique à grande échelle (contrairement à RBAC).

#### Exemple concret

Imaginons :
* un utilisateur `alice`,
* un namespace `dev`,
* elle doit **pouvoir lire** les pods mais **pas les modifier**.

Tu créerais une politique ABAC comme ceci :

```json
{
  "apiVersion": "abac.authorization.kubernetes.io/v1",
  "kind": "Policy",
  "spec": {
    "user": "alice",
    "namespace": "dev",
    "resource": "pods",
    "readonly": true
  }
}
```

Cela veut dire :

> "L’utilisateur `alice` a le droit de lire (`get`, `list`, `watch`) les Pods dans le namespace `dev`, mais pas de les créer, modifier ou supprimer."

Pour appliquer cette règle :

* tu sauvegardes le fichier dans `/etc/kubernetes/abac-policy.json`,
* tu démarres ton **kube-apiserver** avec :

  ```
  $ kube-apiserver --authorization-policy-file=/etc/kubernetes/abac-policy.json --authorization-mode=ABAC
  ```

#### Limites d’ABAC

* Tu dois modifier le fichier à la main à chaque changement.
* Il faut redémarrer le kube-apiserver pour que ce soit pris en compte.
* Pas pratique pour des clusters avec beaucoup d’utilisateurs ou de services.

C’est pourquoi aujourd’hui, **RBAC (Role-Based Access Control)** est presque toujours préféré : il est dynamique et gérable via des objets Kubernetes standard (`Role`, `RoleBinding`, etc.).

***

### 3. Role-Based Access Control (RBAC)
**RBAC** simplifie la gestion des accès en créant des **Role** (ou **ClusterRole**) qui définissent un ensemble de permissions, puis en liant ces rôles à des utilisateurs ou groupes via des **RoleBinding** (ou **ClusterRoleBinding**).

**Exemple complet** :
- Création d'un rôle `developer` :
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-team
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
```
- Association du rôle à un utilisateur :
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev-team
subjects:
- kind: User
  name: dev
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
Ainsi, tous les utilisateurs liés à ce rôle héritent des permissions définies. Une modification dans le rôle est immédiatement appliquée à tous les utilisateurs associés.

```  
  User / ServiceAccount
  ┌───────────────────────┐
  │ - kubectl ou API call │
  │ - Token / Cert        │
  └─────────────┬─────────┘
                │
                │ ① Requête envoyée au kube-apiserver
                ▼
        ┌─────────────────────────────┐
        │      Kubernetes API         │
        │       (kube-apiserver)      │
        └──────────────┬──────────────┘
                       │
                       │ ② kube-apiserver transmet la requête au RBAC Authorizer
                       ▼
            ┌─────────────────────────────┐
            │        RBAC Authorizer      │
            │ (Role-Based Access Control) │
            └──────────────┬──────────────┘
                           │
                           │ ③ Vérifie les associations RBAC
                           ▼
         ┌────────────────────────────────────────────────────────────┐
         │                 RBAC Objects dans le Cluster                │
         ├────────────────────────────────────────────────────────────┤
         │  Role / ClusterRole       │  RoleBinding / ClusterRoleBinding │
         ├───────────────────────────┼───────────────────────────────────┤
         │  Définit les permissions  │  Lie un utilisateur ou groupe à   │
         │  (ex : get, list, create) │  un Role ou ClusterRole           │
         └───────────────────────────┴───────────────────────────────────┘
                           │
                           │
            ┌──────────────▼──────────────┐
            │   Autorisation accordée ?   │
            └──────────────┬──────────────┘
                           │
          ┌────────────────┼─────────────────────┐
          │                │                     │
    Oui (Authorized)   Non (Denied)      Aucun Role trouvé
          │                │                     │
          │                │                     │
┌─────────▼─────────┐ ┌────▼───────────┐ ┌──────▼──────────┐
│ kube-apiserver    │ │ Accès refusé    │ │ Requête rejetée │
│ exécute la requête│ │ (HTTP 403)      │ │ (RBAC check fail)│
└─────────┬─────────┘ └─────────────────┘ └─────────────────┘
          │
          │
④ Requête autorisée → Accès aux ressources Kubernetes
─────────────────────────────────────────────────────────────────────────────
          │
          ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Ressources accessibles selon les Roles :                                │
│  - Pods, Deployments, Services, ConfigMaps, etc.                         │
│  - Limités au Namespace du Role, ou à tout le Cluster si ClusterRole.    │
└──────────────────────────────────────────────────────────────────────────┘
```

**Points clés du flux mis à jour**
1. L’**User** ou le **ServiceAccount** initie la requête avec son token ou certificat.
2. Le **kube-apiserver** reçoit la requête et la transmet au **RBAC Authorizer**.
3. Le **RBAC Authorizer** vérifie les permissions via **Roles/RoleBindings**.
4. Si autorisé → la requête est exécutée sur les ressources Kubernetes. Sinon → rejet (403).

***

### Webhook Authorizer
Le **Webhook Authorizer** permet de déléguer les décisions d'autorisation à un service externe via des appels API. Par exemple, **Open Policy Agent (OPA)** peut recevoir la requête d'autorisation du **kube-apiserver** et décider d'accorder ou refuser l'accès selon des règles avancées.

**Exemple complet** :
- Configurez le **kube-apiserver** avec l'option `--authorization-mode=Webhook` et spécifiez l'URL du service d'autorisation externe.
- À chaque requête, le **kube-apiserver** consulte l'OPA via HTTP pour obtenir la décision.

***

### AlwaysAllow et AlwaysDeny
- **AlwaysAllow** : Toutes les requêtes sont automatiquement permises, aucune vérification n'est faite.
- **AlwaysDeny** : Toutes les requêtes sont refusées systématiquement.
  Ces modes s'activent via l'option `--authorization-mode` du **kube-apiserver**.

***

## Configuration de plusieurs modes d'autorisation
Il est possible de combiner plusieurs modes via une liste séparée par des virgules, par exemple :
```bash
--authorization-mode=Node,RBAC,Webhook
```
Les requêtes sont testées successivement par chaque module, dans l'ordre. Si une requête est approuvée par un module, elle n'est plus testée par les suivants. Si elle est refusée, le contrôle passe au module suivant, jusqu'à une décision finale.

***

### Résumé essentiel

- L'**autorisation** contrôle ce que chaque utilisateur ou machine peut faire dans le **Cluster**.
- Plusieurs mécanismes existent : **Node Authorizer**, **ABAC**, **RBAC**, **Webhook**, **AlwaysAllow**, **AlwaysDeny**.
- **RBAC** est le système recommandé pour gérer facilement des rôles et permissions.
- Les permissions s'appliquent par utilisateur/groupe/ServiceAccount, et peuvent être limitées par **Namespace** grâce à la séparation logique.
- Le **kube-apiserver** peut utiliser plusieurs modes d'autorisation, testés séquentiellement.


