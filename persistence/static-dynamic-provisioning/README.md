## Provisionnement statique et dynamique des volumes persistants dans Kubernetes

Dans cet exemple, nous créons un **PersistentVolumeClaim (PVC)** basé sur un disque persistant Google Cloud.

Le problème de la méthode statique est qu’avant de créer le **PersistentVolume (PV)**, il faut préalablement créer manuellement le disque dans Google Cloud.\
Chaque fois qu'une application a besoin de stockage, il faut d'abord provisionner manuellement ce disque, puis créer un fichier de définition de PV avec le même nom que celui du disque créé.\

=> C’est ce qu’on appelle le **provisionnement statique**.

### Exemple de provisionnement statique
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gce-pd-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  gcePersistentDisk:
    pdName: my-disk
    fsType: ext4
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gce-pd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: gce-pd-pv
```

Ici, le volume est créé manuellement dans Google Cloud avec le nom `my-disk`. Le PV référence ce disque par `pdName`. Ce fonctionnement est rigide, difficile à maintenir et manuel.

***

### Provisionnement dynamique avec StorageClass

Le **provisionnement dynamique** permet à Kubernetes de créer automatiquement un PV et le disque sous-jacent quand un PVC est créé, sans intervention manuelle préalable.

Pour cela, on définit une ressource **StorageClass** qui utilise un *provisioner* adapté, par exemple `kubernetes.io/gce-pd` pour Google Cloud.\
Le StorageClass spécifie aussi des paramètres additionnels comme le type de disque (`standard` ou `ssd`) ou la réplication (simple ou régionale).

Exemple de StorageClass pour Google Cloud Persistent Disk :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: none
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Le PVC référence alors ce StorageClass dans sa définition et Kubernetes gère la création automatique du volume et sa liaison au PVC.

Exemple de PVC utilisant StorageClass :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

***

### Avantages et possibilité de classes de stockage

Avec les StorageClasses, on peut créer plusieurs niveaux de service :
- **silver** : disques standards
- **gold** : disques SSD
- **platinum** : SSD avec réplication régionale

Cela permet aux développeurs de choisir rapidement la qualité de stockage désirée sans se soucier des détails techniques.

***

## Résumé concis

- Le **provisionnement statique** demande de créer manuellement le disque et le PV avant le PVC, ce qui est fastidieux et peu évolutif.
- Le **provisionnement dynamique** via une **StorageClass** automatise la création des volumes, simplifiant la gestion.
- Le PVC fait référence à un StorageClass qui utilise un *provisioner* spécifique (ex. Google Cloud Persistent Disk), et Kubernetes crée automatiquement le PV et le volume sur le cloud.
- Il est possible de définir plusieurs StorageClasses pour offrir différentes **classes de stockage** (standard, SSD, répliqué, etc.) adaptées aux besoins.
- Cette approche améliore la flexibilité et la rapidité de déploiement des applications nécessitant du stockage persistant.

