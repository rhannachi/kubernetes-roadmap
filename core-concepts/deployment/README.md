# Deployments

``` 
+----------------------------------------------------------+
|                      Deployment                          |
|                                                          |
|   +-----------------------------------------------+      |
|   |                  ReplicaSet                   |      |
|   |                                               |      |
|   |   +-----+  +-----+  +-----+  +-----+          |      |
|   |   | POD |  | POD |  | POD |  | POD | ...      |      |
|   |   +-----+  +-----+  +-----+  +-----+          |      |
|   |                                               |      |
|   +-----------------------------------------------+      |
|                                                          |
+----------------------------------------------------------+
```

Pour commencer, oublions un instant les concepts comme les **Pods** et les **ReplicaSets**.\
Imaginons comment vous souhaitez déployer votre application en environnement de production.

Par exemple, vous avez un serveur web à déployer. Il ne s’agit pas d’en exécuter un seul, mais plusieurs instances, pour des raisons évidentes de disponibilité et de charge.

De plus, chaque fois qu’une nouvelle version de votre application est disponible dans le registre Docker, vous souhaitez mettre à jour vos instances Docker de façon transparente.\
Toutefois, vous ne voulez pas mettre à jour toutes les instances simultanément, car cela pourrait impacter les utilisateurs en production. Vous préférez une mise à jour progressive, appelée **rolling update**.

Supposons que l’une de ces mises à jour introduise un bug inattendu : vous aimeriez pouvoir revenir rapidement à la version précédente, via un **rollback**.

Enfin, si vous souhaitez appliquer plusieurs modifications à votre environnement (mise à jour de la version du serveur web, ajustement de la scalabilité, modification des ressources allouées, etc.), vous ne voulez pas les appliquer immédiatement une à une.\
Vous préféreriez pouvoir suspendre temporairement le déploiement, appliquer toutes les modifications, puis reprendre le déploiement pour les déployer toutes ensemble.

Toutes ces fonctionnalités sont disponibles avec les **Deployments** de Kubernetes.

Jusqu’ici, nous avons vu que les **Pods** déploient des instances uniques d’une application, encapsulées dans des conteneurs.\
Plusieurs **Pods** peuvent être gérés par un **ReplicationController** ou un **ReplicaSet**.

Le **Deployment** est un objet Kubernetes qui vient au-dessus de ces couches et offre des fonctionnalités avancées comme les mises à jour progressives, les rollbacks, et la possibilité de suspendre/reprendre des déploiements.

### Comment créer un Deployment ?
Comme avec les autres ressources Kubernetes, on crée d’abord un fichier de définition YAML.

Le contenu est très similaire à celui d’un fichier de définition de **ReplicaSet**, à l’exception du champ `kind` qui sera maintenant `Deployment`.

Le fichier comporte :
- une version d’API `apiVersion: apps/v1`
- une section `metadata` avec un nom et des labels
- une section `spec` contenant un **selector**, un nombre de `replicas`, et un `template` qui définit le **Pod**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      name: front-pod
      labels:
        type: front-end
    spec:
      containers:
        - name: front-container
          image: nginx
```

Une fois le fichier prêt, on crée le Deployment avec la commande :
```
$ kubectl apply -f mon-deployment.yaml
```
Puis, on peut vérifier son existence avec :
```
$ kubectl get deployments
```

Lorsqu’un Deployment est créé, il génère automatiquement un **ReplicaSet**. Ainsi, en listant les ReplicaSets :
```
$ kubectl get replicasets
```
vous verrez celui associé au Deployment. Ce ReplicaSet crée à son tour les Pods.

En listant les Pods :
```
$ kubectl get pods
```
vous verrez les Pods créés par le ReplicaSet, tous portant le nom du Deployment et du ReplicaSet, facilitant le suivi.

### Conclusion
Jusqu’à présent, la différence entre ReplicaSet et Deployment peut sembler mince, mais le Deployment offre un contrôle plus avancé sur le cycle de vie et les mises à jour des Pods, avec un support natif pour les stratégies de déploiement, les rollbacks, et la gestion fine des mises à jour.

Enfin, pour visualiser tous les objets Kubernetes créés (Deployments, ReplicaSets, Pods, etc.), utilisez la commande :
```
$ kubectl get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/app-deployment-86dd5d4479-5jk4v   1/1     Running   0          18s
pod/app-deployment-86dd5d4479-bqdmx   1/1     Running   0          18s
pod/app-deployment-86dd5d4479-xc8n2   1/1     Running   0          18s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   42d

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-deployment   3/3     3            3           18s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/app-deployment-86dd5d4479   3         3         3       18s
```

***

## Résumé concis

- Un **Deployment** Kubernetes facilite le déploiement d’applications en production avec plusieurs instances (**Pods**).
- Il permet des mises à jour progressives (**rolling updates**) et des retours en arrière (**rollbacks**) en cas de problème.
- Les Deployments gèrent automatiquement un **ReplicaSet**, qui crée et supervise les Pods.
- Il est possible de suspendre, modifier et reprendre un déploiement pour appliquer plusieurs changements regroupés.
- Le fichier YAML de Deployment ressemble à celui d’un ReplicaSet, avec `kind: Deployment` et `apiVersion: apps/v1`.
- Commandes clés : `$ kubectl apply -f`, `$ kubectl get deployments`, `$ kubectl get replicasets`, `$ kubectl get pods`, `$ kubectl get all`.

