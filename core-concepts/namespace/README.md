## Les Namespaces dans Kubernetes

Dans cette conférence, nous abordons les **Namespaces** dans Kubernetes à travers une analogie simple.

Imaginez deux garçons qui s’appellent Ramzi. Pour les différencier, on utilise leurs noms de famille : "Hannachi" et "Zoon".\
Ils viennent de maisons différentes, chacune avec ses propres membres.\
À l’intérieur d’une maison, les habitants s’appellent par leur prénom par exemple, le père appelle simplement "Ramzi".\
Mais pour s’adresser à Ramzi dans l’autre maison, il faut utiliser le nom complet. Une personne extérieure fait de même.

Chaque maison a ses propres règles et ressources on peut donc la considérer comme une unité isolée.

### Correspondance avec Kubernetes

Ces maisons correspondent aux **Namespaces** dans Kubernetes.

Jusqu’ici, nous avons créé des objets comme des **Pods**, **Deployments** ou **Services** dans un cluster. Par défaut, tout est créé dans le **default namespace**, qui est automatiquement généré à l’installation du cluster.

Kubernetes crée aussi d’autres namespaces pour ses composants internes, par exemple :
- **kube-system** : pour les services système tels que la solution réseau, DNS, etc.
- **kube-public** : pour des ressources accessibles à tous les utilisateurs.

### Utilisation des Namespaces

Pour un petit cluster ou un environnement d’apprentissage, il est acceptable de rester dans le namespace par défaut. Mais en production, il est recommandé d’utiliser plusieurs **Namespaces**.  
Par exemple, on peut isoler les environnements de **development** et **production** dans des namespaces distincts. Cela évite des modifications accidentelles entre environnements et permet d’appliquer des règles et quotas adaptés à chacun.

### Isolement et accès

Chaque **Namespace** dispose de ses propres politiques qui contrôlent les droits d’accès et d’utilisation des ressources. On peut aussi définir des quotas de ressources (CPU, mémoire, nombre de Pods, etc.) garantissant qu’un namespace ne dépasse pas certaines limites.

### Résolution DNS entre Namespaces

Dans un même namespace, les ressources communiquent simplement par leur nom. Par exemple, un **Pod** d’une application web peut joindre un **Service** de base de données simplement via `dbservice`.

Pour communiquer avec un service dans un namespace différent, il faut utiliser la forme complète DNS :
```
<service>.<namespace>.svc.cluster.local
```
Par exemple, pour joindre un service `dbservice` dans le namespace `dev` depuis un Pod dans le namespace `default`, on utilise :
```
dbservice.dev.svc.cluster.local
```
Kubernetes crée automatiquement l’entrée DNS correspondante.

### Gestion avec kubectl

- Pour lister les ressources dans le namespace par défaut :
```
$ kubectl get pods
```
- Pour lister dans un autre namespace, par exemple `kube-system` :
```
$ kubectl get pods -n kube-system
```
- Pour créer un Pod dans un namespace autre que le défaut, on peut utiliser l’option :
```
$ kubectl apply -f pod.yaml -n dev
```
- Ou inclure le namespace directement dans la définition YAML à la section `metadata.namespace`, ce qui garantit que la ressource sera créée dans le namespace voulu même sans option CLI.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: dev
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
```
### Création d’un Namespace

On peut créer un namespace avec un fichier YAML :
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```
Puis :
```
$ kubectl apply -f namespace-dev.yaml
```
Ou directement :
```
$ kubectl create namespace dev
```

### Changer le namespace par défaut pour kubectl

Pour ne plus spécifier le namespace avec `-n` à chaque commande, on peut définir le namespace par défaut dans le contexte actuel :
```
$ kubectl config set-context --current --namespace=dev
```
Après cela, les commandes kubectl utilisent le namespace `dev` par défaut.

Pour afficher le namespace actuellement actif :
```
$ kubectl config view --minify --output 'jsonpath={..namespace}'
```

### Visualiser les ressources dans tous les namespaces

Pour voir tous les Pods dans tous les namespaces :
```
$ kubectl get pods --all-namespaces
```
Cela permet de voir l’ensemble des ressources dans le cluster, quel que soit le namespace.

## ResourceQuota dans Kubernetes

Un **ResourceQuota** est un objet Kubernetes qui permet de **limiter et contrôler la consommation globale des ressources** au sein d’un **namespace** donné.

### À quoi sert un ResourceQuota ?

- Garantir une consommation **équilibrée et maîtrisée** des ressources entre plusieurs utilisateurs ou équipes partageant un cluster Kubernetes.
- Empêcher qu’une équipe, application ou service ne consomme toutes les ressources disponibles, ce qui pourrait affecter la stabilité ou la disponibilité du cluster.
- Optimiser l’utilisation globale des ressources CPU, mémoire, stockage, et objets Kubernetes (Pods, Services, PersistentVolumeClaims, etc.).
- Appliquer des restrictions sur le nombre de Pods, les demandes (`requests`) et limites (`limits`) de CPU/mémoire dans un namespace.

```yaml
# resource-quota.yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota-cpu-memoire
  namespace: dev
spec:
  hard:
    pods: "10"                   # Nombre maximum de Pods dans le namespace
    requests.cpu: "4"            # CPU total demandé maximum (4 cores)
    requests.memory: "8Gi"       # Mémoire totale demandée maximum
    limits.cpu: "6"              # Limite CPU maximale totale
    limits.memory: "12Gi"        # Limite mémoire maximale totale
```

### Comment l’affecter à un namespace ?
1. Créez le namespace s’il n'existe pas encore :
```
$ kubectl create namespace dev
```
2. Appliquez le ResourceQuota dans ce namespace :
```
$ kubectl apply -f resource-quota.yaml
```
3. Vérifiez l’état du quota appliqué :
```
$ kubectl get resourcequota -n dev
$ kubectl describe resourcequota my-quota-cpu-memoire -n dev
```

- Le quota limite la consommation globale des ressources dans le namespace `dev`.
- Kubernetes empêche la création de nouvelles ressources ou Pods si ces quotas sont dépassés.
- Vous pouvez ajuster les valeurs selon les besoins de chaque namespace et de vos équipes.

***

## Résumé concis

- Les **Namespaces** sont des partitions logiques d’un **Cluster** Kubernetes, assurant isolation et gestion des ressources.
- Kubernetes crée par défaut des namespaces `default`, `kube-system` et `kube-public`.
- En production, il est recommandé de créer des namespaces pour isoler environnements ou équipes.
- Les ressources dans un même namespace peuvent se référencer par leur nom simple, mais pour accéder à une ressource dans un autre namespace, il faut utiliser le nom DNS complet `<service>.<namespace>.svc.cluster.local`.
- La commande `kubectl` utilise par défaut le namespace `default`. On peut changer le namespace par défaut dans le contexte avec `kubectl config set-context --current --namespace=<name>`.
- Pour créer un namespace, soit un YAML soit la commande `kubectl create namespace <name>`.
- Les quotas de ressources et politiques d’accès peuvent être appliqués au niveau des namespaces.

***



