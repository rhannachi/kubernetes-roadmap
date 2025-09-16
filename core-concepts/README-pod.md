# Pod

***

- Kubernetes ne déploie pas de conteneurs directement sur les Nodes: il encapsule chaque conteneur dans un objet appelé **Pod**.
- Un **Pod** est la plus petite unité de déploiement dans Kubernetes. Il représente une seule instance d’application.
- Le cas le plus courant: un pod contient un seul conteneur. Si l’on souhaite augmenter la capacité (scalabilité), il faut créer plusieurs pods, chacun avec une nouvelle instance de l’application.
- Si un Node est trop chargé, des Pods peuvent être répartis sur d’autres Nodes dans le cluster, ce qui permet d’augmenter la capacité globale.
- Pour faire évoluer une application, on ajoute ou supprime des Pods, pas des conteneurs à l’intérieur d’un même Pod.
- Les Pods peuvent parfois contenir plusieurs conteneurs: ces conteneurs doivent être très liés (exemple: une application principale et son conteneur “sidecar” de logging ou traitement de données).
- Dans un même Pod, tous les conteneurs partagent la même adresse IP, le même espace réseau et le même stockage: ils peuvent facilement communiquer et partager des fichiers.
- Cette cohabitation facilite la gestion du cycle de vie: si un Pod est supprimé, tous ses conteneurs le sont automatiquement.
- Dans Docker seul, tout doit être géré manuellement: le lien entre conteneurs, le stockage partagé, la surveillance… Avec Kubernetes, l’objet Pod simplifie et automatise tout cela.
- Même si souvent, on crée des Pods à un seul conteneur, Kubernetes oblige à encapsuler les applications dans des Pods, ce qui reste pérenne et flexible pour l’avenir.
- Pour créer un pod:
    - On utilise par exemple la commande:  
      `kubectl run nginx --image=nginx`
    - L’image du conteneur est téléchargée depuis le registre (Docker Hub par défaut).
- Pour voir les Pods du cluster:
    - Utiliser:  
      `kubectl get pods`
    - Le Pod passe successivement par les états `ContainerCreating` puis `Running` une fois démarré.

***

## Structure Pod YAML

Un fichier YAML Kubernetes suit une structure commune composée de quatre champs racine essentiels :
- `apiVersion` : précise la version de l’API Kubernetes utilisée pour l’objet.
- `kind` : indique le type d’objet à créer (ici, un Pod ; d’autres types possibles sont Service, ReplicaSet, Deployment).
- `metadata` : contient des informations sur l’objet comme son nom (`name`) et ses étiquettes (`labels`).
- `labels` sont des paires clé-valeur utiles pour identifier et filtrer les objets (par exemple, distinguer les Pod de type `front-end` ou `back-end`).
- `spec` : définit les spécifications de l’objet. Dans le cas d’un Pod, cela inclut la liste des conteneurs à déployer ; chaque conteneur est défini par un nom et une image, ici `nginx`.

Voici un exemple de fichier YAML pour créer un Pod nommé `myapp-pod` avec un conteneur NGINX :

```yaml
# pod-definition.yml

apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```

Pour créer ce Pod dans Kubernetes, utilisez la commande `kubectl create` ou `kubectl apply` :

* kubectl create : \
créer un nouvel objet dans le cluster à partir d’un fichier YAML.\
Si l’objet existe déjà avec le même nom, la commande échoue et signale une erreur.\
Usage typique : déploiement initial, quand l’objet n’existe pas encore.\
* kubectl apply : \
créer l’objet s’il n’existe pas, ou mettre à jour la configuration de l’objet s’il existe déjà.\
C’est la commande conseillée pour gérer les objets Kubernetes dans le temps, car elle permet des mises à jour sans supprimer et recréer complètement les ressources.

```
$ kubectl apply -f pod-definition.yml
```
Affiche la liste des Pods présents dans le cluster :\
La colonne READY dans la sortie de la commande kubectl get pods affiche le nombre de conteneurs en cours d'exécution et prêts dans un Pod, par rapport au nombre total de conteneurs définis dans ce Pod.
```
$ kubectl get pods
NAME            READY   STATUS             RESTARTS      AGE
newpods-mvvrh   1/1     Running            1 (13m ago)   29m
webapp          1/2     ImagePullBackOff   0             20m
```

Pour afficher les détails d’un Pod, la commande :\
Donne toutes les informations sur sa configuration, les labels, les conteneurs, et les événements associés.
```
$ kubectl describe pod myapp-pod
```

Création du Pod (méthode impérative):\
Pour créer un Pod nommé `my-pod-nginx` à partir de l’image officielle `nginx`
```
$ kubectl run my-pod-nginx --image=nginx
```
Suppression du Pod
```
$ kubectl delete pod my-pod-nginx
```

***

## Modifier un Pod Kubernetes

- Un Pod est une ressource immuable, on ne peut pas modifier toutes ses propriétés directement.
- Pour modifier un Pod :
  1. Extraire la définition YAML actuelle du Pod avec :  
     `kubectl get pod <nom-du-pod> -o yaml > pod-definition.yaml`
  2. Modifier le fichier YAML obtenu (par exemple changer l’image dans `spec.containers[*].image`).
  3. Supprimer l’ancien Pod avec :  
     `kubectl delete pod <nom-du-pod>`
  4. Recréer le Pod avec la nouvelle définition :  
     `kubectl apply -f pod-definition.yaml`

