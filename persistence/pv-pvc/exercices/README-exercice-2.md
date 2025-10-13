## 1. Définition et analyse de PV et PVC

Commençons par la création d'un **PersistentVolume** (PV) et d'une **PersistentVolumeClaim**(PVC):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/log
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  volumeName: pv-log
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
```

## 2. Observation avec kubectl

Après avoir appliqué ces fichiers, vérifions leur état :

```
$ kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-log   100Mi      RWX            Retain           Available                          <unset>                          9m56s

$ kubectl get pvc
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
claim-log-1   Pending   pv-log   0                                        <unset>                 7m15s
```

On constate que le **PVC** n’a pas été associé au **PV** : son statut est **Pending**, alors que le PV reste **Available**.

***

## 3. Explication

Pour qu’un PVC soit lié à un PV, plusieurs critères doivent être respectés :
- La capacité demandée par le PVC doit être inférieure ou égale à celle du PV.
- Les **modes d'accès** doivent être compatibles.
- Si une propriété `StorageClass` existe, elle doit aussi correspondre.

Dans notre exemple :
- **PV** offre le mode `ReadWriteMany` (RWX).
- **PVC** demande le mode `ReadWriteOnce` (RWO).

**Important** : Le mode demandé par le PVC doit être inclus dans les modes d’accès du PV. Ici, `ReadWriteOnce` n’est pas compatible avec un PV configuré uniquement en `ReadWriteMany`.  
Kubernetes refuse donc de lier le PVC au PV.

***

## 4. Correction : Associer correctement le PVC au PV

Pour résoudre le problème de compatibilité, adaptons le mode d’accès du PVC pour qu’il corresponde à celui du PV :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  volumeName: pv-log
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Mi
```

Après correction et re-déploiement :

```
kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-log   100Mi      RWX            Retain           Bound    default/claim-log-1                  <unset>                          13m

kubectl get pvc
NAME          STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
claim-log-1   Bound    pv-log   100Mi      RWX                           <unset>                 13s
```

- Le champ CLAIM est maintenant rempli dans la liste des PV, ce qui indique que le PV est effectivement lié à un PVC. 
- Le statut est passé à Bound pour les deux ressources, confirmant une association réussie. 
- La capacité affichée pour le PVC est celle du PV (100Mi), même si la demande était de 50Mi, car la capacité du PV est réservée entièrement au PVC.

***

## 5. Points importants à retenir

- **Compatibilité des modes d’accès** : toujours aligner les `accessModes` dans le PVC sur ceux du PV.
- **Capacité du volume** : le PVC peut demander moins que la capacité du PV, mais si le PV est plus grand, toute sa capacité peut être vue comme réservée pour ce PVC.
- **STATUS "Bound"** : signifie que le claim (PVC) utilise un volume (PV) dans le cluster.
- **ATTENTION** : Dans Kubernetes, une fois le binding effectué, **la capacité affichée pour le PVC correspond à celle du PV**, même si la demande était inférieure.

***

## 6. Exemple complet d’utilisation dans un Pod

Pour exploiter ce PVC dans un Pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-reader
spec:
  containers:
  - name: log-reader
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: "/log"
      name: log-storage
  volumes:
  - name: log-storage
    persistentVolumeClaim:
      claimName: claim-log-1
```
Ce Pod pourra lire/écrire dans `/log` grâce au PersistentVolume alloué via le PVC.

***

## Résumé concis

- Le PVC ne s'associe pas au PV si leurs modes d'accès ne sont pas compatibles.
- Corriger le PVC pour qu'il demande `ReadWriteMany` permet d'effectuer le binding.
- Une fois le binding réalisé, la totalité de la capacité du PV est réservée au PVC, même si la demande était inférieure.
- Un Pod peut maintenant utiliser ce stockage pour ses besoins d'accès aux données persistantes.
