### **Rôles et permissions dans Kubernetes (RBAC)**

La création d’un **Role** dans Kubernetes permet de définir un ensemble précis d’autorisations pour des utilisateurs ou des processus au sein d’un **namespace**.

***

#### **1. Création d’un Role**

Un **Role** est un objet Kubernetes défini dans un fichier YAML qui spécifie un ensemble de permissions dans un namespace donné.
Pour le créer, on définit l’API version suivante :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

**Explications :**

* `apiGroups`: indique le groupe d’API où se trouve la ressource :
    * Pour les ressources du **core group** (comme `pods` et `configmaps`), le champ reste vide `[""]`.
    * Pour les ressources du groupe **apps** (comme `deployments`), on précise `"apps"`.
* `resources`: liste des types de ressources sur lesquelles le Role accorde des permissions.
* `verbs`: liste des actions autorisées sur ces ressources :
    * `get`, `list` → lecture
    * `create`, `update`, `patch`, `delete` → écriture et suppression.

Ce **Role** accorde à un utilisateur ou groupe associé les permissions suivantes :

* Gérer les **Pods** (lecture, création et suppression).
* Créer des **ConfigMaps**.
* Lire, créer, modifier, mettre à jour et supprimer des **Deployments** dans le namespace concerné.

Application du Role :

```bash
kubectl apply -f role-developer.yaml
```

***

#### **2. Liaison d’un utilisateur au Role avec un RoleBinding**

Pour permettre à un utilisateur d’utiliser un **Role**, il faut créer un objet **RoleBinding**.  
Un **RoleBinding** associe un utilisateur, un groupe ou un **ServiceAccount** à un **Role** spécifique dans un namespace.

Avant de créer le binding pour un utilisateur humain, celui-ci doit être présent dans la configuration Kubernetes, généralement dans `~/.kube/config`.

Exemple d’entrée utilisateur :
```yaml
users:
- name: dev-user
  user:
    client-certificate: /home/dev/.kube/dev-user.crt
    client-key: /home/dev/.kube/dev-user.key
```

Créons maintenant le **RoleBinding** :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Application du binding :
```bash
kubectl apply -f rolebinding-developer.yaml
```

Ainsi, l’utilisateur `dev-user` aura accès aux **Pods** et **ConfigMaps** définis dans le **Role** `developer` du namespace courant.

Pour restreindre cet accès à un autre namespace, indiquez le nom de celui-ci dans `metadata.namespace`.

***

#### **3. Exemple avec un ServiceAccount**

Un **ServiceAccount** est une entité gérée par Kubernetes, utilisée par les Pods ou les processus internes du cluster pour s’authentifier auprès de l’API serveur.

**Étape 1 : Créer le ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-developer
  namespace: default
```

Application :
```bash
kubectl apply -f serviceaccount-developer.yaml
```

**Étape 2 : Créer le RoleBinding pour ce ServiceAccount**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sa-developer-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: sa-developer
  namespace: default
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Application :
```bash
kubectl apply -f rolebinding-serviceaccount.yaml
```

**Étape 3 : Exemple d’utilisation**

On peut ensuite exécuter un Pod qui utilise ce ServiceAccount :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: developer-pod
spec:
  serviceAccountName: sa-developer
  containers:
  - name: test
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
```

Ce Pod fonctionne avec les permissions associées au **Role** `developer`.  
Il pourra par exemple créer ou lister des **Pods** et des **ConfigMaps**, mais ne pourra pas créer de **Deployments**.

***

#### **4. Visualisation des objets Role et RoleBinding**

Lister les Roles :
```bash
kubectl get roles
```

Lister les RoleBindings :
```bash
kubectl get rolebindings
```

Afficher les détails :
```bash
kubectl describe role developer
kubectl describe rolebinding devuser-developer-binding
kubectl describe rolebinding sa-developer-binding
```

***

#### **5. Vérifier les permissions d’un utilisateur ou ServiceAccount**

Tester les autorisations :
```bash
kubectl auth can-i <action> <resource> --as=<user>
```

Exemples :
```bash
kubectl auth can-i create deployments --as=dev-user
kubectl auth can-i create pods --as=dev-user
kubectl auth can-i create pods --as=system:serviceaccount:default:sa-developer
```

Le **prefixe `system:serviceaccount:<namespace>:<nom>`** est essentiel pour cibler un ServiceAccount.

***

#### **6. Restreindre l’accès à des ressources spécifiques**

Vous pouvez limiter les accès à certaines ressources précises grâce à `resourceNames` :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: partial-pod-access
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
  resourceNames: ["blue-pod", "orange-pod"]
```

Ce Role autorise l’accès uniquement aux Pods `blue-pod` et `orange-pod` dans le namespace défini.

***

### **Résumé concis**

- Un **Role** définit des permissions pour des ressources dans un namespace.
- Un **RoleBinding** associe ce Role à un **utilisateur** (défini dans `~/.kube/config`) ou à un **ServiceAccount**.
- Les **ServiceAccounts** sont recommandés pour les Pods et les applications internes du cluster.
- Les commandes `kubectl get`, `describe`, et `auth can-i` permettent de visualiser et de valider les droits d’accès.
- Le champ `resourceNames` affine les permissions à des objets spécifiques.

