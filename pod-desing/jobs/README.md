## Jobs dans Kubernetes

### Types de workloads

Les workloads dans Kubernetes peuvent être de différents types. Nous avons déjà rencontré plusieurs workloads classiques, tels que les serveurs web, applications, ou bases de données.\
Ces workloads ont pour caractéristique principale de fonctionner **en continu**, tant qu’ils ne sont pas arrêtés manuellement.

Cependant, certains workloads sont conçus pour exécuter des tâches spécifiques et finir leur exécution une fois la tâche accomplie. 
Par exemple :

- Effectuer un calcul mathématique,
- Traiter une image,
- Réaliser de l’analyse sur un grand jeu de données,
- Générer un rapport puis envoyer un email.

Ces workloads ont une durée de vie **limitée** : ils démarrent, accomplissent une ou plusieurs tâches, puis s’arrêtent.

### Exemple avec Docker

Prenons un exemple simple avec Docker pour illustrer un workload ponctuel :

```
$ docker run busybox sh -c "expr 1 + 2"
```

Le container démarrera, réalisera l’opération mathématique (addition de 1 et 2), affichera le résultat puis s’arrêtera.

Si vous exécutez ensuite :

```
$ docker ps -a
```

Vous verrez que le container est dans l’état **exited**, avec un code de sortie `0` indiquant que la tâche s’est bien terminée.

### Transposition avec Kubernetes

On peut reproduire ce même comportement avec Kubernetes en créant un **Pod** qui réalise le calcul.

Voici un exemple simple de définition de Pod (`pod.yaml`) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: math
      image: busybox
      command: ["sh", "-c", "expr 1 + 2"]
  restartPolicy: OnFailure
```

Lorsque ce Pod est créé (`kubectl apply -f pod.yaml`), il lance le container, exécute l’opération, puis termine son exécution. On voit alors le Pod passer en état **Completed**.

#### Comportement par défaut

Si la `restartPolicy` n’est pas modifiée (valeur par défaut : `Always`), Kubernetes tentera de redémarrer ce container une fois terminé, avec l’objectif implicite que les workloads tournent **en continu**.

Pour empêcher ce redémarrage dans le cas d’un Job, on utilise donc `restartPolicy: Never` ou `OnFailure`.

***

## Gestion des Jobs dans Kubernetes

Pour automatiser la gestion de ces workloads ponctuels et multiples, Kubernetes propose la ressource **Job**.

### Comment fonctionne un Job ?

Un **Job** permet de créer et de gérer un ou plusieurs Pods pour exécuter une tâche spécifique jusqu’à sa complétion. Kubernetes s’assure que le nombre de Pods nécessaires réalise la tâche avec succès.

- Si un **Pod échoue**, Kubernetes en crée un nouveau pour reprendre la tâche.
- Une fois que tous les Pods ont terminé avec succès, le **Job est terminé**.

### Exemple de définition d’un Job simple (`job.yaml`)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  completions: 1
  template:
    spec:
      containers:
      - name: math
        image: busybox
        command: ["sh", "-c", "expr 1 + 2"]
      restartPolicy: Never
```

- Ici, `completions: 1` signifie que le Job doit réussir **une fois**.
- La propriété `restartPolicy: Never` indique que Kubernetes ne redémarrera pas un Pod qui a déjà terminé.

Pour créer ce Job :

```
$ kubectl apply -f job.yaml
```

Vous pouvez voir l’état du Job avec :

```
$ kubectl get jobs
```

Pour afficher la sortie du Pod (le résultat de l’addition) :

```
$ kubectl logs <pod-name>
```

Enfin, pour supprimer le Job et les Pods associés :

```
$ kubectl delete job math-add-job
```

### Exécuter plusieurs Pods avec un Job

Pour exécuter plusieurs Pods en séquentiel ou parallèle, vous pouvez spécifier :

- `completions: n` pour le nombre total de Pods à terminer avec succès,
- `parallelism: m` pour le nombre de Pods que Kubernetes peut exécuter en parallèle.

Exemple :

```yaml
spec:
  completions: 3
  parallelism: 2
```

Cela crée jusqu’à 2 Pods en même temps, jusqu’à ce que 3 Pods aient terminé avec succès.

***

### Différence entre Job et ReplicaSet

- Un **ReplicaSet** maintient un nombre fixe de Pods en fonctionnement **en continu** pour assurer la disponibilité de l’application (par exemple 3 serveurs web toujours actifs).
- Un **Job** exécute un ou plusieurs Pods destinés à faire une tâche ponctuelle et à s’arrêter une fois terminée.

***

### backoffLimit

Voici un exemple YAML de Job Kubernetes où la propriété `backoffLimit` est essentielle pour contrôler le nombre de tentatives de relance en cas d’échec, afin d’éviter des relances infinies nuisibles au cluster et aux ressources.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: example-backofflimit-job
spec:
  backoffLimit: 3           # Kubernetes réessaiera 3 fois au maximum en cas d’échec
  template:
    spec:
      containers:
      - name: fail-test
        image: busybox
        command: ["sh", "-c", "exit 1"]  # Ce container échoue toujours volontairement
      restartPolicy: Never
```

### Explications

- Le container retourne un code de sortie `1`, ce qui est une erreur.
- Kubernetes lancera initialement un Pod, qui échouera.
- Le Job créera automatiquement **jusqu’à 3 nouveaux Pods** (nombre défini par `backoffLimit`) pour retenter la tâche.
- Après 3 échecs, le Job est marqué comme **Failed** et aucun nouveau Pod ne sera créé.
- Cela évite des boucles infinies de Pods échoués et l’utilisation excessive des ressources.

Voici comment vérifier l’état du Job :

```
kubectl get jobs example-backofflimit-job
kubectl describe job example-backofflimit-job
kubectl get pods -l job-name=example-backofflimit-job
```

***

## Résumé concis

- Un **Job** permet d’exécuter des Pods pour des tâches ponctuelles, garantissant qu’ils s’exécutent jusqu’à complétion.
- La propriété `restartPolicy` doit être réglée sur `Never` ou `OnFailure` pour que Kubernetes ne redémarre pas un Pod déjà terminé.
- Le Job gère la création et la supervision des Pods nécessaires pour exécuter la tâche avec succès.
- Un **ReplicaSet** diffère en ce qu’il assure la disponibilité d’un nombre constant de Pods actifs en continu.
- Le Job est utilisé pour des traitements batch, alors que le ReplicaSet pour des applications toujours en fonctionnement.
