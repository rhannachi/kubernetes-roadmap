## Sécuriser l’accès à un cluster Kubernetes

### 1. Introduction

Un **Cluster Kubernetes** est un ensemble de **nœuds** (physiques ou virtuels) et de **composants logiciels** qui collaborent pour exécuter des applications, appelées *workloads*.  
Plusieurs types d’acteurs interagissent avec ce cluster :

* **Administrateurs** : assurent la maintenance et la gestion du cluster.
* **Développeurs** : déploient et testent leurs applications.
* **Utilisateurs finaux** : consomment les applications déployées.
* **Applications tierces** : interagissent avec le cluster via son API.

Dans cette leçon, nous allons apprendre à **sécuriser l’accès au cluster Kubernetes** en explorant les **mécanismes d’authentification**.  
L’objectif est de comprendre comment le **kube-apiserver** valide l’identité des entités (humaines ou machines) qui envoient des requêtes au cluster.

***

### 2. Les types d’utilisateurs dans Kubernetes

Kubernetes distingue deux grandes familles d’utilisateurs :

1. **Utilisateurs humains** — administrateurs et développeurs.
2. **Utilisateurs automatisés** — services, pods ou pipelines (CI/CD) qui accèdent au cluster pour exécuter des tâches. Cette catégorie inclut aussi les composants internes Kubernetes (kubelet, controller-manager…) qui s’authentifient via des *ServiceAccounts*.

> ⚠️ Les **utilisateurs finaux** des applications ne sont **pas gérés** par Kubernetes.  
> Leur authentification est assurée **au niveau de l’application** elle-même.

***

### 3. Gestion des comptes utilisateurs

#### 3.1. Utilisateurs humains

Kubernetes **ne gère pas directement les comptes utilisateurs** au sens d’un annuaire interne ou d’une base de données.  
Il délègue cette responsabilité à des **sources d’identité externes**, telles que :

* Des **fichiers statiques** (identifiants ou tokens).
* Des **certificats clients x509**.
* Des **fournisseurs d’identité** comme **LDAP**, **Kerberos** ou **OIDC (OpenID Connect)**.

👉 Cela signifie que vous ne pouvez **ni créer** ni **lister** des utilisateurs avec `kubectl`.  
La création et la gestion des comptes se font côté fournisseur d’identité externe, et `kubectl` interagit via le fichier kubeconfig configuré.

#### Complément :
En production, il est recommandé d’utiliser un fournisseur d’identités OIDC (ex. Dex, Keycloak, AWS IAM, Azure AD) pour gérer de façon centralisée et sécurisée vos utilisateurs.

***

#### 3.2. Comptes de service (ServiceAccounts)

Contrairement aux utilisateurs humains, Kubernetes **gère nativement les comptes de service**, utilisés par les **applications internes** au cluster.

Exemple : création et utilisation d’un `ServiceAccount` :

```bash
kubectl create serviceaccount my-app-sa
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa
  containers:
    - name: app
      image: nginx
```

Ces comptes sont liés à des *Secrets* contenant leurs jetons qui permettent aux pods de s’authentifier auprès du serveur API.

***

### 4. Le rôle du kube-apiserver

Toutes les requêtes vers le cluster passent par le **kube-apiserver**, le point d’entrée unique de l’API Kubernetes.  
Il joue deux rôles essentiels :

1. **Authentification** : vérifier l’identité de l’émetteur de la requête.
2. **Autorisation** : vérifier si cet utilisateur a le droit d’effectuer l’action demandée.

Dans cette section, nous nous concentrerons sur la **phase d’authentification**.

***

### 5. Les mécanismes d’authentification pris en charge

Le **kube-apiserver** peut être configuré pour accepter plusieurs types d’authentification :

| Type                                         | Description                                  | Niveau de sécurité       |
| -------------------------------------------- | -------------------------------------------- | ------------------------ |
| **Fichier de mots de passe**                 | Fichier CSV statique contenant identifiants | ⚠️ Faible (à éviter)     |
| **Fichier de tokens**                        | Authentification via jeton statique          | ⚠️ Faible (à éviter)     |
| **Certificats x509**                         | Authentification basée sur une CA Kubernetes | ✅ Élevé (méthode privilégiée) |
| **Services externes (LDAP, OIDC, Kerberos)** | Délégation à un service d’identité           | ✅ Élevé (à privilégier) |

> Les fichiers statiques sont utiles uniquement pour les tests et environnements locaux.

***

## 🧭 Partie pratique : Configurer l’authentification

### 6. Authentification avec un fichier de mots de passe

#### 6.1. Exemple de fichier CSV

```
password,user,uid,"group"
mypassword,admin,1,"system:masters"
devpass,developer,2,"dev-team"
```

#### 6.2. Configuration du kube-apiserver

```bash
kube-apiserver --basic-auth-file=/etc/kubernetes/users.csv
```

#### 6.3. Connexion avec `kubectl`

```bash
kubectl get pods --username=admin --password=mypassword
```

> ❌ Inconvénient : les mots de passe sont stockés en clair → méthode **non sécurisée**, à éviter en production.

***

### 7. Authentification par token statique

#### 7.1. Exemple de fichier de tokens

```
token,user,uid,"group"
abc123,admin,1,"system:masters"
xyz456,developer,2,"dev-team"
```

#### 7.2. Configuration du kube-apiserver

```bash
kube-apiserver --token-auth-file=/etc/kubernetes/tokens.csv
```

