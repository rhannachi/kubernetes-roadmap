# Comprendre les Multi-Container Pods dans Kubernetes

Dans Kubernetes, un **Pod** peut contenir un ou plusieurs containers. Lorsqu’un Pod contient plusieurs containers, différentes stratégies (ou *patterns*) peuvent être utilisées pour définir leur rôle, leur ordre de démarrage et leur comportement tout au long du cycle de vie du Pod.  
Les trois principaux patterns sont :

1. **Co-located containers**
2. **Init Containers**
3. **Sidecar Containers**

***

## 1. Co-located containers

Les *co-located containers* représentent la forme la plus simple de multi-container Pod : plusieurs containers s’exécutent ensemble dans le même Pod.

- Tous démarrent en même temps
- Aucun ordre d’exécution n’est garanti
- Ils fonctionnent jusqu’à la fin du Pod
- Utiles lorsque deux services complémentaires doivent tourner ensemble sans dépendance stricte

#### Exemple

Dans cet exemple, deux containers simples (`nginx` et `redis`) sont définis dans le même Pod. Ils démarrent en parallèle :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: colocated-pod
spec:
  containers:
  - name: web
    image: nginx
  - name: cache
    image: redis
```

Ici, **nginx** et **redis** sont lancés ensemble. Mais il est impossible de s’assurer que `redis` démarre avant `nginx`.

***

## 2. Init Containers

Les **Init Containers** sont exécutés avant les containers applicatifs. Ils s’enchaînent de façon **séquentielle** et doivent terminer correctement avant que l’application principale ne démarre.\
Le rôle principal des Init Containers dans Kubernetes est précisément de préparer un environnement ou espace d’exécution pour les conteneurs d’application du Pod, puis de disparaître une fois leur tâche terminée

- Exécutés une seule fois
- Utile pour préparer l’environnement (ex. attendre un service externe, créer un volume, charger une configuration)
- Plusieurs Init Containers peuvent être définis, chacun s’exécutant dans l’ordre

#### Exemple

Un Pod dont l’application principale (`main-app`) attend que l'init container ait fini son exécution :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-init
spec:
  containers:
    - name: main-app
      image: busybox:1.28
      command: ['sh', '-c', 'echo "App démarrée" && sleep 3600']
  initContainers:
    - name: init-1
      image: busybox:1.28
      command: ['sh', '-c', 'echo "Init 1 terminé !" && sleep 3']
```

- L’Init Container `init-1` exécute une tâche d’initialisation (ici un simple message et un sleep de 3 secondes).
- Le conteneur principal `main-app` ne démarre qu’après la réussite complète de cet Init Container.
- Si plusieurs Init Containers étaient déclarés, ils s’exécuteraient séquentiellement, un par un, dans l’ordre où ils sont définis dans la spécification du Pod.

### Démarrage séquentiel des Init Containers 

Le démarrage séquentiel des Init Containers dans Kubernetes garantit que chaque étape d’initialisation s’exécute dans l’ordre avant le lancement du conteneur principal.\
Créer un Pod dont deux Init Containers **_s’exécutent l’un après l’autre_** avant que le conteneur d’application ne démarre.

#### Étape 1 : Exemple de manifeste YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-init-sequence
spec:
  containers:
    # (3)
    - name: main-app
      image: busybox:1.28
      command: ['sh', '-c', 'echo "App démarrée" && sleep 3600']
  initContainers:
    # (1)
    - name: init-1
      image: busybox:1.28
      command: ['sh', '-c', 'echo "Init 1 terminé !" && sleep 3']
    # (2)
    - name: init-2
      image: busybox:1.28
      command: ['sh', '-c', 'echo "Init 2 terminé !" && sleep 3']
```
- `init-1` s’exécute en premier, puis `init-2` démarre lorsque `init-1` a terminé sans erreur.
- Après l’exécution des deux, `main-app` se lance à son tour.

```
$ kubectl apply -f init-sequence.yaml
```

#### Étape 2 : Vérification du démarrage séquentiel
Pour observer l’ordre d’exécution, lancer cette commande :

```
$ kubectl describe pod demo-init-sequence

