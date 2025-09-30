# Déploiements Kubernetes : Rollouts, Updates et Rollbacks

Ce cours explique comment Kubernetes gère les mises à jour et les retours en arrière (*rollbacks*) d’un **Deployment**.  
Nous allons voir :
- ce qu’est un **rollout** et comment le suivre,
- les **stratégies de mise à jour** possibles (avec ou sans indisponibilité),
- comment **mettre à jour** une application,
- comment **documenter les changements avec CHANGE-CAUSE**,
- et enfin comment **revenir en arrière** en cas de problème.

***

### Qu’est-ce qu’un Rollout ?

Un **rollout** est le processus par lequel Kubernetes déploie une nouvelle version de votre application.  
Chaque rollout crée une **nouvelle révision** du Deployment.

- Exemple :
  - Création initiale du Deployment → **Revision 1**
  - Mise à jour de l’image, ajout d’un replica, modification d’un port… → **Revision 2**
  - Nouvelle mise à jour → **Revision 3**, etc.

Cela permet de garder un historique des versions et, si besoin, de revenir en arrière vers une version stable.

**Commandes pratiques :**
```
# Vérifier le statut du rollout en cours
$ kubectl rollout status deployment/my-app

# Voir l’historique des révisions
$ kubectl rollout history deployment/my-app
```

***

### Stratégies de mise à jour

Le comportement d’un Deployment lors d’un rollout est défini par le champ `strategy`.  
Il existe deux stratégies principales :

#### 1. Recreate
- Tous les anciens Pods sont supprimés avant de lancer les nouveaux.
- L’application est donc **indisponible** pendant la transition.
- Utile uniquement si vos Pods ne peuvent pas tourner en parallèle (par exemple : base de données non clusterisée).

Exemple :
```yaml
spec:
  strategy:
    type: Recreate
```

#### 2. RollingUpdate (par défaut)
- Les Pods sont remplacés **progressivement**.
- Kubernetes supprime un ancien Pod, puis crée un nouveau Pod, et répète jusqu’à migration complète.
- L’application reste disponible, avec éventuellement de petites variations de capacité.

Exemple :
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # maximum 1 Pod en moins
      maxSurge: 1         # maximum 1 Pod en plus
```

👉 Retenez que **RollingUpdate est le choix par défaut** et assure une continuité de service.

***

### Mettre à jour un Deployment

Il existe deux manières de mettre à jour un Deployment.

#### 1. Via un manifeste YAML
C’est la bonne pratique si vous gérez votre infra comme du code (GitOps, CI/CD).
```
$ kubectl apply -f deployment.yaml
```

#### 2. Via la ligne de commande
Rapide mais moins traçable, car le fichier YAML n’est pas mis à jour.
```
$ kubectl set image deployment/my-app my-container=nginx:1.27
```

***

### Documenter les changements avec CHANGE-CAUSE

Quand vous affichez l’historique d’un Deployment avec :

```
$ kubectl rollout history deployment/my-app
```

vous verrez une colonne **CHANGE-CAUSE**.
- Si elle est vide, c’est que vous n’avez pas fourni de message lors de la mise à jour.
- Pour la renseigner, il existe deux méthodes :

#### 1. Avec l’annotation `kubernetes.io/change-cause`
Ajoutez-la dans votre manifeste YAML :
```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Upgrade vers nginx:1.27"
```
Puis appliquez le fichier :
```
$ kubectl apply -f deployment.yaml
```

#### 2. Via la ligne de commande
```
$ kubectl annotate deployment my-app kubernetes.io/change-cause="Upgrade vers nginx:1.27"
```

Exemple complet d’historique :

```
$ kubectl rollout history deployment/my-app
deployment.apps/my-app 
REVISION  CHANGE-CAUSE
1         Création initiale
2         Upgrade vers nginx:1.27
```

👉 Cette pratique est fortement conseillée pour garder une **traçabilité claire** des déploiements effectués par l’équipe.

***

### Observer le comportement sous le capot

Un Deployment utilise des **ReplicaSets**, qui eux-mêmes gèrent les Pods.

- Quand vous créez un Deployment avec 5 replicas : → 1 ReplicaSet de 5 Pods.
- Quand vous mettez à jour : Kubernetes crée un **nouveau ReplicaSet** pour la nouvelle version.
- Pendant un rolling update :
  - ancien ReplicaSet ↓ progressivement
  - nouveau ReplicaSet ↑ progressivement

```
$ kubectl get rs
```

Vous verrez ainsi deux ReplicaSets : un qui diminue en taille, et un autre qui monte en charge.

***

### Faire un rollback

Si une nouvelle version pose problème, vous pouvez restaurer une version stable.

**Rollback simple (vers la version précédente) :**
```
$ kubectl rollout undo deployment/my-app
```

**Rollback vers une révision spécifique :**
```
$ kubectl rollout undo deployment/my-webapp --to-revision=3
```

***

### Explorer l’historique

Afficher toutes les révisions :
```
$ kubectl rollout history deployment/my-webapp
```

Voir le détail d’une révision (contenu du template Pod généré) :
```
$ kubectl rollout history deployment/my-webapp --revision=4
```

***

### Résumé des commandes essentielles

```
# Créer un Deployment
kubectl create -f deployment.yaml

# Lister les Deployments
kubectl get deployments

# Mettre à jour un Deployment (YAML)
kubectl apply -f deployment.yaml

# Mettre à jour une image directement
kubectl set image deployment/my-app my-container=nginx:1.27

# Suivre un rollout
kubectl rollout status deployment/my-app

# Consulter l’historique des rollouts
kubectl rollout history deployment/my-app

# Annoter un déploiement avec une cause de changement
kubectl annotate deployment my-app kubernetes.io/change-cause="Upgrade nginx:1.27"

# Faire un rollback
kubectl rollout undo deployment/my-app

# Rollback vers une version spécifique
kubectl rollout undo deployment/my-app --to-revision=2
```

***

### Points clés à retenir

- Un **Deployment** gère les mises à jour via des **rollouts** (Révision 1, 2, 3…).
- Stratégies disponibles :
  - **Recreate** → downtime garanti.
  - **RollingUpdate** (par défaut) → mise à jour progressive sans interruption.
- Kubernetes gère automatiquement les **ReplicaSets** derrière chaque version.
- Vous pouvez suivre le déploiement et **revenir à une version précédente** en cas de problème.
- Ajoutez toujours un **CHANGE-CAUSE** pour documenter vos mises à jour et garder une traçabilité claire.
