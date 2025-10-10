## Volume persistants et claims dans Kubernetes
En Kubernetes, la gestion du stockage peut être centralisée grâce aux **Persistent Volumes** (PV) et **Persistent Volume Claims** (PVC).

Dans un petit environnement, on peut définir le volume directement dans le fichier YAML du Pod.\
Mais dans un environnement avec de nombreux utilisateurs et Pods, cette approche oblige chaque utilisateur à configurer le stockage à chaque déploiement, ce qui devient lourd et difficile à maintenir.

Les **Persistent Volumes** permettent à un administrateur de créer un **pool de stockage** à l’échelle du Cluster.\
Les utilisateurs peuvent ensuite demander une portion de ce pool via un **Persistent Volume Claim**.

***

## Exemple : création d’un Persistent Volume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
```
**Étapes :**
1. Définir `apiVersion` et `kind`.
2. Nommer le PV (`metadata.name`).
3. Spécifier les **Access Modes** :
    - `ReadOnlyMany` (lecture seule depuis plusieurs Nodes)
    - `ReadWriteOnce` (lecture/écriture depuis un seul Node)
    - `ReadWriteMany` (lecture/écriture depuis plusieurs Nodes)
4. Définir la capacité (`storage: 1Gi`).
5. Spécifier le type de volume (ici `hostPath`, uniquement en test, jamais en production).

**Commande pour création :**
```
$ kubectl apply -f pv.yaml
$ kubectl get pv
```

***

## Exemple : création d’un Persistent Volume Claim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```
**Étapes :**
1. Nommer le PVC (`my-claim`).
2. Définir `accessModes` correspondant au PV.
3. Spécifier la capacité demandée (`500Mi`).
4. Créer le PVC et vérifier :
```
$ kubectl apply -f pvc.yaml
$ kubectl get pvc
```
Si aucun PV correspondant n'est disponible, le PVC reste **Pending**. Dès qu’un PV adéquat est disponible, Kubernetes lie automatiquement le PV au PVC.

***
## Liens et règles de binding PV ↔ PVC
- Kubernetes associe un PVC à un PV si la capacité, les modes d’accès et les autres propriétés (ex. StorageClass) correspondent.
- Un PVC ne peut être lié qu'à **un seul PV** à la fois.
- Un PVC plus petit peut être lié à un PV plus grand si aucun autre PV ne correspond.

### Précision sur la liaison 1-à-1 entre PV et PVC
En Kubernetes, **il n'est normalement pas possible de lier plusieurs PVC à un seul PV**, même si le PV a suffisamment d’espace disponible.\
Cette liaison est exclusive pour éviter les conflits d’accès concurrents et assurer l’intégrité des données.\
Cependant, un même PV, si configuré avec un mode d’accès comme `ReadWriteMany`, peut être monté par plusieurs Pods via le même PVC, mais la relation PV-PVC reste 1-à-1.

### Liaison explicite via `volumeName`
À part le mécanisme automatique de liaison, il existe un moyen d’associer explicitement un PVC à un PV précis grâce au champ `volumeName` dans la définition du PVC :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
  volumeName: host-pv  # Liaison explicite au PV nommé "host-pv"
```
Cette approche force Kubernetes à lier ce PVC au PV nommé `host-pv`, évitant la recherche automatique d’un PV compatible.\
Elle est utile quand on souhaite réserver un PV précis à un PVC spécifique, par exemple pour des volumes préexistants.

**Bonnes pratiques et recommandations :**
- Le binding automatique sans spécifier `volumeName` est généralement préféré pour préserver la flexibilité et l’indépendance entre PV et PVC.
- La liaison explicite par `volumeName` réduit cette flexibilité et doit être utilisée uniquement dans des cas spécifiques (stockage statique préalloué).
- Pour une gestion moderne et scalable, il est recommandé d’utiliser le **provisionnement dynamique** via **StorageClass** et des **CSI drivers**. Cela évite la nécessité de créer manuellement PV et d’importer des associations explicites.

***
## Suppression et politiques de recyclage
Lors de la suppression d’un PVC :
- **Retain** (défaut) : PV conservé, non réutilisable tant que l’administrateur ne le supprime pas.
- **Delete** : PV supprimé automatiquement.
- **Recycle** (déprécié) : ancien procédé de nettoyage (ex. `rm -rf *` via un Pod recycler) non sûr et limité.

> Aujourd’hui, Kubernetes recommande le **dynamic provisioning** via **StorageClass** et **CSI drivers** pour gérer automatiquement l’approvisionnement et la suppression sécurisée.

***
**Résumé rapide :**  
PV = stockage configuré par l’administrateur au niveau du Cluster.  
PVC = demande de stockage par un utilisateur/Pod.  
Binding automatique selon contraintes (capacité, access mode, StorageClass).  
Utilisation recommandée : dynamic provisioning.

### Résumé concis
- **Persistent Volume (PV)** : ressource de stockage configurée au niveau **Cluster** par un administrateur.
- **Persistent Volume Claim (PVC)** : requête faite par un utilisateur pour obtenir du stockage à partir du pool de PV.
- Binding automatique basé sur capacité, modes d’accès et StorageClass.
- Politiques de recyclage : Retain, Delete (Recycle obsolète).
- Bonne pratique : utiliser **StorageClass** + **CSI drivers** pour provisioning dynamique.