Init Containers:
  init-1:
    Container ID:  docker://1c117d6d573938090b4bc48fb9e3b60d217d6f3f7052a0c6c1ff41ec76e195a9
    Image:         busybox:1.28
    Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
    Command:
      sh
      -c
      echo "Init 1 terminé !" && sleep 3
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Fri, 26 Sep 2025 10:25:59 +0200
      Finished:     Fri, 26 Sep 2025 10:26:02 +0200
    Ready:          True
  init-2:
    Container ID:  docker://e951da55b6e6e03724bf6036da390ac9b510849a94231cbeb638309bd9c5a6dd
    Image:         busybox:1.28
    Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      echo "Init 2 terminé !" && sleep 3
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Fri, 26 Sep 2025 10:26:03 +0200
      Finished:     Fri, 26 Sep 2025 10:26:06 +0200
    Ready:          True
Containers:
  main-app:
    Container ID:  docker://2ac31cc87525aa8d92567dbb1e1c3d1294267c5c4d5bd1f611f451a86d2f68a4
    Image:         busybox:1.28
    Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      echo "App démarrée" && sleep 3600
    State:          Running
      Started:      Fri, 26 Sep 2025 10:26:07 +0200
    Ready:          True
...  
```
Dans l’état (State) du Pod, il sera visible que `init-1` a terminé avant que `init-2` ne commence, et enfin le conteneur principal prend le relais.\
On peut aussi vérifier dans les champs `Started` et `Finished` pour voir précisément quand chaque conteneur a démarré et terminé son exécution.

### Note
- Si un Init Container échoue, Kubernetes redémarre le Pod en repartant du premier Init Container.
- Les Init Containers doivent produire des effets idempotents, c’est-à-dire être conçus pour se réexécuter sans provoquer d’erreur ou d’état incohérent.

***

## 3. Sidecar Containers

Un **Sidecar Container** fonctionne aux côtés de l’application principale pendant toute la durée de vie du Pod.  
Contrairement aux Init Containers, qui s’arrêtent une fois leur tâche effectuée, le Sidecar reste actif.

- Peut démarrer avant l’application
- Continue de tourner tant que l’application tourne
- Utile pour fournir un service complémentaire (ex. collecte de logs, proxy, monitoring, synchronisation...)

#### Exemple

Pod avec une application `nginx` et un Sidecar basé sur **Filebeat** qui collecte et envoie les logs :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  containers:
  - name: web
    image: nginx
  - name: log-shipper
    image: docker.elastic.co/beats/filebeat:7.17.0
    volumeMounts:
    - name: varlog
      mountPath: /var/log/nginx
  volumes:
  - name: varlog
    emptyDir: {}
```

- `nginx` génère des logs dans `/var/log/nginx`.
- `filebeat` (le Sidecar) lit en continu ce répertoire et envoie les logs vers Elasticsearch ou une autre destination.
- Les deux containers restent actifs pendant toute la vie du Pod.

***

### Comparaison des patterns

| Pattern               | Démarrage          | Durée d'exécution      | Cas d’usage typique |
|-----------------------|-------------------|------------------------|---------------------|
| **Co-located**        | Tous en parallèle | Tout le cycle du Pod   | Deux services simples qui fonctionnent ensemble |
| **Init Containers**   | Séquentiel        | S’arrêtent après exécution | Préparation d’environnement, attente de dépendances |
| **Sidecar Containers**| Peut précéder l’app | Tout le cycle du Pod   | Logs, monitoring, proxy, synchronisation |

***

## Conclusion

- Utilisez **Co-located containers** pour exécuter plusieurs services ensemble sans dépendance stricte.
- Utilisez **Init Containers** pour gérer des dépendances et préparer votre environnement avant de démarrer votre application.
- Utilisez **Sidecar Containers** pour ajouter des fonctionnalités auxiliaires actives tout au long de la vie de votre application.
