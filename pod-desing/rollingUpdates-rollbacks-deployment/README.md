# Déploiements : Rollouts, Updates et Rollbacks

Dans ce cours, nous allons voir comment gérer les **updates** et **rollbacks** dans un **Deployment** Kubernetes.\
Pour comprendre clairement ces notions, nous allons explorer les mécanismes de **rollouts**, les différentes stratégies de mise à jour, et les commandes essentielles pour gérer vos applications.

### Rollouts et versions d’un Deployment

Quand vous créez un **Deployment**, cela déclenche automatiquement un **rollout**, qui correspond à une nouvelle révision.
- Le premier déploiement correspond, par exemple, à la **Revision 1**.
- Si vous mettez à jour votre application (changement de version d’image, modification du nombre de replicas, etc.), un nouveau **rollout** est lancé et une nouvelle révision est créée (**Revision 2**, **Revision 3**, etc.).

Cela permet de suivre l’historique des modifications et de revenir à une version précédente en cas de problème.

**Commandes utiles :**
```
# Vérifier le statut d’un rollout
$ kubectl rollout status deployment/my-app

# Consulter l’historique des rollouts
$ kubectl rollout history deployment/my-app
```

***

### Stratégies de mise à jour d’un Deployment

Il existe deux stratégies principales définies par le champ `strategy` dans un **Deployment**.

#### 1. Recreate strategy
- Kubernetes détruit **toutes les instances (Pods)** de l’ancienne version puis crée les nouvelles.
- Problème : pendant la transition, l’application est totalement indisponible.

Exemple (définition d’un Deployment avec `Recreate`):
```yaml
spec:
  strategy:
    type: Recreate
```

#### 2. RollingUpdate strategy (par défaut)
- Kubernetes remplace progressivement les Pods : il détruit un ancien Pod, puis crée un nouveau Pod, et ainsi de suite.
- L’application reste disponible tout au long du processus.
- Stratégie **par défaut** si aucune n’est précisée.

Exemple (définition d’un Deployment avec `RollingUpdate`):
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

***

### Mettre à jour un Deployment

Pour effectuer une mise à jour, deux méthodes existent :

#### 1. Modifier le fichier manifeste et l’appliquer :
```
# Éditer le fichier deployment.yaml
$ kubectl apply -f deployment.yaml
```
Cela déclenche un nouveau **rollout**.

#### 2. Utiliser `kubectl set image`
```
$ kubectl set image deployment/my-app my-app-container=nginx:1.27
```

⚠️ Attention : cette commande modifie l’objet en direct sans mettre à jour votre fichier YAML, ce qui peut provoquer une désynchronisation entre le manifeste et l’état réel du cluster.

***

### Vérifier un Deployment

Vous pouvez obtenir des détails et comprendre ce qu’il se passe en regardant la description du Deployment :

```
$ kubectl describe deployment my-app
```

Avec la stratégie `Recreate`, vous verrez que l’ancien ReplicaSet est d’abord réduit à 0 Pods puis le nouveau est créé.  
Avec `RollingUpdate`, vous verrez des Pods migrer progressivement entre l’ancien et le nouveau ReplicaSet.

***

### Comment Kubernetes gère un Deployment sous le capot

Lorsqu’un **Deployment** est créé pour 5 replicas :
- Kubernetes génère un **ReplicaSet**, qui crée lui-même les **Pods**.
- Lors d’une mise à jour (avec RollingUpdate), un **nouveau ReplicaSet** est automatiquement généré.
- Pendant la transition :
    - L’ancien ReplicaSet est réduit progressivement.
    - Le nouveau ReplicaSet monte en charge progressivement.

Vous pouvez observer cela avec :
```
$ kubectl get rs
```

Avant la mise à jour :
- Ancien ReplicaSet : 5 Pods
- Nouveau ReplicaSet : 0 Pods

Pendant/Après la mise à jour :
- Ancien ReplicaSet : 0 Pods
- Nouveau ReplicaSet : 5 Pods

***

### Rollback d’un Deployment

Si la nouvelle version présente un problème, Kubernetes permet un retour rapide à la révision précédente.

**Commande de rollback :**
```
$ kubectl rollout undo deployment/my-app
```

Conséquence :
- Les Pods du nouveau ReplicaSet sont supprimés.
- Les Pods de l’ancien ReplicaSet sont relancés.

Vérification avec :
```
$ kubectl get rs
```

Avant rollback :
- Ancien ReplicaSet : 0 Pods
- Nouveau ReplicaSet : 5 Pods

Après rollback :
- Ancien ReplicaSet : 5 Pods
- Nouveau ReplicaSet : 0 Pods

***

### Résumé des commandes principales

```
# Créer un Deployment
$ kubectl create -f deployment.yaml

# Lister les Deployments
$ kubectl get deployments

# Mettre à jour un Deployment via fichier
$ kubectl apply -f deployment.yaml

# Mettre à jour l’image d’un conteneur
$ kubectl set image deployment/my-app my-app-container=nginx:1.27

# Consulter l’historique des rollouts
$ kubectl rollout history deployment/my-app

# Suivre le statut d’un rollout
$ kubectl rollout status deployment/my-app

# Faire un rollback
$ kubectl rollout undo deployment/my-app
```

***

## Résumé concis (essentiel)

- Un **Deployment** génère des **rollouts** et des révisions (Revision 1, Revision 2, etc.).
- Deux stratégies de mise à jour :
    - **Recreate** : tous les Pods sont supprimés, puis recréés (temps d’indisponibilité).
    - **RollingUpdate** (par défaut) : remplacement progressif des Pods (zéro downtime).
- Pour mettre à jour : soit modifier le manifeste (`kubectl apply`), soit changer l’image (`kubectl set image`).
- Kubernetes crée automatiquement un nouveau **ReplicaSet** lors d’une mise à jour.
- En cas d’échec, on peut revenir en arrière avec `kubectl rollout undo`.

***

### Afficher toutes les versions (révisions)

Pour lister les révisions disponibles d’un Deployment :
```
$ kubectl rollout history deployment/my-webapp
```
Cela affiche la liste des révisions disponibles avec leur numéro (`REVISION`).  
Pour obtenir les détails d’une révision particulière :
```
$ kubectl rollout history deployment/my-webapp --revision=4
```
Cela montrera le contenu du Pod template et les modifications apportées lors de cette révision.

***

### Faire un "undo" vers une version spécifique

Par défaut, un rollback revient à la dernière version stable (l’avant-dernière révision).  
Pour choisir une révision précise, utilisez l’option `--to-revision` :

```
$ kubectl rollout undo deployment/my-webapp --to-revision=3
```
Cette commande restaurera l’état du Deployment à la révision numéro 3 (ou le numéro que tu veux).

***

### Remarques

- Le champ `CHANGE-CAUSE` peut être vide si tu n’as pas utilisé l’annotation `kubernetes.io/change-cause` lors de tes mises à jour.
- N’oublie pas que le nombre de révisions conservées dépend du paramètre `revisionHistoryLimit` du Deployment (par défaut : 10).
- Les commandes sont valables pour tous les objets compatibles rollout (Deployment, DaemonSet, StatefulSet).

***

**Exemples pour manipuler l’historique et faire un rollback ciblé :**
```
# Lister toutes les révisions
$ kubectl rollout history deployment/my-webapp

# Afficher une révision particulière
$ kubectl rollout history deployment/my-webapp --revision=3

# Rollback vers une révision spécifique
$ kubectl rollout undo deployment/my-webapp --to-revision=3
```
