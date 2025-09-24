## les ServiceAccounts dans Kubernetes

### Qu’est-ce qu’un ServiceAccount ?
Un **ServiceAccount** est une identité utilisée par les applications (Pods) internes au cluster Kubernetes pour s’authentifier auprès de l’API server.\
Contrairement aux comptes utilisateurs, qui sont destinés aux humains (administrateurs, développeurs), les ServiceAccounts sont destinés aux machines et processus automatisés.

### Pourquoi utiliser un ServiceAccount ?
Les applications déployées dans un cluster ont souvent besoin de communiquer avec le **kube-apiserver** pour récupérer des informations ou effectuer des actions.\
Un ServiceAccount fournit un jeton d’authentification sécurisé pour que ces applications puissent s’authentifier et interagir avec l’API Kubernetes.

### Exemple de création et utilisation simple d’un ServiceAccount

#### 1. Créer un ServiceAccount

Exécutez la commande suivante pour créer un ServiceAccount nommé `myapp-sa` dans le namespace courant :

```
$ kubectl create serviceaccount myapp-sa
```

#### 2. Vérifier la création du ServiceAccount

Pour lister les ServiceAccounts dans le namespace :

```
$ kubectl get serviceaccount
NAME       SECRETS   AGE
default    0         48d
myapp-sa   0         3s
```

Pour affiche des informations détaillées sur le ServiceAccount myapp-sa
``` 
$ kubectl describe serviceaccount myapp-sa
```

#### 3. Déployer un Pod utilisant ce ServiceAccount

Voici un extrait YAML d’un Pod qui utilise le ServiceAccount `myapp-sa` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: myapp-container
    image: busybox
    command: ["sleep", "3600"]
```

Déployez ce Pod avec :

```
$ kubectl apply -f myapp-pod.yaml
```

Ce Pod montera automatiquement le jeton du ServiceAccount `myapp-sa` dans le chemin `/var/run/secrets/kubernetes.io/serviceaccount` à l’intérieur du conteneur.\
L’application peut lire ce jeton pour s’authentifier auprès de l’API Kubernetes.
```
$ kubectl get pod myapp-pod -o yaml
...
spec:
  containers:
  - name: myapp-container
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-tftxv
      readOnly: true
...
```

***

### Accéder au jeton d’un ServiceAccount

Avant Kubernetes 1.24, un objet **Secret** contenant le jeton du ServiceAccount était automatiquement créé et lié au ServiceAccount.\
Depuis 1.24, il faut générer un jeton à la demande.

Si ton cluster utilise une version antérieure :
* Le token est dans un Secret, utilisable via :
```
kubectl get secret -n <namespace>
kubectl describe secret <nom-du-secret> -n <namespace>
```

#### Générer/Afficher un jeton manuel (Kubernetes 1.24+)

Pour générer/afficher un jeton OAuth à usage unique lié à un ServiceAccount :

```
$ kubectl create token myapp-sa
```

Ce jeton est à utiliser par votre application pour s’authentifier auprès du kube-apiserver.

***

### Exemple pratique : Interroger l’API Kubernetes depuis un Pod

Une fois le jeton disponible dans le Pod, une application peut interroger l’API Kubernetes. Par exemple, depuis un conteneur, on peut exécuter :

```
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/pods
```

Cette commande liste les Pods du namespace courant, en utilisant le jeton du ServiceAccount monté automatiquement.

### Conseil sécurité et bonnes pratiques

- Par défaut, le ServiceAccount `default` est associé à chaque Pod s’il n’en spécifie pas un autre, mais ses permissions sont limitées.
- Il est recommandé de créer un ServiceAccount dédié à chaque application avec des permissions précises via les **Role** et **RoleBinding** (RBAC).
- Utilisez la **TokenRequestAPI** (via `kubectl create token` ou l’auto-montage des jetons avec durée limitée) plutôt que de créer manuellement des Secrets avec des jetons non expirants, pour améliorer la sécurité.
- Surveillez et limitez les permissions accordées au ServiceAccount pour suivre le principe du moindre privilège.
