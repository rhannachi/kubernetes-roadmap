# ClusterRole et ClusterRoleBinding

## 1. Introduction
Dans Kubernetes, les **ClusterRoles** et **ClusterRoleBindings** permettent de gérer les **autorisations globales au niveau du cluster** (Cluster Scoped), contrairement aux **Roles** et **RoleBindings**, qui sont **limités à un namespace** (Namespaced).

Les droits sont gérés via le mécanisme **RBAC** (Role-Based Access Control).

***

## 2. Concepts clés

| Terme | Niveau | Description |
|--------|----------|-------------|
| **Namespace** | Namespaced | Un espace logique isolant les ressources. |
| **Namespaced Resource** | Oui | Ressources spécifiques à un namespace (Pods, Deployments, ConfigMaps...). |
| **Cluster Scoped Resource** | Non | Ressources globales au cluster (Nodes, Namespaces, PersistentVolumes...). |

### Vérification avec `kubectl`
- Pour lister les ressources non liées à un namespace:
  ```
  $ kubectl api-resources --namespaced=false
  ```
- Pour lister celles qui le sont:
  ```
  $ kubectl api-resources --namespaced=true
  ```

***

## 3. ClusterRole
Une **ClusterRole** définit un ensemble de permissions applicables à l’échelle du cluster.

Exemple d’un `ClusterRole` pour un **Storage Admin** :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-admin
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "create", "delete"]
```

➡️ Cette **ClusterRole** donne les droits de gestion des volumes et des classes de stockage.

***

## 4. ClusterRoleBinding
Une **ClusterRoleBinding** lie une ClusterRole à un utilisateur, un groupe ou un service account pour accorder ces autorisations à l’échelle du cluster.

Exemple:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: storage-admin-binding
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io
```

Cet exemple lie la **ClusterRole** `storage-admin` à l’utilisateur `alice@example.com`.

***

## 5. Comparaison : ClusterRole vs Role
| Type | Scopé par Namespace | Portée |
|-------|----------------------|---------|
| **Role** | Oui | Un seul namespace |
| **ClusterRole** | Non | Tout le cluster |

Les **ClusterRoles** peuvent être utilisées aussi **dans un Binding Namespaced (RoleBinding)** pour accorder ces droits dans un seul namespace, ce qui permet de réutiliser les règles globales au niveau local.

Exemple:
```bash
kubectl create rolebinding view-storage \
  --clusterrole=storage-admin \
  --user=bob@example.com \
  --namespace=development
```
➡️ Ici, `bob` obtient les permissions définies dans la ClusterRole `storage-admin` **uniquement dans le namespace `development`**.

***

## 6. Exemple pratique : Cluster Admin
Kubernetes fournit un rôle intégré `cluster-admin` :

```bash
kubectl get clusterrole cluster-admin -o yaml
```
Ce rôle donne tous les droits sur l’ensemble du cluster : il est donc à utiliser avec prudence !

Pour donner accès à un utilisateur:
```bash
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=admin@example.com
```

***

## 7. Exercices pratiques à tester
1. **Lister les ressources globales et namespaced**
   ```bash
   kubectl api-resources --namespaced=false
   kubectl api-resources --namespaced=true
   ```
2. **Créer un ClusterRole pour un storage admin** (comme ci-dessus)
3. **Associer cette ClusterRole à un utilisateur** via un ClusterRoleBinding
4. **Tester avec un RoleBinding** dans un namespace spécifique
5. **Consulter les ClusterRoles existantes**:
   ```bash
   kubectl get clusterroles
   ```

***

## 8. Résumé visuel simplifié
```
+---------------------+      +--------------------------+
| ClusterRole         | ---> | ClusterRoleBinding       |
| (Ex: storage-admin) |      | Lie la ClusterRole à un  |
|                     |      | utilisateur/groupe/SA    |
+---------------------+      +-----------+--------------+
                                        |
                                        v
                             +-------------------------+
                             | Utilisateur ou Service  |
                             | Account (accède selon   |
                             | les règles définies)    |
                             +-------------------------+
```

***

## Solution :
## 1. Lister les ressources globales et namespaced

### Commandes :
```
$ kubectl api-resources --namespaced=false
$ kubectl api-resources --namespaced=true
```

### Explication:
- La première commande affiche les ressources **Cluster Scoped** comme `nodes`, `namespaces`, `persistentvolumes`.
- La seconde affiche les ressources **Namespaced**, comme `pods`, `deployments`, `configmaps`, `services`.

***

## 2. Créer un ClusterRole pour un storage admin

### Fichier `storage-admin-clusterrole.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-admin
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "create", "delete"]
```

### Application:
```
$ kubectl apply -f storage-admin-clusterrole.yaml
```

### Vérification:
```
$ kubectl get clusterroles | grep storage-admin

$ kubectl describe clusterrole storage-admin
Name:         storage-admin
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources                      Non-Resource URLs  Resource Names  Verbs
  ---------                      -----------------  --------------  -----
  persistentvolumes              []                 []              [get list create delete]
  storageclasses.storage.k8s.io  []                 []              [get list create delete]
```

***

## 3. Lier la ClusterRole à un utilisateur

### Fichier `storage-admin-binding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: storage-admin-binding
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io
```

### Application:
```
$ kubectl apply -f storage-admin-binding.yaml
```

### Vérification:
```
$ kubectl get clusterrolebindings | grep storage-admin-binding
$ kubectl describe clusterrolebinding storage-admin-binding
```

***

## 4. Tester un RoleBinding Namespaced utilisant la même ClusterRole

### Exemple : namespace `dev`
```bash
kubectl create namespace dev
kubectl create rolebinding storage-dev-access \
  --clusterrole=storage-admin \
  --user=bob@example.com \
  --namespace=dev
```

### Vérification:
```bash
kubectl describe rolebinding storage-dev-access -n dev
```

➡ Cela permet à `bob@example.com` d’utiliser la ClusterRole `storage-admin` dans **le seul namespace `dev`**.

***

## 5. Consulter les ClusterRoles préinstallées

```bash
kubectl get clusterroles
```

Tu verras par exemple :
```
NAME            AGE
admin           2d
cluster-admin   2d
edit            2d
view            2d
storage-admin   10m
```

Pour voir le détail d’un rôle donné :
```bash
kubectl describe clusterrole cluster-admin
```

***

## 6. Test des droits et validation RBAC

Vérifie les permissions d’un utilisateur simulé :
```bash
kubectl auth can-i create persistentvolumes --as alice@example.com
kubectl auth can-i delete pods --as alice@example.com
```

- Si la première commande renvoie **yes**, ton `ClusterRoleBinding` fonctionne bien.
- Si la seconde renvoie **no**, c’est attendu, car la permission n’a pas été donnée.

***

## 7. Nettoyage (optionnel)
Supprimer les objets créés pour le test :
```bash
kubectl delete clusterrole storage-admin
kubectl delete clusterrolebinding storage-admin-binding
kubectl delete rolebinding storage-dev-access -n dev
kubectl delete namespace dev
```

***