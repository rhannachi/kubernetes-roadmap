# ReplicaSets

[exercices](exercices/README.md)

### Le besoin de réplication
Imaginons un scénario simple avec un seul Pod hébergeant notre application. Si ce Pod échoue, les utilisateurs perdront immédiatement l’accès à l’application.\
Pour éviter cela, il est préférable d’exécuter plusieurs Pods de la même application en parallèle. Ainsi, même si un Pod tombe, d’autres continuent à fonctionner, assurant **haute disponibilité** et **tolérance aux pannes**.

Le **ReplicationController** joue justement ce rôle. Il garantit qu’un nombre défini de réplicas d’un Pod s’exécutent en permanence, qu’il s’agisse d’un seul Pod ou de plusieurs dizaines.\
Il crée automatiquement un nouveau Pod lorsqu’un Pod existant échoue. Cela permet non seulement de maintenir la disponibilité, mais aussi de **répartir la charge entre plusieurs Pods** exécutés sur différents Nodes du Cluster.

Le **ReplicationController/ReplicaSet** dans Kubernetes assure trois fonctions essentielles :
- **Haute disponibilité** : il maintient en permanence le nombre de Pods souhaité.
- **Tolérance aux pannes** : il recrée automatiquement un Pod lorsqu’un Pod existant échoue.
- **Répartition de charge** : en exécutant plusieurs Pods sur différents Nodes, le trafic peut être réparti entre eux (généralement via un Service).

***

### ReplicationController vs ReplicaSet
Il existe deux objets Kubernetes très similaires : **ReplicationController (RC)** et **ReplicaSet (RS)**.
- Le ReplicationController est la version historique.
- Le ReplicaSet est la version moderne et recommandée.

Tous deux assurent le même objectif : maintenir un nombre déterminé de Pods actifs. La principale différence est que le **ReplicaSet exige la définition d’un Selector**, qui permet de faire correspondre explicitement des Pods existants aux labels définis. Avec un ReplicationController, ce champ est implicite (pas obligatoire) : il reprend par défaut les labels spécifiés dans le Pod template.

***

### Création d’un ReplicationController
[replication-controller.yaml](replication-controller.yaml)\
Un ReplicationController est défini dans un fichier YAML et contient habituellement quatre sections principales :
1. **apiVersion** : pour RC, il s’agit de `v1`.
2. **kind** : défini comme `ReplicationController`.
3. **metadata** : inclut un nom et des labels d’identification.
4. **spec** : section clé qui définit :
    - `replicas` : nombre de Pods souhaités.
    - `selector` : (optionnel dans un RC) : labels permettant d’identifier les Pods à **surveiller/gérés**. Si absent, Kubernetes déduit automatiquement le selector à partir des labels définis dans le `template`.
    - `template` : définition du Pod à répliquer (similaire à un Pod YAML).

Une fois le fichier prêt, on exécute :
```
$ kubectl apply -f replication-controller.yaml
```
Cela crée le ReplicationController, qui déploie automatiquement le nombre spécifié de Pods. On peut ensuite :
- Vérifier les RC avec : `kubectl get rc` ou `kubectl get replicationcontroller`
- Vérifier les Pods créés avec : `kubectl get pods`

***

### Création d’un ReplicaSet
[replica-sets.yaml](replica-sets.yaml)\
Le processus est très similaire, mais avec quelques différences :
- **apiVersion** : `apps/v1`
- **kind** : `ReplicaSet`
- **spec** : doit contenir `replicas`, `template`, et obligatoirement un `selector`.

Exemple de commandes :
- Créer un RS : `kubectl apply -f replica-sets.yaml`
- Vérifier les ReplicaSets : `kubectl get rs` ou `kubectl get replicaset`
- Lister les Pods associés : `kubectl get pods`

### Mise à l’échelle
Un ReplicaSet peut être mis à l’échelle de deux manières :
1. Modifier directement la valeur **replicas** dans le fichier YAML, puis exécuter :
   ```
   $ kubectl apply -f replica-sets.yaml
   ```
2. Utiliser la commande d’échelle :
   ```
   $ kubectl scale rs app-rs --replicas=6
   ```

À noter : Kubernetes propose aussi des mécanismes avancés comme le **HorizontalPodAutoscaler** pour adapter automatiquement le nombre de Pods en fonction de la charge.

***

### À propos du champ `selector`

- Dans un **ReplicationController** :
    - Le champ `selector` est **optionnel**.
    - S'il n'est pas défini explicitement, Kubernetes utilise par défaut les labels du pod `template` pour déterminer quels Pods il doit gérer.
    - Le ReplicationController gère donc automatiquement les Pods créés à partir de son `template` et peut aussi reconnaître d’autres Pods existants ayant les mêmes labels. Cependant, seuls les Pods créés via son `template` sont sous sa gestion complète (création, remplacement).

- Dans un **ReplicaSet** (successeur recommandé du ReplicationController) :
    - Le champ `selector` est **obligatoire** et doit être défini explicitement (par `matchLabels` ou `matchExpressions`).
    - Ce selector indique précisément quels Pods le ReplicaSet doit surveiller pour maintenir le nombre désiré de réplicas.
    - Seuls les Pods créés à partir de son propre `template` sont gérés automatiquement (créés, remplacés en cas d’échec).
    - Les autres Pods, même s'ils ont des labels correspondants, ne seront *pas* sous la gestion automatique du ReplicaSet à moins d'avoir une référence propriétaire (ownerReference) liée à ce ReplicaSet, ce qui n'arrive que quand ils sont créés via son `template`.

En résumé, **un ReplicationController ou ReplicaSet gère automatiquement uniquement les Pods qu’il crée via son propre template.**  
Le selector sert à identifier les Pods associés, mais ne suffit pas à étendre la gestion automatique à des Pods externes au template.

