# Admission Controllers dans Kubernetes

## 1. Flux général d’une requête Kubernetes

Chaque requête envoyée via **kubectl** suit un chemin précis à travers le **plan de contrôle** :

```
Utilisateur (kubectl)
        ↓
Authentication → Authorization → Admission Controllers → etcd
```

1. **Requête vers kube-apiserver**  
   Le **kube-apiserver** reçoit toutes les requêtes (liste, création, suppression, etc.) et enregistre les états finaux des objets dans **etcd**.

2. **Authentication**  
   L’authentification identifie la source de la requête. Avec **kubectl**, cela s’appuie sur les certificats contenus dans le fichier **kubeconfig**.  
   *Exemple :* vérification du certificat client avant de traiter la commande :
   ```bash
   kubectl get pods --kubeconfig ~/.kube/config
   ```

3. **Authorization (RBAC)**  
   Une fois authentifié, **kube-apiserver** vérifie si l’utilisateur a le droit d’effectuer l’opération via **Role-Based Access Control (RBAC)**.
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: dev
     name: developer
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "create", "update", "delete"]
   ```
   Si le rôle permet l’action, la requête progresse. Sinon, elle est rejetée.

4. **Limites de RBAC**  
   RBAC contrôle uniquement les **opérations API** autorisées, pas le contenu des objets. Il ne peut pas, par exemple :
    - Interdire les images provenant de registres publics non sécurisés.
    - Interdire l’utilisation du tag `latest`.
    - Rejeter les containers tournant en tant que **root**.
    - Exiger des **labels** ou **annotations** spécifiques.

***

## 2. Admission Controllers : définition et rôle

Les **Admission Controllers** prolongent la sécurité et la conformité au-delà du RBAC. Ils interviennent **après l’autorisation**, pour :
- **Valider** une requête.
- **Modifier** la requête avant son enregistrement.
- **Refuser** la requête si elle ne respecte pas certaines règles.

Deux types :
- **Mutating Admission Controllers** : peuvent modifier les objets avant leur création (ex. injection automatique de secrets, sidecars, labels).
- **Validating Admission Controllers** : valident ou rejettent les requêtes selon des règles de conformité.

### Exemples d’Admission Controllers natifs

| Nom | Rôle |
|------|------|
| **AlwaysPullImages** | Force le téléchargement de l’image à chaque création de Pod. |
| **DefaultStorageClass** | Attribue automatiquement une StorageClass par défaut aux PVC. |
| **EventRateLimit** | Limite le volume de requêtes vers le kube-apiserver. |
| **NamespaceLifecycle** | Vérifie l’existence des Namespaces et empêche la suppression des namespaces système (`default`, `kube-system`, `kube-public`). |
| **PodSecurity** | Assure que les Pods respectent des règles de sécurité prédéfinies (remplace `PodSecurityPolicy`). |

***

## 3. Exemple pratique : Namespace Lifecycle

### Cas 1 : Namespace inexistant
```bash
kubectl run nginx --image=nginx --namespace=blue
```
Si le **Namespace `blue`** n’existe pas :
- La requête est **authentifiée** (certificats kubeconfig)
- Puis **autorisée** (RBAC du développeur)
- Enfin **refusée** par **NamespaceLifecycle**, car le Namespace n’existe pas.

### Cas 2 : Ancien workflow déprécié
Les anciens Admission Controllers :
- `NamespaceExists`  ❌
- `NamespaceAutoProvision`  ❌

sont désormais **remplacés** par `NamespaceLifecycle` ✅.

`NamespaceLifecycle` :
- Empêche la suppression des Namespaces essentiels.
- Rejette les requêtes dans des Namespaces inexistants.

***

## 4. Gestion et activation des Admission Controllers

### 🔍 Vérifier les contrôleurs activés
Sur un cluster **kubeadm** ou **EKS** :
```bash
ps -ef | grep kube-apiserver | grep enable-admission-plugins
```
ou, dans un Pod du plan de contrôle :
```bash
kubectl -n kube-system exec -it kube-apiserver-<nom> -- ps -ef | grep kube-apiserver
```

### ⚙️ Modifier la configuration
- Pour **activer** un Admission Controller :
  ```bash
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,AlwaysPullImages
  ```
- Pour **désactiver** un contrôleur :
  ```bash
  --disable-admission-plugins=EventRateLimit
  ```

Sous **Minikube**, on peut passer ces options au démarrage :
```bash
minikube start --extra-config=apiserver.enable-admission-plugins=NamespaceLifecycle,AlwaysPullImages
```

***

## 5. Résumé concis

* Les requêtes Kubernetes passent par : **kube-apiserver → Authentication → Authorization → Admission Controllers → etcd**.
* **RBAC** gère les permissions d’accès API, mais ne contrôle pas le contenu des objets.
* **Admission Controllers** apportent un niveau de validation et de conformité supplémentaire.
* **NamespaceLifecycle** remplace les anciens contrôleurs NamespaceExists et NamespaceAutoProvision.
* **EKS** active par défaut plusieurs admission plugins essentiels : `NamespaceLifecycle`, `LimitRanger`, `ServiceAccount`, `DefaultStorageClass`, `ResourceQuota`, `MutatingAdmissionWebhook`, `ValidatingAdmissionWebhook`, `PodSecurity`.
* **Good practice EKS :** toujours limiter la portée des webhooks et ajouter un mode de repli en cas d’erreur.

***

```
Utilisateur (kubectl)
        |
        v
+-----------------------+
| kube-apiserver         |
+-----------------------+
        |
        | 1️⃣ Authentication
        |   - Vérifie identité via certificats kubeconfig
        |   - Résultat :
        |       ✔ Authenticated → passe à Authorization
        |       ✖ Rejet → fin
        v
+-----------------------+
| Authorization (RBAC)  |
+-----------------------+
        |   - Vérifie permissions de l'utilisateur
        |   - Exemple : developer peut create Pods
        |   - Résultat :
        |       ✔ Authorized → passe à Admission Controllers
        |       ✖ Rejet → fin
        v
+-----------------------+
| Admission Controllers |
+-----------------------+
        |
        | 3️⃣ Validation et modifications possibles
        |
        |-------------------------------|
        | Controller: NamespaceExists    |
        |   - Vérifie si namespace existe
        |   - ✖ Rejet si non existant
        |
        | Controller: NamespaceAutoProvision
        |   - Crée automatiquement le namespace si inexistant
        |
        | Controller: AlwaysPullImages
        |   - Force le pull de l'image pour le Pod
        |
        | Controller: DefaultStorageClass
        |   - Ajoute une StorageClass par défaut aux PVC si non spécifié
        |
        | Controller: EventRateLimit
        |   - Limite le nombre de requêtes simultanées à kube-apiserver
        |
        | Autres controllers possibles (ex : enforce labels, disallow root user)
        |-------------------------------|
        v
+-----------------------+
| Validation finale      |
| - ✔ Request acceptée   |
| - ✖ Request rejetée    |
+-----------------------+
        |
        v
+-----------------------+
| etcd (Cluster DB)      |
| - Objet final stocké   |
+-----------------------+

```
**Exemple concret : Pod dans namespace `blue`** :
1. Commande :
   ```
   kubectl create pod nginx --namespace=blue
   ```
2. **Authentication** : OK
3. **RBAC Authorization** : developer peut créer → OK
4. **Admission Controllers** :
    * NamespaceExists → `blue` n’existe pas → ✖ rejet
    * NamespaceAutoProvision → crée `blue` automatiquement → ✔ accepté
    * AlwaysPullImages → force pull de l’image `nginx`
5. Objet final → créé dans **etcd**

