## Gestion des utilisateurs
[Exercice](exercices/pod-security-context.yaml)\
Par défaut, Docker exécute les processus à l’intérieur d’un conteneur avec l’utilisateur root. Cela peut représenter un risque, même si Docker restreint les privilèges root à l’aide des *capabilities* Linux.

Il est recommandé de :
- Lancer un conteneur avec un utilisateur non-root via l’option `--user` de `docker run`.
- Définir l’utilisateur dans l’image Docker elle-même grâce à l’instruction `USER`.

Les *capabilities* permettent de limiter ou étendre les privilèges disponibles. Par exemple :
- `--cap-add` pour ajouter des privilèges.
- `--cap-drop` pour en retirer.
- `--privileged` pour accorder tous les privilèges (fortement déconseillé en production).

Voici des exemples concrets pour bien comprendre les concepts liés à l'utilisateur non-root, aux capabilities et à l'option --privileged en Docker.

### 1. Lancer un conteneur avec un utilisateur non-root via --user

Par défaut, un conteneur Docker s'exécute en tant que root, ce qui peut poser des problèmes de sécurité. Pour lancer un conteneur avec un utilisateur non-root, on peut utiliser l'option --user :

```
docker run --user 1001 nginx
```

Ici, le conteneur tourne avec l'utilisateur dont l'UID est 1001. Cela empêche le conteneur d'avoir les privilèges root.

***

### 2. Définir l'utilisateur dans l'image Docker avec l'instruction USER

Dans le Dockerfile, on peut définir l'utilisateur qui va exécuter les processus par défaut via l'instruction USER :

```Dockerfile
FROM nginx
RUN useradd -u 1001 appuser
USER 1001
```

Ainsi, quand on lance un conteneur basé sur cette image, il s'exécutera avec l'utilisateur "appuser" (UID 1001) par défaut, renforçant la sécurité.

***

### 3. Limiter ou étendre les privilèges avec les capabilities

Linux capabilities sont des privilèges spécifiques que l'on peut ajouter ou retirer d'un conteneur pour limiter les risques.

- Ajouter une capability avec --cap-add :

```
$ docker run --cap-add=NET_ADMIN ubuntu ip link add dummy0 type dummy
```

Ici, on ajoute la capability NET_ADMIN permettant de gérer les interfaces réseaux, nécessaire par exemple pour la commande `ip link add`.

- Retirer une capability avec --cap-drop :

```
$ docker run --cap-drop=CHOWN alpine
```

Cela retire la capability CHOWN qui permet de modifier la propriété des fichiers, renforçant la sécurité.

- Exemple combiné pour une sécurité renforcée :

```
$ docker run --cap-drop=ALL --cap-add=CHOWN alpine
```

On retire toutes les capabilities sauf celle nécessaire pour changer la propriété des fichiers.

***

### 4. Usage de --privileged (à éviter en production)

Le flag --privileged donne tous les privilèges au conteneur, lui permettant quasiment tout ce qu’un processus root sur l’hôte peut faire :

```
$ docker run --privileged -it ubuntu bash
```

Exemple où on peut monter un système de fichiers dans le conteneur (impossible sans privilèges étendus) :

```
$ docker run --privileged -it ubuntu bash
# mount -t tmpfs none /mnt
```

***
***

## Transition vers Kubernetes
Comme nous l’avons vu avec Docker, il est possible de configurer différents paramètres de sécurité pour les conteneurs. Kubernetes reprend ces concepts via le champ **securityContext**.

Un Pod encapsule un ou plusieurs conteneurs. Le *securityContext* peut être appliqué :
- Au niveau du Pod (s’applique à tous ses conteneurs).
- Au niveau d’un conteneur (prioritaire sur celui défini au Pod).

Les options disponibles incluent :
- `runAsUser` pour exécuter les processus avec un utilisateur spécifique.
- `capabilities` pour ajouter ou supprimer certains privilèges Linux.

***

### Exemple au niveau d'un Pod

Dans cet exemple, le **securityContext** est défini dans la spécification du Pod. Toutes les containers du Pod héritent de ces paramètres (exemple simplifié) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-securitycontext-example
spec:
  securityContext:
    runAsUser: 1000          # L'ensemble du Pod s'exécute avec cet utilisateur
    capabilities:
      add: ["NET_ADMIN"]      # Ajoute la capacité NET_ADMIN à tous les conteneurs
      drop: ["SYS_ADMIN"]     # Supprime la capacité SYS_ADMIN
  containers:
  - name: app-container
    image: ubuntu
    command: ["sleep", "3600"]
```

Ici, tous les containers du Pod s’exécutent avec l’UID 1000 et bénéficient des capabilities configurées dans le Pod. Ces paramètres s’appliquent globalement sauf s’ils sont explicitement surchargés au niveau des containers.

***

### Exemple au niveau d'un Container

Dans cet exemple, le Pod définit certains paramètres, mais un container applique un **securityContext** spécifique qui prévaut pour lui seul :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: container-securitycontext-example
spec:
  securityContext:
    runAsUser: 1000            # Par défaut au niveau du Pod
  containers:
  - name: app-container
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 2000          # Surcharge l'UID pour ce container uniquement
      capabilities:
        add: ["SYS_TIME"]       # Ajoute la capacité SYS_TIME uniquement à ce container
        drop: ["NET_RAW"]       # Supprime la capacité NET_RAW uniquement à ce container
```

Ici, le container s’exécute avec l’UID 2000 (différent du Pod) et a ses propres capacités Linux définies indépendamment du Pod.

Ce mécanisme permet d’améliorer la sécurité en maîtrisant précisément les privilèges accordés aux conteneurs dans un Cluster Kubernetes.

***

## Résumé concis

- Les conteneurs Docker partagent le noyau avec l’hôte et utilisent les espaces de noms Linux pour isoler leurs processus.
- Par défaut, Docker exécute les conteneurs avec l’utilisateur root, mais il est fortement recommandé de définir un utilisateur non-root (via `--user` ou l’instruction `USER`).
- Docker limite les privilèges root grâce aux *capabilities* Linux, qu’il est possible de contrôler avec `--cap-add`, `--cap-drop` ou `--privileged`.
- Dans Kubernetes, les règles de sécurité sont définies via **securityContext**.
- Ces paramètres peuvent être appliqués au niveau d’un Pod ou d’un conteneur, avec priorité au niveau du conteneur.
- Les options incluent notamment `runAsUser` et `capabilities`.

***
