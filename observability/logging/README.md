# Logging Kubernetes

Dans cette section, nous abordons les mécanismes de **logging** dans Kubernetes.  
Commençons par rappeler le fonctionnement des logs dans **Docker**.

Lorsqu’on exécute un conteneur Docker, comme par exemple un simulateur d’événements (`event-simulator`) qui génère des événements aléatoires simulant des requêtes web, les messages produits par l’application sont envoyés vers la sortie standard (`stdout`).

- Si le conteneur est lancé en mode détaché (`-d`), les logs ne s’affichent pas directement.
- Pour les consulter, on utilise la commande :

```
$ docker logs <container_id>
```

En ajoutant l’option `-f`, on peut suivre les logs en temps réel.

***

### Passage à Kubernetes

Lorsque l’on crée un **Pod** basé sur la même image Docker, on peut visualiser ses logs avec la commande suivante :

```
$ kubectl logs <pod_name>
```

Comme pour Docker, l’option `-f` permet de suivre les logs en continu.

Il faut bien noter que ces logs correspondent uniquement aux conteneurs d’un **Pod**. Or, un Pod peut contenir plusieurs conteneurs.
- Dans ce cas, il est obligatoire de préciser le nom du conteneur dans la commande, faute de quoi `kubectl` retournera une erreur :

```
$ kubectl logs <pod_name> -c <container_name>
```

Par exemple :

```
$ kubectl logs event-pod -c event-simulator
```

***

### Exemple pratique

Voici un fichier YAML définissant un Pod avec deux conteneurs :
- `event-simulator` : génère des événements aléatoires
- `image-processor` : simule un service de traitement d’images

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: event-pod
spec:
  containers:
    - name: event-simulator
      image: busybox
      command: ["sh", "-c", "while true; do echo $(date) - Event generated; sleep 2; done"]
    - name: image-processor
      image: busybox
      command: ["sh", "-c", "while true; do echo $(date) - Processing image...; sleep 5; done"]
```

- Pour afficher les logs du service `event-simulator` :

```
$ kubectl logs event-pod -c event-simulator -f
```

- Pour afficher les logs du service `image-processor` :

```
$ kubectl logs event-pod -c image-processor -f
```

***

## Résumé concis

- En **Docker** :
    - `$ docker logs <container_id>` → affiche les logs.
    - `$ docker logs -f <container_id>` → suit les logs en temps réel.

- En **Kubernetes** :
    - `$ kubectl logs <pod_name>` → affiche les logs du Pod (si un seul conteneur).
    - `$ kubectl logs <pod_name> -c <container_name>` → nécessaire si le Pod contient plusieurs conteneurs.
    - `-f` → option pour suivre les logs en direct.

- Exemple avec un Pod multi-conteneurs : il faut toujours indiquer le nom du conteneur dont on veut afficher les logs.
