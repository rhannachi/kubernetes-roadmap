## Readiness probes

Dans cette section, nous allons parler des **probes** dans Kubernetes, en commençant par les **readiness probes**.  
Avant d’entrer dans le détail, rappelons brièvement le **cycle de vie d’un Pod**, car il est essentiel pour comprendre l’importance des probes.

### Cycle de vie d’un Pod
Un **Pod** possède deux éléments principaux pour décrire son état : le **status** et les **conditions**.

- **Le status du Pod** indique une étape de haut niveau :
    - Lors de sa création, le Pod est en état *Pending*, en attente que le **scheduler** lui trouve un **Node**.  
      Si aucun Node n’est disponible, il reste bloqué à ce stade.  
      Pour en diagnostiquer la cause, on utilise la commande :
      ```
      $ kubectl describe pod
      ```
    - Une fois planifié, il passe en **ContainerCreating**, ce qui correspond au téléchargement des images et au lancement des containers.
    - Quand tous les containers fonctionnent, le Pod atteint l’état **Running**.  
      Il y reste jusqu’à ce que le programme se termine ou soit supprimé.

On peut consulter cet état général avec :
```
$ kubectl get pods
```

Attention : le **status** du Pod donne uniquement une vision globale, mais il ne permet pas de savoir si l’application à l’intérieur est réellement disponible.

***

### Les conditions du Pod
Pour avoir plus de détails, Kubernetes utilise des **conditions**. Ce sont des drapeaux booléens (`True` ou `False`) qui précisent le statut réel du Pod :

- *PodScheduled* → `True` quand le Pod est affecté à un Node.
- *Initialized* → `True` une fois les initialisations terminées.
- *ContainersReady* → `True` lorsque tous les containers du Pod sont prêts.
- *Ready* → `True` lorsque l’application est considérée disponible pour recevoir du trafic.

On peut voir ces conditions avec :
```
$ kubectl describe pod
```
ou directement dans la colonne *READY* avec :
```
$ kubectl get pods
```

La condition *Ready* est capitale, car elle détermine si un **Service** Kubernetes enverra du trafic vers ce Pod.

***

### Le problème : quand une application n’est pas immédiatement prête
Par défaut, Kubernetes suppose qu’un container est *Ready* dès qu’il est démarré.  
Cependant, dans la réalité, les applications peuvent nécessiter un temps d’initialisation :

- Un script peut être disponible en quelques millisecondes.
- Une base de données peut prendre plusieurs secondes avant d’ouvrir son socket.
- Un serveur web ou un **Jenkins server** peut mettre 10 à 15 secondes, voire plus, avant d’être réellement prête à servir les utilisateurs.

Pendant ce temps, Kubernetes considère le Pod comme *Ready* et donc un Service peut rediriger du trafic trop tôt.\
**Résultat : les utilisateurs tombent sur un Pod qui n’a pas encore d’application opérationnelle.**

***

### La solution : readiness probes
Pour éviter ce problème, Kubernetes propose les **readiness probes**.  
Elles permettent de tester l’état réel de l’application dans le container et de synchroniser la condition *Ready* avec la disponibilité effective du service.

Un développeur peut définir le critère exact qui signifie “application prête”. Kubernetes propose trois types de probes :

1. **HTTP GET** : envoie une requête HTTP vers un chemin (ex : `/healthz`) et considère l’application prête si la réponse est positive.
```yaml
  containers:
  - name: web-app
    image: my-web-app:1.0
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
```
2. **TCP Socket** : teste l’ouverture d’un port TCP. Si le port accepte la connexion, l’application est considérée prête.
```yaml
  containers:
  - name: mysql
    image: mysql:8.0
    ports:
    - containerPort: 3306
    readinessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
```
3. **Exec command** : exécute un script ou une commande dans le container. Si celle-ci se termine avec un code 0 (succès), la probe indique que l’application est prête.
```yaml
  containers:
  - name: custom-app
    image: custom-app:1.0
    readinessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - /check-ready.sh
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
```
***

### Configuration d’une readiness probe
Dans le fichier de définition d’un Pod, on ajoute un champ `readinessProbe`.  
Exemple pour un test HTTP :
```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
```

On peut également personnaliser le comportement :
- `initialDelaySeconds`: délai avant de commencer les tests (utile si l’on sait que l’application met au moins X secondes pour démarrer).
- `periodSeconds`: fréquence des tests.
- `failureThreshold`: nombre d’échecs consécutifs avant de marquer le Pod comme non prêt.

Ainsi, tant que la probe échoue, le Pod n’est pas marqué *Ready* et ne reçoit pas de trafic via le Service.

***

### Cas pratique : un Deployment multi-Pods
Imaginons un **ReplicaSet** ou un **Deployment** avec plusieurs Pods derrière un Service.
- Deux Pods existants servent déjà les utilisateurs.
- Un nouveau Pod est ajouté mais met 60 secondes à se préparer.

Sans readiness probe : le Service envoie immédiatement du trafic au nouveau Pod → certains utilisateurs rencontrent des erreurs.  
Avec readiness probe : le trafic continue vers les Pods déjà prêts. Le nouveau Pod ne reçoit de requêtes que lorsqu’il a passé le test avec succès.

Résultat : aucune interruption pour les utilisateurs.

***

## Résumé concis
- Le **status** du Pod (Pending, Running, etc.) donne une vue globale mais ne garantit pas la disponibilité réelle de l’application.
- Les **conditions** précisent si le Pod est prêt (*Ready*).
- Par défaut, Kubernetes suppose qu’un container est prêt dès son lancement, ce qui est faux pour certaines applications longues à démarrer (BDD, serveurs web, Jenkins, etc.).
- Une **readiness probe** permet d’indiquer à Kubernetes quand l’application est effectivement prête.
    - Types : HTTP GET, TCP Socket, Exec command.
    - Options : délais (`initialDelaySeconds`), fréquence (`periodSeconds`), seuils (`failureThreshold`).
- Dans un déploiement multi-Pods, les readiness probes évitent d’envoyer du trafic trop tôt à un Pod non fonctionnel, assurant une meilleure fiabilité.

