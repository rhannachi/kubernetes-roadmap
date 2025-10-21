# Admission Controller

Lorsque nous utilisons la ligne de commande avec l’outil **kubectl** pour effectuer des opérations sur notre **Kubernetes cluster**, chaque requête passe par un processus précis :

1. **Requête vers le kube-apiserver** :\
   Par exemple, lorsqu’on crée un **Pod**, la requête est envoyée au **kube-apiserver**, puis le Pod est créé et l’information est persistée dans la base **etcd**.

2. **Authentification (Authentication)** :\
   La requête doit d’abord passer par un processus d’authentification, généralement basé sur des certificats.
    * Si la requête provient de **kubectl**, le fichier **kubeconfig** contient les certificats nécessaires.
    * L’authentification identifie l’utilisateur et vérifie qu’il est valide.

3. **Autorisation (Authorization)** :\
   Une fois authentifié, **kube-apiserver** vérifie si l’utilisateur a le droit d’exécuter l’action demandée, via **Role-Based Access Control (RBAC)**.
    * Exemple : un utilisateur avec le rôle **developer** peut **list**, **get**, **create**, **update** ou **delete Pods**.
    * Si la requête correspond aux permissions, elle est autorisée ; sinon, elle est rejetée.
    * RBAC permet aussi de restreindre l’accès à certains **Namespaces** ou à des noms de ressources spécifiques (ex : Pods nommés `blue` ou `orange`).

4. **Limites de RBAC** :\
   RBAC agit uniquement au niveau de l’API, et ne peut pas contrôler certains aspects des objets eux-mêmes, comme :
    * Interdire l’utilisation d’images provenant de Docker Hub public.
    * Interdire l’usage du tag `latest`.
    * Interdire l’exécution d’un container en tant que **root**.
    * Imposer que certains labels soient toujours présents.

5. **Admission Controllers** :\
   Pour aller au-delà de RBAC, Kubernetes utilise les **Admission Controllers**, qui peuvent :
    * Valider les requêtes.
    * Modifier la requête avant sa création.
    * Effectuer des opérations supplémentaires côté cluster.

   **Exemples :**
    * **AlwaysPullImages** : force le pull des images à chaque création de Pod.
    * **DefaultStorageClass** : applique automatiquement une **StorageClass** aux **PVC** si aucune n’est spécifiée.
    * **EventRateLimit** : limite le nombre de requêtes envoyées au **kube-apiserver** pour éviter un flood.
    * **NamespaceExists** : rejette les requêtes pour des **Namespaces** inexistants.

   **Exemple pratique :**
   ```
   $ kubectl create pod nginx --namespace=blue
   ```
    * Si le **Namespace `blue`** n’existe pas, la requête passe par l’authentification et l’autorisation, puis est rejetée par **NamespaceExists**.
    * Si on active **NamespaceAutoProvision**, le namespace est créé automatiquement avant la création du Pod.

6. **Gestion des Admission Controllers** :\
    * Pour lister les plugins activés par défaut :
      ```
      $ kube-apiserver --enable-admission-plugins | grep enable-admission-plugins
      ```
    * Pour activer un admission controller, modifier le flag `--enable-admission-plugins` dans le **kube-apiserver manifest** ou dans le service si kubeadm est utilisé.
    * Pour désactiver, utiliser `--disable-admission-plugins`.

7. **Évolution : Namespace Lifecycle** :\
   Les admission controllers **NamespaceExists** et **NamespaceAutoProvision** sont désormais dépréciés, remplacés par **NamespaceLifecycle**, qui :
    * Rejette les requêtes sur des namespaces inexistants.
    * Empêche la suppression des namespaces par défaut : `default`, `kube-system`, et `kube-public`.

---

### Résumé concis

* Les requêtes **kubectl** passent par : **kube-apiserver → Authentication → Authorization → Admission Controllers → etcd**.
* **RBAC** : contrôle les permissions des utilisateurs au niveau API (list, get, create, update, delete).
* **Admission Controllers** : vont au-delà de RBAC, permettent de valider ou modifier des objets avant leur création.

    * Exemples : **AlwaysPullImages**, **DefaultStorageClass**, **EventRateLimit**, **NamespaceExists**, **NamespaceAutoProvision**.
* **NamespaceLifecycle** remplace les anciens admission controllers liés aux namespaces, en sécurisant leur création et leur suppression.

---

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

