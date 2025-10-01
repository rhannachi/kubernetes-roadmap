## CronJobs dans Kubernetes

### Qu'est-ce qu'un CronJob ?

Un **CronJob** dans Kubernetes est un objet qui permet de **planifier l'exécution périodique** de tâches, très similaire à la commande `crontab` sous Linux. Contrairement à un **Job** qui s’exécute une seule fois, un CronJob lance des Jobs **à des intervalles réguliers** suivant une expression cron.

Par exemple, si vous avez un Job qui génère un rapport et envoie un email, un CronJob vous permettra de planifier ce Job pour qu’il s’exécute automatiquement tous les jours, toutes les heures, ou à toute autre fréquence souhaitée.

***

### Construction d’un CronJob

Pour définir un CronJob, on crée un fichier YAML suivant ce modèle :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: reporting-cronjob
spec:
  schedule: "0 8 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report-generator
            image: busybox
            command: ["sh", "-c", "echo ‘Génération du rapport’"]
          restartPolicy: Never
```

**Explications :**

- `apiVersion: batch/v1` : version actuelle de l’API pour les CronJobs.
- `kind: CronJob` : spécifie qu’il s’agit d’un objet CronJob.
- `metadata.name` : nom donné à ce CronJob, ici `reporting-cronjob`.
- `spec.schedule` : la planification en format **cron**. Ici, le Job s'exécutera tous les jours à 8h00.
- `spec.jobTemplate` : contient la définition du **Job** qui sera lancé à chaque occurrence.
- À l’intérieur du template, on retrouve la définition classique d’un Pod (containers, images, commandes).
- `restartPolicy: Never` indique que lorsque le Pod termine, il ne doit pas être redémarré.

Il est important de noter qu'on a désormais trois niveaux de spécifications dans un cronjob : celui du CronJob lui-même, du Job généré, puis du Pod.

***

### Commandes pratiques

Pour créer un CronJob :

```
$ kubectl apply -f cronjob.yaml
```

Pour vérifier les CronJobs :

```
$ kubectl get cronjobs
```

Pour voir les Jobs créés par le CronJob et leurs états :

```
$ kubectl get jobs
```

Pour afficher les Pods et leurs logs :

```
$ kubectl get pods
$ kubectl logs <pod-name>
```

Pour supprimer un CronJob ainsi que tous les Jobs correspondants :

```
$ kubectl delete cronjob reporting-cronjob
```

***

### Remarques importantes

- La planification suit la syntaxe standard de cron (5 champs : minute, heure, jour du mois, mois, jour de la semaine).
- Les CronJobs génèrent un **Job** chaque fois que le planning est déclenché. Chaque Job gère un ou plusieurs Pods jusqu’à complétion.
- Il est possible de régler le niveau de parallélisme et la gestion des chevauchements (s’assurer qu’un Job ne démarre pas avant la fin du précédent).
- On peut limiter le nombre d’historiques de Jobs conservés (réussis ou échoués) dans Kubernetes pour éviter la saturation des ressources.

## Résumé concis

- Un **CronJob** est une ressource Kubernetes qui planifie l’exécution répétée de **Jobs** selon une expression **cron**.
- Il crée un **Job** à chaque exécution planifiée, et ce Job lance un ou plusieurs **Pods** qui réalisent la tâche souhaitée.
- La définition d’un CronJob contient des sections pour le CronJob, le Job, et le Pod, avec des spécifications imbriquées.
- La planification se fait via la propriété `schedule` au format cron (minutes, heures, jours, etc.).
- Utile pour automatiser des tâches récurrentes comme sauvegardes, rapports, nettoyage, synchronisation.
- On gère l’historique des Jobs pour éviter l’accumulation et on peut contrôler la concurrence des exécutions.
