## Variables d'environnement dans Kubernetes

Dans Kubernetes, les variables d'environnement permettent de passer des données de configuration à un conteneur à son démarrage.

### Exemple simple de variable d'environnement dans un Pod :

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

## ConfigMaps dans Kubernetes

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

```bash
kubectl apply -f configmap.yaml
```

***

## Utiliser un ConfigMap dans un Pod comme variable d'environnement

Pour injecter une valeur issue d’un ConfigMap dans un conteneur, on utilise `valueFrom` avec `configMapKeyRef` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-configmap
spec:
  containers:
    - name: mycontainer
      image: alpine
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "Log level is: $LOG_LEVEL"
            sleep 10
          done
```

Ici, la variable d'environnement `LOG_LEVEL` prend sa valeur depuis la clé `LOG_LEVEL` du ConfigMap `app-config`.

```
$ kubectl get configmap
$ kubectl describe configmap app-config
```

***

## Résumé

- Les **variables d'environnement** permettent de passer des configurations simples directement dans un Pod via l'attribut `env`.
- Les **ConfigMaps** servent à stocker des configurations plus complexes ou partagées, qui peuvent être injectées dans les Pods via `valueFrom`.
- Cette séparation facilite la gestion des configurations, la réutilisation et la modification sans rebuild des images.