***

## Diagnostiquer un Pod `kubectl describe pod <my-pod>`

```
$ kubectl describe pod myapp-pod

Name:             myapp-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Tue, 16 Sep 2025 12:04:36 +0200
Labels:           app=myapp
                  type=front-end
Annotations:      <none>
Status:           Running
IP:               10.244.0.115
IPs:
  IP:  10.244.0.115
Containers:
  nginx-container:
    Container ID:   docker://7e6e814951f6daf1749416b05c1a730131bedb271083061407393a4af074953f
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:d5f28ef21aabddd098f3dbc21fe5b7a7d7a184720bc07da0b6c9b9820e97f25e
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 16 Sep 2025 12:04:37 +0200
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kzvtz (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-kzvtz:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  35s   default-scheduler  Successfully assigned default/myapp-pod to minikube
  Normal  Pulling    35s   kubelet            Pulling image "nginx"
  Normal  Pulled     34s   kubelet            Successfully pulled image "nginx" in 1.21s (1.21s including waiting). Image size: 192385289 bytes.
  Normal  Created    34s   kubelet            Created container: nginx-container
  Normal  Started    33s   kubelet            Started container nginx-container
```

Pour diagnostiquer rapidement si un Pod est en bon état à partir du résultat de la commande `kubectl describe pod <pod-name>`, voici les éléments essentiels à vérifier :

### 1. État général du Pod
- **Status :**
    - `Running` signifie que le Pod fonctionne normalement.
    - D'autres états comme `Pending`, `CrashLoopBackOff` ou `Failed` indiquent des problèmes.
- **Conditions :**
    - Cherchez `Ready` avec la valeur `True`. Cela signifie que le Pod est prêt à recevoir du trafic.
    - `Initialized` et `PodScheduled` doivent également être à `True`.

### 2. Conteneurs du Pod
- **State (État) :**
    - `Running` indique que le conteneur tourne.
    - `Waiting` ou `Terminated` peut signaler des problèmes ; vous pouvez inspecter les raisons dans les événements ou logs.
- **Ready :**
    - `True` signifie que le conteneur est prêt.
- **Restart Count :**
    - Un nombre élevé indique que le conteneur redémarre souvent, symptôme de crashs ou erreurs (souvent `CrashLoopBackOff`).

### 3. Informations sur l’image
- Vérifiez que l’image (`Image`) est bonne et a été correctement téléchargée (`Image pulled`).

### 4. Node et IP
- Confirmez sur quel **Node** le Pod tourne.
- Notez l’**IP** attribuée au Pod.

### 5. Volumes montés
- Vérifiez que les volumes nécessaires sont bien montés et notés dans la section `Mounts`.

### 6. Événements récents
- Regardez les événements en bas du descriptif :
    - `Scheduled` : Pod bien affecté à un node.
    - `Pulling` et `Pulled` : image récupérée correctement.
    - `Created` et `Started` : conteneur lancé correctement.
    - Événements de type `Warning` ou erreurs indiquent des problèmes à investiguer.

### En résumé rapide pour un Pod sain
- `Status` = Running
- Conditions `Ready`, `Initialized`, `PodScheduled` = True
- Conteneurs `State` = Running, `Ready` = True, `Restart Count` = 0
- Événements normaux sans erreurs ni warnings

***

## Diagnostiquer un Pod `kubectl logs <my-pod>`

Si des dysfonctionnements apparaissent (ex : Pod en Pending, CrashLoopBackOff, erreurs d’image), la commande `kubectl logs <my-pod>` permet d’aller plus loin pour analyser les logs des conteneurs.

```
$ kubectl logs myapp-pod

rypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2025/09/16 10:04:38 [notice] 1#1: using the "epoll" event method
2025/09/16 10:04:38 [notice] 1#1: nginx/1.29.1
2025/09/16 10:04:38 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14+deb12u1) 
2025/09/16 10:04:38 [notice] 1#1: OS: Linux 6.1.0-37-amd64
2025/09/16 10:04:38 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2025/09/16 10:04:38 [notice] 1#1: start worker processes
2025/09/16 10:04:38 [notice] 1#1: start worker process 30
2025/09/16 10:04:38 [notice] 1#1: start worker process 31
2025/09/16 10:04:38 [notice] 1#1: start worker process 32
```

Voici les éléments essentiels à vérifier :

### Points clés relevés dans les logs
- Le conteneur initialise correctement sa configuration sans erreur :
  `Configuration complete; ready for start up`
- Le serveur NGINX démarre avec la méthode d’événement `epoll` (optimisée pour Linux) :
  `using the "epoll" event method`
- La version de NGINX utilisée est la 1.29.1, compilée avec GCC 12.2 sur Debian Linux.  
- Le serveur lance plusieurs processus worker qui vont gérer les requêtes web :
```
  start worker process 30
  start worker process 31
  start worker process 32
```

