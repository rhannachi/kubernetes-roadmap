# NodeSelector

## Contraindre un Pod à s’exécuter sur un Node spécifique avec nodeSelector

Nous allons voir comment orienter l’exécution d’un **Pod** vers un **Node** précis dans un **Cluster** Kubernetes, en utilisant les **labels** et le champ **nodeSelector**.

### 1. Situation initiale

Imaginons un **Cluster** composé de trois **Nodes** :
- Deux petits **Nodes** avec peu de ressources.
- Un **Node** plus grand et puissant.

Certains workloads (comme un job de traitement de données) nécessitent davantage de ressources.\
Par défaut, le **scheduler** Kubernetes peut placer les **Pods** sur n’importe quel **Node**, y compris sur ceux qui ne sont pas assez puissants.

Notre objectif est donc que ce workload ne soit exécuté **que** sur le **Node** le plus puissant.

***

### 2. Étape 1 — Ajouter un label au Node

Avant de pouvoir utiliser un **nodeSelector**, il faut d’abord **étiqueter** le **Node** cible avec un label.  
Par exemple, donnons au **Node** le label `size=large` :

```
$ kubectl label nodes <nom-du-node> size=large
```

Vous pouvez vérifier que le label a bien été appliqué avec la commande :

```
$ kubectl get nodes --show-labels
```

***

### 3. Étape 2 — Définir un Pod avec nodeSelector

Ensuite, nous allons créer un manifeste YAML pour notre **Pod**, en ajoutant la propriété **nodeSelector** dans la section `spec`.

Exemple de fichier `pod-dataproc.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dataproc-pod
spec:
  containers:
  - name: data-processor
    image: my-dataproc-image:latest
  nodeSelector:
    size: large
```

Ici, le **Pod** demandera à être planifié uniquement sur un **Node** ayant le label `size=large`.

***

### 4. Étape 3 — Déploiement du Pod

Déployez le **Pod** avec la commande :

```
kubectl apply -f pod-dataproc.yaml
```

Puis vérifiez sur quel **Node** il a été placé :

```
kubectl get pod dataproc-pod -o wide
```

Le résultat doit montrer que le **Pod** s’exécute sur le **Node** étiqueté `size=large`.

***

### 5. Limites de nodeSelector

L’approche avec **nodeSelector** est simple et fonctionne pour des besoins basiques.  
Cependant, elle ne permet pas de définir des règles plus complexes comme :
- placer un **Pod** sur un **Node** de type *large* ou *medium*,
- éviter tous les **Nodes** de type *small*.

Pour ce type de besoin avancé, Kubernetes propose les mécanismes **nodeAffinity** et **antiAffinity**, qui offrent bien plus de flexibilité.

***
***

# NodeAffinity

## Utiliser nodeAffinity pour un placement avancé des Pods

### 6. Pourquoi utiliser nodeAffinity ?

Contrairement à **nodeSelector**, qui ne permet de définir qu’une seule clé-valeur simple, **nodeAffinity** offre la possibilité :
- de préciser des règles "OU" et "ET",
- de sélectionner plusieurs labels,
- d’exclure des **Nodes**,
- d’utiliser des opérateurs plus riches comme `In`, `NotIn`, `Exists`, etc.

***

### 7. Exemple pratique : Placer un Pod sur un Node de taille *large* ou *medium*

Supposons que vos **Nodes** disposent des labels `size=large`, `size=medium` ou `size=small`.\
Vous voulez que votre **Pod** soit planifié uniquement sur un **Node** `large` **ou** `medium` (excluant les petits).

Voici un exemple de manifeste YAML avec **nodeAffinity** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dataproc-pod-affinity
spec:
  containers:
  - name: data-processor
    image: my-dataproc-image:latest
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - large
            - medium
```

***

### 8. Explications sur ce manifeste

- La section `affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution` définit une règle obligatoire lors de la planification (**scheduling**) du **Pod**.
- `nodeSelectorTerms` contient une liste de conditions alternatives (OU).
- Ici, on demande que la clé `size` du label du **Node** soit soit `large`, soit `medium`.

***

### 9. Déploiement et vérification

Déployez ce **Pod** avec la commande :

```
$ kubectl apply -f pod-dataproc-affinity.yaml
```

Puis vérifiez son placement avec :

```
$ kubectl get pod dataproc-pod-affinity -o wide
```

Le **Pod** sera uniquement planifié sur un **Node** qui correspond à l’un des labels spécifiés (`large` ou `medium`).

***

### 10. Résumé

| Fonctionnalité       | nodeSelector                      | nodeAffinity                            |
|---------------------|---------------------------------|---------------------------------------|
| Complexité des règles| Simple (clé = valeur)            | Avancée (opérateurs multiples, OU/ET)  |
| Possibilité d’exclusion | Non                           | Oui                                   |
| Types d’opérateurs   | =                               | In, NotIn, Exists, etc.                |
| Flexibilité          | Limitée                         | Très flexible                         |

