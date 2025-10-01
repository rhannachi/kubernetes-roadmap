# Comprendre CronJob → Job → Pod en Kubernetes

Lorsque tu crées un CronJob en Kubernetes, tu définis une tâche planifiée. Cette ressource est construite hiérarchiquement :

1. Le **CronJob** définit le planning et contient un modèle de **Job**.
2. Chaque fois que le planning est atteint, le CronJob crée un **Job**.
3. Le **Job** exécute un ou plusieurs **Pods**, basés sur le PodTemplate défini.
4. Les **Pods** sont les unités réelles d’exécution, où les containers tournent.

***

```yaml
#### POD ####
apiVersion: v1
kind: Pod
metadata:
  name: my-pod             # Exemple de Pod (instancié par le Job)
spec:
  containers:              # Le Pod exécute réellement le container BusyBox
    - name: math
      image: busybox
      command: ["sh", "-c", "expr 1 + 2"]
  restartPolicy: OnFailure
---
#### JOB ####
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job             # Exemple de Job (créé automatiquement par le CronJob)
spec:
  completions: 4
  parallelism: 2
  template:                # Le Job encapsule le PodTemplate
    spec:
      containers:
        - name: math
          image: busybox
          command: ["sh", "-c", "expr 1 + 2"]
      restartPolicy: OnFailure
```

#### cronjob.yaml

```yaml
#### CronJob ####
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/2 * * * *"  # Le job sera lancé toutes les 2 minutes
  jobTemplate:             # Le CronJob encapsule un Job
    spec:
      completions: 4        # Le Job attend 4 Pods terminés avec succès
      parallelism: 2        # 2 Pods peuvent tourner en parallèle
      template:             # Le Job contient un PodTemplate
        spec:
          containers:       # Chaque Pod exécute ce container
            - name: math
              image: busybox
              command: ["sh", "-c", "expr 1 + 2"]
          restartPolicy: OnFailure
```
***

#### Schéma hiérarchique

Voici une représentation sous forme d’arborescence montrant l’encapsulation :

```
CronJob (my-cronjob)
 └── Job (my-job) [créé automatiquement chaque 2 minutes]
      └── Pod (my-pod-xxxxx) [un ou plusieurs créés pour exécuter la tâche]
           └── Container (math) [image: busybox]
```

### Résultat de la planification du CronJob

Lorsque tu définis un CronJob avec `schedule: "*/2 * * * *"`, Kubernetes crée automatiquement un Job toutes les 2 minutes.  
À chaque nouvelle échéance, le système génère un nouveau Job, qui lui-même va lancer ses propres Pods pour exécuter la tâche spécifiée.

***

### Explication de chaque commande

#### 1. Liste des CronJobs

```shell
$ kubectl get cronjob
NAME         SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-cronjob   */2 * * * *   <none>     False     0        <none>          28s
```
- Ce résultat montre qu’un CronJob nommé `my-cronjob` est programmé pour s’exécuter toutes les 2 minutes.
- La colonne **ACTIVE** indique le nombre de Jobs actuellement actifs (ici, aucun juste après la création).
- La colonne **LAST SCHEDULE** reste vide car le CronJob vient d’être créé et n’a pas encore lancé de Job.

***

#### 2. Liste des Jobs et Pods juste après création

```shell
$ kubectl get job
NAME                  STATUS     COMPLETIONS   DURATION   AGE
my-cronjob-29321788   Complete   4/4      

$ kubectl get pod
NAME                        READY   STATUS      RESTARTS   AGE
my-cronjob-29321788-76ssf   0/1     Completed   0          89s
my-cronjob-29321788-gbc5d   0/1     Completed   0          89s
my-cronjob-29321788-l6fdw   0/1     Completed   0          84s
my-cronjob-29321788-mnkj7   0/1     Completed   0          85s
```
- Le Job nommé `my-cronjob-29321788` a été créé et a terminé l’exécution de 4 Pods (voir colonne COMPLETIONS).
- Les Pods associés montrent tous le statut `Completed`, ce qui signifie qu’ils ont fini leur tâche avec succès et n’ont pas eu besoin d’être relancés (RESTARTS = 0).
- L’âge des Pods (84-89 secondes) correspond juste après l’exécution planifiée par le CronJob.

***

#### 3. Résultat après deux minutes (une nouvelle échéance)

```shell
$ kubectl get job
NAME                  STATUS     COMPLETIONS   DURATION   AGE
my-cronjob-29321788   Complete   4/4           9s         2m26s
my-cronjob-29321790   Complete   4/4           9s         26s

$ kubectl get pod
NAME                        READY   STATUS      RESTARTS   AGE
my-cronjob-29321788-76ssf   0/1     Completed   0          2m55s
my-cronjob-29321788-gbc5d   0/1     Completed   0          2m55s
my-cronjob-29321788-l6fdw   0/1     Completed   0          2m50s
my-cronjob-29321788-mnkj7   0/1     Completed   0          2m51s
my-cronjob-29321790-6gxrk   0/1     Completed   0          55s
my-cronjob-29321790-kmbxf   0/1     Completed   0          51s
my-cronjob-29321790-q2dqc   0/1     Completed   0          50s
my-cronjob-29321790-z7fwf   0/1     Completed   0          55s
```
- Après 2 minutes, un second Job `my-cronjob-29321790` est créé automatiquement par le CronJob.
- Chacun de ces Jobs (premier et second) comporte 4 Pods, ce qui correspond aux spécifications (`completions: 4`).
- Les Pods du second Job ont tous le statut `Completed` et un âge d’environ 50-55 secondes.
- Ce cycle continuera à chaque échéance du CronJob, créant un nouveau Job puis 4 nouveaux Pods à chaque fois.

***

### Explications supplémentaires et concepts clés

- **Encapsulation hiérarchique** :
    - Le **CronJob** gère la planification.
    - À chaque échéance, il génère un nouvel objet **Job**.
    - Le **Job** gère l’exécution, la reprise en cas d’échec, le parallélisme et le nombre de Pods créés.
    - Les **Pods** sont lancés pour exécuter la commande voulue.

- **Cycle d’exécution** :
    - À chaque intervalle de deux minutes, un nouveau Job est instancié par le CronJob.
    - Chaque Job lance 4 Pods pour exécuter la tâche demandée (ici un calcul simple), avec un maximum de 2 Pods lancés simultanément (`parallelism: 2`).
    - Une fois les Pods terminés, le Job passe en statut `Complete`. Le CronJob attend la prochaine échéance.

