# **Kubernetes – Roles, RoleBindings et kube-proxy (Minikube)**

## Objectifs pédagogiques

* Comprendre les **principes du RBAC** (Role-Based Access Control) dans Kubernetes.
* Créer et appliquer des **Roles** et **RoleBindings**.
* Tester les accès avec `kubectl auth can-i`.
* Découvrir le fonctionnement du **kube-proxy** (service réseau).

## **Partie 1 – Préparation de l’environnement**

### Étapes de démarrage :

```
# Démarrer Minikube
$ minikube start

# Créer deux namespaces pour les tests
$ kubectl create namespace dev
$ kubectl create namespace test
```

✅ **Vérification :**

```
$ kubectl get ns
```

---

## **Exercice 1 – Créer un Role avec permissions limitées**

**Objectif :** créer un Role dans le namespace `dev` qui permet uniquement de **voir** et **lister** les pods.

### Étapes :

Créer un fichier `role-view-pods.yaml` :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

Appliquer le rôle :

```
$ kubectl get role -n dev
NAME         CREATED AT
pod-viewer   2025-10-20T08:24:01Z

$ kubectl describe role pod-viewer -n dev
Name:         pod-viewer
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list]
```

✅ **Vérification :**

```
$ kubectl get role -n dev
$ kubectl describe role pod-viewer -n dev
```

---

## **Exercice 2 – Créer un ServiceAccount et un RoleBinding**

**Objectif :** donner à un **ServiceAccount** (`dev-user`) les permissions définies dans le Role.

### Étapes :

Créer le ServiceAccount :

```
$ kubectl create serviceaccount dev-user -n dev
```

Créer un fichier `rolebinding-dev-user.yaml` :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-pod-viewer
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

Appliquer :

```
$ kubectl apply -f rolebinding-dev-user.yaml
```

✅ **Vérification :**

```
$ kubectl get rolebinding -n dev
NAME              ROLE              AGE
bind-pod-viewer   Role/pod-viewer   82s

$ kubectl describe rolebinding bind-pod-viewer -n dev
Name:         bind-pod-viewer
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  pod-viewer
Subjects:
  Kind            Name      Namespace
  ----            ----      ---------
  ServiceAccount  dev-user  dev
```

---

## **Exercice 3 – Vérifier les permissions du ServiceAccount**

**Objectif :** tester concrètement les accès de `dev-user`.

### Tester avec `kubectl auth can-i`

```
$ kubectl auth can-i list pods --as=system:serviceaccount:dev:dev-user -n dev
$ kubectl auth can-i get pods --as=system:serviceaccount:dev:dev-user -n dev
$ kubectl auth can-i delete pods --as=system:serviceaccount:dev:dev-user -n dev
```

✅ **Résultats attendus :**

| Commande    | Résultat attendu |
| ----------- | ---------------- |
| list pods   | yes              |
| get pods    | yes              |
| delete pods | no               |

---

## **Exercice 4 – Tester l’isolation entre namespaces**

**Objectif :** montrer que le Role du namespace `dev` ne s’applique **pas** au namespace `test`.

```
$ kubectl auth can-i list pods --as=system:serviceaccount:dev:dev-user -n test
```

✅ **Résultat attendu :** `no`
Cela prouve que les **Roles sont limités à leur namespace**.

---

## **Exercice 5 – Étendre le rôle (modifier les permissions)**

**Objectif :** ajouter la possibilité de **créer** des pods dans le namespace `dev`.

1. Modifier le Role :

```
$ kubectl edit role pod-viewer -n dev
```

2. Ajouter `create` dans la section `verbs` :

```yaml
verbs: ["get", "list", "create"]
```

3. Sauvegarder et tester :

```
$ kubectl auth can-i create pods --as=system:serviceaccount:dev:dev-user -n dev
```

✅ **Résultat attendu :** `yes`

---

# **Partie 2 – kube-proxy : fonctionnement et diagnostic**

## **Exercice 7 – Comprendre kube-proxy**

### Q1. Qu’est-ce que kube-proxy ?

> `kube-proxy` est un composant du plan de données Kubernetes.
> Il gère la **connectivité réseau** entre les Services et les Pods, en créant des règles de routage locales sur chaque nœud (iptables ou IPVS).

### Q2. Quels sont les modes possibles ?

| Mode      | Description                                                 |
| --------- | ----------------------------------------------------------- |
| userspace | Obsolète, redirige le trafic via un processus utilisateur   |
| iptables  | Par défaut, gère le routage via des règles iptables         |
| ipvs      | Plus performant, gère le routage via le kernel Linux (IPVS) |

### Q3. Comment vérifier le mode utilisé sur ton cluster Minikube ?

```
$ kubectl -n kube-system get pods -l k8s-app=kube-proxy
$ kubectl -n kube-system logs -l k8s-app=kube-proxy | grep "Using"
```

✅ Exemple de sortie :

```
Using iptables Proxier
```

### Q4. Que fait kube-proxy lorsqu’un Service est créé ?

> * Il observe les objets `Service` et `Endpoints`.
> * Il crée des règles pour diriger le trafic vers les Pods correspondants.
> * Il assure la **répartition du trafic** entre les Pods disponibles.

---

## **Exercice 8 – Expérience pratique kube-proxy**

1. Crée un simple Pod :

```
$ kubectl run nginx --image=nginx --port=80
```

2. Expose-le via un Service :

```
$ kubectl expose pod nginx --type=ClusterIP --port=80
```

3. Liste les règles réseau appliquées :

```
$ kubectl get svc
minikube ssh
sudo iptables -t nat -L KUBE-SERVICES | grep nginx
```

Tu verras des règles créées automatiquement par `kube-proxy` pour router le trafic vers ton pod NGINX.

