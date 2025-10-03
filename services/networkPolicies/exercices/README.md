### Partie 1 : Déploiement de l’architecture

Crée les ressources suivantes dans ton cluster Kubernetes :

- **Trois Pods** :
    - Un pod de type web server (`web-server`) qui utilise l’image `curlimages/curl:latest`.
    - Un pod de type API server (`api-server`) utilisant également l’image `curlimages/curl:latest`.
    - Un pod de type database server (`database-server`) avec l’image `hashicorp/http-echo:0.2.3` qui expose le port 5678.

- **Un Service ClusterIP** appelé `db-service`, qui expose le port 3306 (redirigé vers le 5678 du pod database) et cible le pod avec le label `role: db`.

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
      command: ["sh", "-c", "while true; do curl -s http://db-service:5678/ && echo OK || echo KO; sleep 5; done"]
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
      command: ["sh", "-c", "while true; do curl -s http://db-service:5678/ && echo OK || echo KO; sleep 5; done"]
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
      port: 5678       # Aligné avec containerPort de database-server
      targetPort: 5678
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

- Seuls les pods avec le label `role: api` doivent pouvoir communiquer avec le pod de base de données, sur le port TCP 3306 via le Service.
- Les autres pods (notamment `web-server`) doivent être bloqués.

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
          port: 5678

```

Grâce à cette règle, le pod `web-server` n’aura plus accès au service de base de données, tandis que le pod `api-server` conservera l’accès.

```bash
kubectl logs api-server
OK
```
```bash
kubectl logs web-server
KO
```






