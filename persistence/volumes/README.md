## Volumes en Kubernetes

En **Kubernetes**, les *Pods* suivent la même logique que les containers Docker, ils sont temporaires.  
Un Pod créé pour traiter des données et supprimé ensuite entraîne par défaut la perte de ces données.

Pour conserver ces données, on attache un **Volume** au Pod.  
Ainsi, ces données sont stockées dans le volume et restent disponibles même si le Pod est supprimé.

***

### Exemple simple : "HostPath" volume

Voici une démonstration sur un **Single Node Cluster** :
1. Création d’un Pod qui génère un nombre aléatoire entre 1 et 100.
2. Ce nombre est écrit dans un fichier situé dans `/opt` à l’intérieur du container.
3. Sans volume, la suppression du Pod entraîne la disparition du fichier.

#### Configuration avec Volume
Nous allons créer un Volume et utiliser **HostPath** pour stocker les données sur le Node.

- **Répertoire sur le Node** : `/data`
- **Montage dans le Pod** : `/opt`
- Les fichiers créés dans `/opt` à l’intérieur du container seront en réalité stockés dans `/data` sur le Node.

**Manifest YAML :**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-pod
spec:
  containers:
  - name: number-generator
    image: busybox
    command: ["sh", "-c", "echo $((RANDOM % 100 + 1)) > /opt/random.txt && sleep 30"]
    volumeMounts:
    - name: data-volume
      mountPath: /opt
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: DirectoryOrCreate
```

Après suppression du Pod, le fichier `random.txt` reste dans `/data` sur le Node.

***

### Limites de HostPath

- **Utilisation recommandée uniquement sur Single Node**.
- Dans un cluster multi-nodes, chaque Node possède son propre répertoire `/data`, ce qui entraîne des incohérences.
- Solution : utiliser un **storage partagé ou distribué**, par exemple **NFS**, **GlusterFS**, **CephFS**, ou des services cloud.

***

### Autres options de stockage en Kubernetes

Kubernetes supporte plusieurs types de stockage :

- **NFS** : système de fichiers réseau
- **GlusterFS**, **CephFS** : stockage distribué
- **AWS EBS** : disque persistant sur Amazon Web Services
- **Azure Disk** et **Azure File**
- **Google Persistent Disk**
- **Fibre Channel**, **ScaleIO**, etc.

**Exemple : Volume AWS EBS**
```yaml
volumes:
- name: ebs-volume
  awsElasticBlockStore:
    volumeID: vol-0abcdef1234567890
    fsType: ext4
```
Les données seront stockées sur un volume EBS et resteront disponibles même si le Pod s’exécute sur différents Nodes (avec montage approprié).

***

## Résumé concis

- Les **containers Docker** et les **Pods Kubernetes** sont *transients* (données perdues à la suppression).
- Les **Volumes** permettent de conserver les données après la suppression d’un Pod ou container.
- **HostPath** stocke les données dans un répertoire du Node, adapté pour un **Single Node Cluster**, pas pour un cluster multi-nodes.
- Pour un stockage durable et partagé dans un **Cluster**, utiliser des solutions comme **NFS**, **CephFS**, ou **AWS EBS**.
- En Kubernetes, le Volume est déclaré au niveau du Pod et monté via `volumeMounts`.


