# ReplicaSets

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
    - Le `selector` n’est **pas obligatoire**.
    - Si tu ne le fournis pas, Kubernetes utilise automatiquement les labels du `template` pour identifier les Pods à gérer.
- Dans un **ReplicaSet** (successeur recommandé du RC) :
    - Le `selector` est **obligatoire**.
    - Tu dois définir un `matchLabels` (ou `matchExpressions`) pour indiquer explicitement quels Pods sont contrôlés.
    - Cela rend le comportement plus flexible, car un ReplicaSet peut gérer des Pods créés en dehors de lui, du moment qu’ils correspondent aux labels spécifiés.

