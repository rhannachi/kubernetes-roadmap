# Taints and Tolerations

Dans cette section, nous allons examiner la relation entre un Pod et un Node, et comprendre comment restreindre le placement des Pods sur certains Nodes.\
Pour cela, Kubernetes utilise les mécanismes de **taints** et **tolerations**, souvent perçus comme complexes au premier abord.

## Analogie introductive
Considérons une analogie simple : une personne représente un Node et un insecte représente un Pod.\
Si la personne applique un répulsif (un *taint*), la plupart des insectes ne pourront pas s’approcher, sauf ceux qui supportent ce répulsif (les Pods disposant d’une *toleration*). Ainsi :
- Les **taints** sont appliqués aux **Nodes**.
- Les **tolerations** sont définies dans les **Pods**.  
  Les deux concepts fonctionnent ensemble pour contrôler **quels Pods peuvent être acceptés par quels Nodes.**

## Fonctionnement général
Par défaut, Kubernetes scheduler distribue les Pods de manière équilibrée sur les Nodes disponibles, sans contraintes particulières.\
En ajoutant un *taint* à un Node, celui-ci refuse tout Pod qui ne possède pas de *toleration* correspondante.\
Cela permet de réserver un Node pour un usage précis.

Exemple :
- Un *taint* « app=blue:NoSchedule » est appliqué sur un Node spécifique.
- Tous les Pods sans *toleration* correspondante sont refusés.
- Un Pod disposant d’une *toleration* pour « app=blue » pourra être déployé sur ce Node.

## Configuration des taints sur un Node et des tolerations correspondantes sur un Pod

### 1. Taint avec effet **NoSchedule**

- Ajoute un taint sur le Node `<node-name>` avec la clé `"app"`, la valeur `"blue"` et l’effet `"NoSchedule"`.
- **Conséquence** : Aucun Pod sans *toleration* correspondante ne pourra être planifié sur ce Node.

```
$ kubectl taint nodes <node-name> app=blue:NoSchedule
```

- Exemple de manifest Pod avec *toleration* correspondante :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-noschedule
spec:
  containers:
  - name: app
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

***

### 2. Taint avec effet **PreferNoSchedule**

- Ajoute un taint sur le Node `<node-name>` avec la clé `"app"`, la valeur `"blue"` et l’effet `"PreferNoSchedule"`.
- **Conséquence** : Kubernetes évite de programmer des Pods non tolérants à ce taint sur ce Node, mais ce n’est pas strictement garanti.

```
$ kubectl taint nodes <node-name> app=blue:PreferNoSchedule
```

- Exemple de manifest Pod avec *toleration* correspondante :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-prefernoschedule
spec:
  containers:
  - name: app
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "PreferNoSchedule"
```

***

### 3. Taint avec effet **NoExecute**

- Ajoute un taint sur le Node `<node-name>` avec la clé `"app"`, la valeur `"blue"` et l’effet `"NoExecute"`.
- **Conséquence** :
    - Les nouveaux Pods non tolérants à ce taint ne seront pas planifiés sur ce Node.
    - Les Pods déjà en cours d’exécution sans *toleration* correspondante seront expulsés (éviction).

```
$ kubectl taint nodes <node-name> app=blue:NoExecute
```

- Exemple de manifest Pod avec *toleration* correspondante :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-noexecute
spec:
  containers:
  - name: app
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoExecute"
```

***

## Limites et bonnes pratiques
Il est important de noter que **taints et tolerations n’indiquent jamais explicitement à un Pod sur quel Node aller**.\
Ils définissent uniquement quels Nodes acceptent ou refusent certains Pods. Pour réellement cibler un Pod sur un Node spécifique, il faut utiliser le mécanisme de **Node affinity** (abordé séparément).

Par ailleurs, lors de l’installation d’un Cluster Kubernetes, les **master Nodes** ont automatiquement un *taint* appliqué par défaut afin d’empêcher le scheduling de Pods applicatifs sur ces Nodes.\
Il est possible de supprimer ou modifier ce taint, mais la bonne pratique reste de ne pas exécuter de charges applicatives sur les *control plane Nodes*.
```
$ kubectl describe node kubermaster | grep Taint
```

***

## Résumé concis

- Les **taints** sont appliqués sur les Nodes pour restreindre quels Pods peuvent y être planifiés.
- Les **tolerations** sont appliquées aux Pods pour leur permettre d’être acceptés par certains Nodes.
- Un *taint* peut avoir trois effets : **NoSchedule**, **PreferNoSchedule**, **NoExecute**.
- Exemple : `kubectl taint nodes node1 app=blue:NoSchedule` empêche tous les Pods de s’exécuter sur `node1`, sauf ceux dotés d’une *toleration* correspondante.
- Ces mécanismes ne garantissent pas le placement d’un Pod sur un Node précis. Pour cela, il faut utiliser la **Node affinity**.
- Par défaut, les Nodes de type master sont protégés par un *taint* automatique empêchant le scheduling de Pods applicatifs.
