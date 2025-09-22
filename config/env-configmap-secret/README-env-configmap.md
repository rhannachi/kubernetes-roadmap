## Variables d'environnement

Dans Kubernetes, les variables d'environnement permettent de passer des données de configuration à un conteneur à son démarrage.

### Variable d'environnement dans un Pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
    - name: mycontainer
      image: alpine
      env:
        - name: DEMO_ENV
          value: "Bonjour depuis la variable d'environnement"
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo $DEMO_ENV
            sleep 10
          done
```

Dans cet exemple, la variable `DEMO_ENV` est définie avec une valeur simple. Le conteneur affichera cette valeur toutes les 10 secondes.

***
***

## ConfigMaps

Un **ConfigMap** est un objet Kubernetes qui permet de stocker des données de configuration sous forme de paires clé-valeur, séparément de l’image conteneur. Cela permet de modifier la configuration sans reconstruire l’image.

### Exemple de création d’un ConfigMap :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "DEBUG"
  APP_COLOR: "green"
```

Créer ce ConfigMap avec :

```
$ kubectl apply -f configmap.yaml
```

***
Voici des exemples concrets pour bien comprendre l’utilisation des **ConfigMaps** dans Kubernetes :

***

### 1. Création d’un ConfigMap simple

Fichier `myconfigmap.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
data:
  username: k8s-admin
  access_level: "1"
```

Création dans Kubernetes :

```
$ kubectl apply -f myconfigmap.yaml
$ kubectl describe configmap myconfigmap
```

***

### 2. Utiliser un ConfigMap dans un Pod comme variables d’environnement (toutes les clés)

Fichier `env-configmap.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-configmap
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c", "printenv"]
    envFrom:
    - configMapRef:
        name: myconfigmap
```

- Toute la config du ConfigMap `myconfigmap` est injectée comme variables d’environnement.

***

### 3. Utiliser une clé spécifique du ConfigMap comme variable d’environnement

```yaml
env:
- name: USER_NAME
  valueFrom:
    configMapKeyRef:
      name: myconfigmap
      key: username
```

- Ici on injecte seulement la clé `username` du ConfigMap comme variable `USER_NAME`.

***

### 4. Monter un ConfigMap comme volume pour accéder aux données comme fichiers

```yaml
volumes:
- name: config-volume
  configMap:
    name: myconfigmap

volumeMounts:
- name: config-volume
  mountPath: /etc/config
```

- Les clés du ConfigMap deviennent des fichiers dans `/etc/config`.