#### 7.3. Exemple d’appel API

```bash
curl -H "Authorization: Bearer abc123" https://<api-server>:6443/api/v1/pods
```

> ⚠️ Comme pour les mots de passe, cette méthode est utile pour les tests mais **non adaptée à la production**.

***

### 8. Authentification par certificats x509

Le **kube-apiserver** peut authentifier les utilisateurs à l’aide de **certificats x509**, signés par la **CA (Certificate Authority)** du cluster.

* Le **CN (Common Name)** → nom d’utilisateur.
* L’attribut **O (Organization)** → groupe d’appartenance.

Ce mécanisme est **sécurisé** et constitue la **méthode privilégiée** dans Kubernetes.

***

#### 8.1. Étape 1 – Génération de la clé et de la CSR

```bash
openssl genrsa -out developer.key 2048
openssl req -new -key developer.key -out developer.csr -subj "/CN=developer/O=dev-team"
```

#### 8.2. Étape 2 – Signature par la CA du cluster

```bash
openssl x509 -req -in developer.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out developer.crt -days 365
```

#### 8.3. Étape 3 – Configuration du profil utilisateur dans `kubectl`

```bash
kubectl config set-credentials developer \
  --client-certificate=developer.crt \
  --client-key=developer.key

kubectl config set-context developer-context \
  --cluster=kubernetes-cluster \
  --user=developer

kubectl config use-context developer-context
```

✅ L’utilisateur `developer` peut désormais interagir avec le cluster tant que son certificat est valide.

> Note : La gestion de la révocation et du renouvellement des certificats est essentielle en production.

***

### 9. Vérification de l’accès

Test simple :

```bash
kubectl get pods
```

Si le certificat est valide et reconnu, la requête réussit.  
Sinon, vous obtiendrez :

```
error: You must be logged in to the server (Unauthorized)
```

> 💡 **Astuce :** les composants internes de Kubernetes (kubelet, controller-manager, scheduler…) utilisent eux aussi des certificats x509 pour s’authentifier auprès du cluster.

***

## 🧩 Conclusion

L’authentification dans Kubernetes repose sur une logique simple mais puissante :

1. **Aucune gestion interne des utilisateurs humains.**
2. **Appui sur des mécanismes standards** (certificats, tokens, providers externes).
3. **Séparation claire** entre *authentification* (qui suis-je ?) et *autorisation* (ai-je le droit ?).

👉 Pour une utilisation en production, privilégiez **OIDC** ou les **certificats x509**, et évitez les fichiers statiques.

***

## Points complémentaires pour la sécurité en production

- Préférez un fournisseur d’identité OIDC (ex. Dex, Keycloak, AWS IAM, Azure AD) pour gérer les utilisateurs humains et intégrer l’authentification unique (SSO) et la gestion centralisée des identités.

- Restreignez l’accès réseau au kube-apiserver via des firewalls, security groups ou règles CIDR.

- Utilisez fortement **RBAC** (Role-Based Access Control) pour contrôler finement les permissions après authentification.

- Sur AWS EKS, l’authentification s’appuie sur IAM et l’authentificateur AWS IAM, avec des mappings dans `aws-auth` ConfigMap pour relier identités IAM et groupes RBAC Kubernetes.

- Les fichiers de mots de passe et tokens statiques ne sont adaptés qu’aux environnements de test et ne doivent jamais être utilisés en production.

***

## Schéma ASCII clair et pédagogique du flux d’authentification Kubernetes

```
                         +-----------------------------+
                         |        Utilisateur          |
                         | (kubectl / API client / App)|
                         +-------------+---------------+
                                       |
                                       | 1️⃣ Envoie une requête HTTP
                                       |    (kubectl get pods, etc.)
                                       v
+---------------------------------------------------------------+
|                        kube-apiserver                         |
|---------------------------------------------------------------|
|  2️⃣ Vérifie l'identité (AUTHENTICATION)                      |
|     ├── Basic Auth (user/pass)                                |
|     ├── Token statique (Bearer token)                         |
|     ├── Certificat x509 (TLS)                                 |
|     └── Fournisseur externe (OIDC, LDAP, Kerberos)            |
|                                                               |
|  3️⃣ Vérifie les permissions (AUTHORIZATION)                   |
|     ├── RBAC (Role-Based Access Control)                      |
|     └── ABAC / Webhook / Node                                 |
|                                                               |
|  4️⃣ Admission Controllers (politiques de validation, quotas)  |
|---------------------------------------------------------------|
|  ✅ Si tout est OK → la requête est transmise à l’étape suivante
|  ❌ Sinon → "Unauthorized" ou "Forbidden"
+---------------------------------------------------------------+
                                       |
                                       | 5️⃣ Requête validée
                                       v
+---------------------------------------------------------------+
|                       etcd (Base de données)                  |
|  - Stocke l’état du cluster                                   |
|  - Contient les objets Kubernetes (Pods, Deployments, etc.)   |
+---------------------------------------------------------------+
```

***

### Lecture du schéma

* **(1)** L’utilisateur ou le service envoie une requête à l’API Kubernetes.
* **(2)** Le **kube-apiserver** authentifie l’émetteur (identité).
* **(3)** Il vérifie ensuite les droits d’accès (autorisation via RBAC, etc.).
* **(4)** Si les contrôles passent, la requête est **admise**.
* **(5)** Le **cluster exécute** ou **lit** l’état demandé dans **etcd**.

