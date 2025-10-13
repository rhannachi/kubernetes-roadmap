## Objectif

1. Créer un **PersistentVolume (PV)** et un **PersistentVolumeClaim (PVC)**.
2. Déployer une **application (Pod)** qui écrit régulièrement dans `log.txt`.
3. Vérifier que les logs persistent même après suppression du Pod.

---

## Étape 1 — Créer le PersistentVolume (PV)

Fichier : **pv.yaml**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""   # Désactive le StorageClass pour ce PV statique
  hostPath:
    path: /data/pv
```

Tu dois créer le répertoire manuellement si tu utilises `hostPath`
**Commandes :**

```
$ minikube ssh
$ sudo mkdir -p /data/pv && sudo chmod 777 /data/pv
$ exit

$ kubectl apply -f pv.yaml
$ kubectl get pv
```

Tu devrais voir :

```
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
local-pv    1Gi        RWO            Retain           Available           manual         5s
```

---

## Étape 2 — Créer le PersistentVolumeClaim (PVC)

Fichier : **pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  volumeName: local-pv   # Lie ce PVC explicitement au PV local-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: ""   # Désactive le StorageClass du PVC pour correspondre au PV
```

**Commandes :**

```
$ kubectl apply -f pvc.yaml
$ kubectl get pvc
```

Résultat attendu :

```
NAME         STATUS   VOLUME     CAPACITY   ACCESS MODES   AGE
local-pvc    Bound    local-pv   1Gi        RWO            5s
```

---

## Étape 3 — Créer un Pod qui écrit dans un fichier log.txt

Fichier : **log-writer.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-writer
spec:
  containers:
    - name: logger
      image: busybox
      command: [ "sh", "-c" ]
      args:
        - |
          echo "Starting log writer..."; 
          while true; do 
            echo "$(date) - Application still running" >> /mnt/data/log.txt; 
            sleep 5; 
          done
      volumeMounts:
        - mountPath: "/mnt/data"
          name: my-storage
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: local-pvc
```

**Commandes :**

```
$ kubectl apply -f log-writer.yaml
$ kubectl get pods
```

Tu devrais voir :

```
NAME          READY   STATUS    RESTARTS   AGE
log-writer    1/1     Running   0          5s
```

---

## Étape 4 — Vérifier le contenu du fichier `log.txt`

Entre dans le conteneur :

```
$ kubectl exec -it log-writer -- sh
```

Affiche le contenu :

```
$ cat /mnt/data/log.txt
```

Tu devrais voir :

```
Fri Oct 10 10:15:02 UTC 2025 - Application still running
Fri Oct 10 10:15:07 UTC 2025 - Application still running
Fri Oct 10 10:15:12 UTC 2025 - Application still running
...
```

🚀 Le Pod écrit bien dans le fichier `log.txt` situé sur ton PV.

---

## Étape 5 — Vérifier la persistance après suppression

Supprime le Pod :

```
$ kubectl delete pod log-writer
```

Crée un **nouveau Pod** pour relire le fichier :

Fichier : **log-reader.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-reader
spec:
  containers:
    - name: reader
      image: busybox
      command: [ "sh", "-c" ]
      args:
        - cat /mnt/data/log.txt; sleep 60;
      volumeMounts:
        - mountPath: "/mnt/data"
          name: my-storage
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: local-pvc
```

**Commandes :**

```
$ kubectl apply -f log-reader.yaml
$ kubectl logs log-reader
```

Tu verras les **lignes précédemment écrites par l’application** :

```
Fri Oct 10 10:15:02 UTC 2025 - Application still running
Fri Oct 10 10:15:07 UTC 2025 - Application still running
...
```

✅ **Les logs ont bien persisté**, même après suppression du Pod initial.

---

## Étape 6 — Nettoyage

```
$ kubectl delete pod log-writer log-reader
$ kubectl delete pvc local-pvc
$ kubectl delete pv local-pv

$ minikube ssh
$ sudo rm -rf /data/pv
$ exit
```

