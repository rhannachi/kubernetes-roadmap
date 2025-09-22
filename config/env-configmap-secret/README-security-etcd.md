# Sécurisation des Secrets Kubernetes par chiffrement au repos dans etcd

## Introduction

Dans Kubernetes, les objets **Secret** contiennent souvent des données sensibles (ex : mots de passe, clés API). Par défaut, ces Secrets sont encodés en base64 mais stockés en clair dans le backend **etcd**, ce qui constitue un risque si quelqu’un obtient l’accès à etcd. Pour renforcer la sécurité, Kubernetes permet d’activer le **chiffrement au repos (Encryption at Rest)**, garantissant que les Secrets sont chiffrés avant leur stockage dans etcd.

Ce tutoriel détaille toutes les étapes nécessaires pour mettre en place ce chiffrement.

***

## Prérequis

- Un cluster Kubernetes fonctionnel (idéalement configuré avec kubeadm)
- Accès au serveur **kube-apiserver** (sudo/root)
- Utilitaire `kubectl` configuré pour accéder au cluster
- Accès à l’outil client etcd (`etcdctl`) pour vérifier les données dans etcd (optionnel mais recommandé)
    ```
    $ sudo apt-get update
    $ sudo apt-get install etcd-client
    ```
- Connaissance basique des pods, volumes, et manifests Kubernetes

***

## Étape 1 : Vérifier le statut actuel des Secrets

1. Créez un Secret générique simple pour voir son fonctionnement de base.

```
$ kubectl create secret generic my-secret --from-literal=password=supersecret
```

2. Vérifiez que le Secret a bien été créé.

```
$ kubectl get secret my-secret
NAME        TYPE     DATA   AGE
my-secret   Opaque   1      3m59s

$ kubectl describe secret my-secret
Name:         my-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>
Type:  Opaque
Data
====
password:  11 bytes
```

3. Observez que les données du Secret sont encodées en base64, mais non chiffrées. Pour vérifier ce point, extrayez la donnée encodée et décodez-la :

```
$ kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 --decode
```

Vous verrez la valeur originale "supersecret" en clair.

***

## Étape 2 : Vérifier que le chiffrement au repos n’est pas activé

Le chiffrement s’active via l’option `--encryption-provider-config` du **kube-apiserver**. Contrôlez si cette option est présente :

1. Repérez le manifeste du **kube-apiserver**. Sur un cluster kubeadm, il se trouve souvent dans `/etc/kubernetes/manifests/kube-apiserver.yaml`.

2. Recherchez la ligne suivante dans le manifeste :

```yaml
--encryption-provider-config=/path/to/encryption-config.yaml
```

Si cette ligne n’existe pas, le chiffrement n’est pas activé.

***

## Étape 3 : Créer un fichier de configuration de chiffrement

1. Créez un fichier YAML, par exemple `encryption-config.yaml`, avec ce contenu minimaliste pour chiffrer uniquement les Secrets :

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <BASE64_ENCODED_32_BYTE_KEY>
      - identity: {}
```

- **resources** : liste des ressources à chiffrer (ici seulement les `secrets`).
- **providers** : liste des méthodes de chiffrement. Le premier sera utilisé pour le chiffrement, les autres pour le déchiffrement.
- `aescbc` est un algorithme de chiffrement symétrique sécurisé.
- `identity` signifie « pas de chiffrement », nécessaire pour la compatibilité.

2. Générer une clé secrète de 32 octets encodée en base64 (par exemple sur votre machine Linux) :

```
head -c 32 /dev/urandom | base64
```

Remplacez `<BASE64_ENCODED_32_BYTE_KEY>` par la valeur retournée.

***

## Étape 4 : Déployer la configuration dans le kube-apiserver

1. Placez `encryption-config.yaml` sur le **Control Plane Node**, par exemple dans `/etc/kubernetes/pki/`.

```
sudo mv encryption-config.yaml /etc/kubernetes/pki/
sudo chmod 600 /etc/kubernetes/pki/encryption-config.yaml
```

2. Modifiez le fichier manifeste du **kube-apiserver** (`/etc/kubernetes/manifests/kube-apiserver.yaml`) pour ajouter l’option suivante dans le bloc `command` :

```yaml
- --encryption-provider-config=/etc/kubernetes/pki/encryption-config.yaml
```

3. Montez le fichier `encryption-config.yaml` dans le pod en ajoutant un volume `hostPath` et le volumeMount associé :

```yaml
volumes:
  - name: encryption-config
    hostPath:
      path: /etc/kubernetes/pki/encryption-config.yaml
      type: File
containers:
  - name: kube-apiserver
    volumeMounts:
      - mountPath: /etc/kubernetes/pki/encryption-config.yaml
        name: encryption-config
        readOnly: true
```

4. Enregistrez et quittez. Kubernetes détectera automatiquement la modification du manifeste et redémarrera le pod **kube-apiserver**.

***

## Étape 5 : Vérification après l’activation du chiffrement

1. Créez un nouveau Secret, par exemple :

```
$ kubectl create secret generic my-secret2 --from-literal=password=topsecret
```

2. Utilisez la commande `etcdctl` pour récupérer la donnée dans etcd et vérifier qu’elle est désormais chiffrée (non lisible en clair).

En pratique, vous aurez besoin des certificats client etcd et de la commande adaptée, par exemple :

```
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key \
  get /registry/secrets/default/my-secret2
```

3. Vous devriez voir la valeur chiffrée, illisible directement.

4. Constats importants :
- Les Secrets créés avant activation restent non chiffrés.
- Pour chiffrer les anciens Secrets, mettez-les à jour en les ré-appliquant avec `$ kubectl apply -f secret.yaml`.

***

## Conclusion

En activant le chiffrement au repos, vous sécurisez efficacement les données sensibles des Secrets dans votre cluster Kubernetes en les protégeant au niveau de la couche de stockage etcd.

Cette méthode est indispensable pour répondre aux exigences de conformité et pour limiter la surface d’attaque en cas d’accès non autorisé à etcd.
