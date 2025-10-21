## Exercice 1:
Quel contrôleur d'admission n'est pas activé par défaut?

**Réponse correcte:**  
NamespaceAutoProvision

**Explication:**  
Dans Kubernetes, les contrôleurs d'admission tels que ValidatingAdmissionWebhook, MutatingAdmissionWebhook et NamespaceLifecycle sont activés par défaut dans la plupart des clusters pour assurer la conformité, la sécurité et le bon cycle de vie des namespaces.  
En revanche, NamespaceAutoProvision n'est pas activé par défaut, car il permettrait de créer automatiquement des namespaces lorsque des ressources les utilisent, ce qui pourrait entraîner la création involontaire de namespaces non désirés. Cette fonctionnalité doit donc être explicitement activée si besoin.[2][21]

***

## Exercice 2:

#### Exemple de commandes pour observer et modifier les Admission Controllers

1. **Voir les namespaces existants :**
```bash
kubectl get namespace
```
2. **Essayer de créer un pod dans un namespace non existant :**
```bash
kubectl run nginx --image=nginx -n blue
```
Cette commande échoue si le namespace `blue` n'existe pas.

3. **Vérifier la configuration des admission controllers du kube-apiserver :**
```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep enable-admission-plugins
```
Ici, tu vois souvent une liste de plugins activés. Par défaut, certains admission controllers, comme `NamespaceLifecycle`, sont activés.

#### Activer le contrôleur NamespaceAutoProvision

`NamespaceAutoProvision` est un admission controller qui crée automatiquement un namespace s'il n'existe pas au moment de la création d'une ressource. Il est souvent **désactivé par défaut** pour éviter des créations non désirées.

Pour activer `NamespaceAutoProvision`, il faut :

1. Modifier le manifeste du kube-apiserver (`/etc/kubernetes/manifests/kube-apiserver.yaml`) sur le nœud control-plane.
2. Ajouter `NamespaceAutoProvision` à la liste des plugins activés dans la ligne :

```yaml
- --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
```

Puis sauvegarder et laisser Kubernetes redémarrer automatiquement le kube-apiserver.

**Attention :** Active ce plugin uniquement si tu comprends les conséquences, car il créera de nouveaux namespaces automatiquement, ce qui peut compliquer la gouvernance du cluster.

```
$ kubectl run nginx --image=nginx -n blue
pod/nginx created
```

Si tu utilises Minikube, tu peux démarrer le cluster avec ce flag :
```bash
minikube delete
minikube start \
  --extra-config=apiserver.enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
```

#### Info  
Veuillez noter que les *admission controllers* NamespaceExists et NamespaceAutoProvision ont été dépréciés et sont maintenant remplacés par le *admission controller* NamespaceLifecycle.  
Ce dernier garantit que toute requête faite sur un namespace non existant est rejetée, et il protège les namespaces système par défaut, notamment default, kube-system et kube-public, contre la suppression.

En résumé, l’admission controller NamespaceLifecycle prend en charge la gestion complète du cycle de vie des namespaces, rendant ainsi obsolètes les anciens contrôleurs NamespaceExists et NamespaceAutoProvision.  
Il assure la sécurité en empêchant la création d’objets dans des namespaces inexistants et protège les namespaces critiques du cluster.

C’est la raison pour laquelle tu ne trouveras plus ces anciens admission controllers activés dans les versions récentes de Kubernetes, ils ont été consolidés sous NamespaceLifecycle pour plus de cohérence et de simplicité

***

## Exercice 3:
**Effet du plugin `DefaultStorageClass` sur la création de PVC**

Cet exercice te permettra d’observer en pratique comment Kubernetes attribue automatiquement une **StorageClass par défaut** à un `PersistentVolumeClaim (PVC)` lorsque le plugin d’admission `DefaultStorageClass` est activé — puis de constater son absence après désactivation.

### Objectif
1. Comprendre le rôle du plugin `DefaultStorageClass`.
2. Observer son effet sur la création automatique d’une StorageClass.
3. Appliquer deux méthodes pour le désactiver (dans Minikube et dans un environnement Kubernetes standard).
4. Vérifier que Kubernetes n’associe plus automatiquement de StorageClass après désactivation.

### Partie 1 – Situation avant désactivation du plugin

1. **Créer un PVC sans préciser de StorageClass :**

```bash
kubectl apply -f myclaim.yaml
```

Contenu du manifeste :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 0.5Gi
```

2. **Lister les StorageClasses existantes :**

```bash
kubectl get storageclass
```

Exemple de sortie :

| NAME | PROVISIONER | DEFAULT |
|------|--------------|----------|
| default (default) | rancher.io/local-path | ✅ |

3. **Observer le PVC créé :**

```bash
kubectl get pvc
```

Résultat attendu : la colonne `STORAGECLASS` affiche `default`. Cela montre que **le plugin `DefaultStorageClass`** a automatiquement associé la StorageClass par défaut, bien que le manifeste du PVC ne la définisse pas.

### Partie 2 – Après désactivation du plugin

#### Méthode 1 : Dans un cluster classique (kubeadm)

1. Édite le manifeste du kube-apiserver :

```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

2. Ajoute l’option suivante à la ligne de commande du `kube-apiserver` :

```yaml
--disable-admission-plugins=DefaultStorageClass
```

3. Sauvegarde et ferme le fichier. Attends quelques minutes que le **kube-apiserver** redémarre (puisque c’est un pod statique sous `/etc/kubernetes/manifests`).

4. **Vérifie à nouveau :**

```bash
kubectl delete pvc myclaim
kubectl apply -f myclaim.yaml
kubectl get pvc
```

Résultat : aucune StorageClass n’est désormais affectée par défaut (`STORAGECLASS` vide).

#### Méthode 2 : Dans Minikube

1. **Relance Minikube** en désactivant explicitement le plugin :

```bash
minikube start \
  --extra-config=apiserver.disable-admission-plugins=DefaultStorageClass
```

2. **Vérifie à nouveau :**

```bash
kubectl delete pvc myclaim
kubectl apply -f myclaim.yaml
kubectl get pvc
```

Résultat attendu : la colonne `STORAGECLASS` reste vide → **aucune StorageClass par défaut appliquée.**

#### Vérification complémentaire

Tu peux également supprimer toute StorageClass marquée comme par défaut pour éviter le provisionnement automatique :

```bash
kubectl patch storageclass default -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

#### Résumé

| État du cluster | Action sur le PVC | Résultat |
|------------------|------------------|-----------|
| Plugin actif | Créer un PVC sans `storageClassName` | `STORAGECLASS` automatiquement = `default` |
| Plugin désactivé | Créer un PVC sans `storageClassName` | Aucun `STORAGECLASS` attribué |

***

