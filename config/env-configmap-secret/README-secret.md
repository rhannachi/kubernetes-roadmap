# Secrets
[Exercice](deployment-secret.yaml)\
Les Secrets dans Kubernetes sont des objets conçus pour stocker et gérer des informations sensibles comme les mots de passe, les clés API, les certificats, etc. Ils permettent d’isoler ces données confidentielles du code source, en les encodant en base64 et en limitant leur accès aux pods qui en ont besoin, évitant ainsi de les exposer dans les configurations standard ou le code.

## Concepts clés des Secrets Kubernetes
- Un Secret est un objet au sein d’un namespace Kubernetes qui contient des paires clé-valeur sensibles.
- Les données sont encodées en base64 dans l’objet Secret.
- Ils peuvent être utilisés de deux manières principales : injectés en tant que variables d’environnement dans les pods ou montés en tant que fichiers dans des volumes.
- Par défaut, les Secrets sont stockés dans etcd en clair, mais dans des environnements comme AWS EKS, il est recommandé d’utiliser des solutions comme AWS KMS pour chiffrer les secrets.
- L’accès aux Secrets est restreint au namespace et aux pods autorisés, mais il faut être conscient que tous les pods dans un même namespace peuvent accéder aux Secrets.

## Exemple simple de création de Secret

Supposons que l’on veuille créer un Secret avec un nom d’utilisateur et un mot de passe.

D'abord, on encode les valeurs en base64 :
```
$ echo -n 'admin' | base64     # YWRtaW4=
$ echo -n '1f2d1e2e67df' | base64 # MWYyZDFlMmU2N2Rm
```

Exemple de manifeste YAML pour un Secret :
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

On crée ensuite ce Secret dans Kubernetes avec la commande :
```
$ kubectl apply -f secret.yaml
```

## Utilisation des Secrets dans un Pod

### Injection en variables d’environnement
Un pod peut référencer un Secret pour alimenter ses variables d’environnement comme suit :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: nginx
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password
```

### Montage en volume
On peut aussi monter le Secret en volume pour que les données sensibles apparaissent sous forme de fichiers dans un chemin donné :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secret"
  volumes:
  - name: secret-volume
    secret:
      secretName: mysecret
```
Dans ce cas, les fichiers `/etc/secret/username` et `/etc/secret/password` contiennent les données décodées du Secret.

## Précautions et bonnes pratiques
- N’utilisez pas les Secrets comme variables d’environnement si la confidentialité est cruciale, car ces variables peuvent apparaître dans les logs.
- Utilisez un chiffrement externe comme AWS KMS pour protéger les Secrets au repos.
- Faites pivoter vos secrets régulièrement, Kubernetes ne le fait pas automatiquement.
- Séparez les namespaces pour isoler les secrets des applications différentes.
- Préférez le montage de Secrets sous forme de volumes montés en tmpfs pour une meilleure sécurité.
- Pour aller plus loin dans la gestion et la configuration des secrets avec AWS et Kubernetes: https://www.youtube.com/watch?v=MTnQW9MxnRI
