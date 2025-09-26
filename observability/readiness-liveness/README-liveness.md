## Liveness probes

Dans cette partie, nous allons découvrir les **liveness probes**, un mécanisme essentiel pour garantir la bonne santé de vos applications dans Kubernetes.  

### Comprendre le problème
Commençons par une situation simple.  
Vous lancez une image NGINX avec **Docker**. Le serveur démarre correctement et sert les utilisateurs.  
Mais supposons que, pour une raison quelconque, le processus NGINX plante. Le container **s’arrête** aussi.  
En exécutant `docker ps`, vous constatez que le container est mort.  

Problème : **Docker n’est pas un outil d’orchestration**. Il n’essaie pas de réparer la situation.\
L’application reste inaccessible jusqu’à ce que vous recréiez manuellement le container.  

C’est là qu’intervient **Kubernetes**.  
Si vous exécutez cette même application dans un Pod géré par Kubernetes, alors à chaque plantage, le **kubelet** tente automatiquement de redémarrer le container.\
Vous pouvez le constater dans la commande :  
```
$ kubectl get pods
```
où la colonne *RESTARTS* affiche le nombre de redémarrages effectués.  

Ce mécanisme fonctionne bien **tant que l’application s’arrête réellement**.  

***

### Cas plus subtil : application bloquée
Imaginons maintenant un autre scénario.  
L’application continue de tourner, mais un bug dans le code la bloque dans une boucle infinie.  
- Pour Kubernetes : le container est toujours en vie → il est considéré comme « up ».  
- Pour l’utilisateur : l’application ne répond plus aux requêtes → service indisponible.  

Dans ce cas, Kubernetes **ne peut pas savoir** qu’il faut redémarrer le container, puisque celui-ci ne s’est pas arrêté.\
Résultat : les utilisateurs subissent une panne silencieuse.  

***

### La solution : les liveness probes
C’est ici qu’interviennent les **liveness probes**.  
Une liveness probe est un test configuré sur un container afin de vérifier périodiquement si l’application est toujours **saine** (*healthy*).  

- Si le test réussit → Kubernetes considère le container comme sain et le laisse tourner.  
- Si le test échoue → le container est jugé **unhealthy**, il est détruit, puis remplacé par un nouveau.  

Ainsi, même dans le cas où l’application tourne « en apparence » mais ne fonctionne pas correctement, Kubernetes peut réagir et restaurer le service.  

***

### Définir une liveness probe
Comme pour une readiness probe, c’est au **développeur** de définir ce que signifie une application « en bonne santé ». Les options sont similaires :  

- **HTTP GET** : envoyer une requête vers un endpoint (ex. `/healthz`) et vérifier s’il répond.  
- **TCP Socket** : tester si un port est réellement à l’écoute (utile pour une base de données).  
- **Exec command** : exécuter une commande ou un script dans le container, qui doit retourner un succès (code 0).  

***

### Exemple de configuration
Voici un exemple simple dans un fichier de définition de Pod :  
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

***

### Paramètres supplémentaires
La liveness probe possède différents paramètres de configuration, par exemple :  
- **initialDelaySeconds** : délai avant d’exécuter la première vérification.  
- **periodSeconds** : fréquence des tests.  
- **successThreshold / failureThreshold** : nombre de réussites ou d’échecs nécessaires avant de changer l’état du container.  

***

### Pourquoi c’est essentiel
- La **readiness probe** sert à dire si l’application est prête à recevoir du trafic.  
- La **liveness probe** sert à dire si l’application continue de fonctionner correctement au fil du temps.  

En combinant les deux, Kubernetes peut garantir à la fois :  
- que seul un Pod prêt reçoit du trafic,  
- et qu’un Pod bloqué ou en panne est automatiquement relancé.  

***

## Résumé concis
- Avec **Docker**, un container planté reste arrêté, et il faut le recréer manuellement.  
- Avec **Kubernetes**, un container qui **s’arrête** est automatiquement redémarré.  
- Problème : si l’application tourne mais est bloquée (ex. boucle infinie), Kubernetes croit qu’elle est saine.  
- Solution : on configure une **liveness probe** pour tester régulièrement l’état réel de l’application.  
- Types de probes possibles : **HTTP GET**, **TCP Socket**, **Exec command**.  
- Paramètres personnalisables : `initialDelaySeconds`, `periodSeconds`, `successThreshold`, `failureThreshold`.  
- Utilité : Kubernetes redémarre automatiquement un container jugé **unhealthy**, assurant la continuité du service.  


