### Partie 1 : Déploiement de l’architecture

Crée les ressources suivantes dans ton cluster Kubernetes :

- **Trois Pods** :
    - Un pod de type web server (`web-server`) qui utilise l’image `curlimages/curl:latest`.
    - Un pod de type API server (`api-server`) utilisant également l’image `curlimages/curl:latest`.
    - Un pod de type database server (`database-server`) avec l’image `hashicorp/http-echo:0.2.3` qui expose le port 5678.

- **Un Service ClusterIP** appelé `db-service`, qui expose le port 3000 (redirigé vers le 5678 du pod database) et cible le pod avec le label `role: db`.

``` 
+----- POD ------+       +---- Service ---+       +------- POD ----------+
|                |       |                |       |                      |
|   api-server   |       |   db-service   |       |  database-server     |
|  (role=api)    |       |                |       |    (role=db)         |
|                |------>|  port: 3000    |       |                      |
|                |       | targetPort:5678|------>| port container: 5678 |
|                |       |                |       |                      |
+----------------+       +----------------+       +----------------------+
        |                        |                         ^
        |  Requête HTTP vers     |                         |
        |  http://db-service:3000|                         |
        |                        |                         |
        |                    Redirection                   |
        |                    du port 3000                  |
        |                    vers port 5678                |
        |                        |                         |
        |                        |                         |
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  labels:
    role: web
spec:
  containers:
    - name: web-container
      image: curlimages/curl:latest
      command: ["sh", "-c", "while true; do curl -s http://db-service:3000/ && echo OK || echo KO; sleep 5; done"]
---
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  labels:
    role: api
spec:
  containers:
    - name: api-container
      image: curlimages/curl:latest
      command: ["sh", "-c", "while true; do curl -s http://db-service:3000/ && echo OK || echo KO; sleep 5; done"]
---
apiVersion: v1
kind: Pod
metadata:
  name: database-server
  labels:
    role: db
spec:
  containers:
    - name: db-container
      image: hashicorp/http-echo:0.2.3
      args:
        - "-text=OK"
      ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  selector:
    role: db
  ports:
    - protocol: TCP
      port: 3000       
      targetPort: 5678 # Aligné avec containerPort de database-server
  type: ClusterIP
```

```
$ kubectl apply -f app.yaml
```

#### Tester la communication entre `api-server` et `web-server` vers `database-server`

```bash
kubectl logs api-server
OK
```
```bash
kubectl logs web-server
OK
```

**Conclusion :** les deux pods (`api-server` et `web-server`) réussissent à communiquer avec le pod `database-server` via le service `db-service`.

### Partie 2 : Sécurisation avec une NetworkPolicy

``` 
$ minikube start --cni=calico

$ kubectl get pods -n kube-system -l k8s-app=calico-node
NAME                READY   STATUS    RESTARTS        AGE
calico-node-6llrr   1/1     Running   1 (4m49s ago)   8m52s
calico-node-gxvb2   1/1     Running   1 (4m50s ago)   8m52s
calico-node-ng8h4   1/1     Running   1 (4m47s ago)   8m52s
calico-node-z6lnx   1/1     Running   1 (4m56s ago)   8m52s
```

Ajoute une NetworkPolicy pour restreindre l’accès à la base de données :

- Seuls les pods avec le label `role: api` doivent pouvoir communiquer avec le pod de base de données, sur le port TCP 3000 via le Service.
- Les autres pods (notamment `web-server`) doivent être bloqués.

``` 
+----- POD ------+       +---- Service ---+       +------- NetworkPolicy ------+
|                |       |                |       |                            |
|   api-server   |       |   db-service   |       |     NetworkPolicy          |
|  (role=api)    |       |                |       |   (pour pods role=db)      |
|                |------>|  port: 3000    |       |  +------ POD ----------+|  |
|                |       | targetPort:5678|------>|  |  database-server     |  |
|                |       |                |       |  |    role=db           |  |
+----------------+       +----------------+       |  | port container: 5678 |  |
        |                        |                |  +---------------------+   |
        |  Requête HTTP vers     |                 +---------------------------+
        |  http://db-service:3000|                           ^
        |                        |                           |
        |                    Redirection                     |
        |                    port 3000                       |
        |                    vers port 5678                  |
        |                        |                           |
        +------------------------+---------------------------+

(Note : La NetworkPolicy agit au niveau du Pod database-server en filtrant le trafic entrant uniquement vers le port 5678, et autorise uniquement les Pod avec label role=api)
```

### Compléments d'explication :

- La requête part du pod api-server vers le service db-service sur le port 3000.
- Le service fait la redirection vers le port 5678 du pod database-server.
- La NetworkPolicy sélectionne les pods role=db (ici database-server) et autorise seulement le trafic entrant sur le port 5678 depuis les pods role=api.
- Toute autre connexion vers le pod database-server est bloquée (Ingress contrôle).
- Ce contrôle au niveau pod sur le port 5678 est la raison pour laquelle la NetworkPolicy doit s'aligner sur le port réel d'écoute du pod.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: api   # Seuls les pods role=api sont autorisés
      ports:
        - protocol: TCP
          # autoriser du port "5678", et non pas le service 3000 !
          port: 5678 # Le Port du NetworkPolicy s’appliquent sur les ports du pod (c'est-à-dire containerPort), pas sur ceux du Service !!!!!
```

Grâce à cette règle, le pod `web-server` n’aura plus accès au service de base de données, tandis que le pod `api-server` conservera l’accès.

```
$ kubectl logs api-server
OK
```
```
$ kubectl logs web-server
KO
```






