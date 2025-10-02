
# Exercice : Déploiement d’une architecture multi-tier Kubernetes avec Pods, Services et Nodes

## Objectif

Mettre en place dans un cluster Kubernetes une architecture simple reposant sur 3 types de Pods (front-end, back-end et base de données simulée)\
déployés sur différentes nodes selon leur rôle, et exposer ces Pods via différents types de Services (ClusterIP, NodePort, LoadBalancer).

## Contexte

Vous disposez d’un cluster Kubernetes composé de plusieurs nodes étiquetés par rôle : front, back, db. Vous allez déployer trois Pods, chacun affecté selon leur rôle à un node spécifique via `nodeSelector`.
- `front-pod` : pod front-end, conteneur Alpine avec un serveur HTTP simple (netcat sur port 80)
- `back-pod` : pod back-end, conteneur Alpine avec serveur HTTP (netcat sur port 8080)
- `db-pod` : pod base de données simulée, serveur HTTP (netcat sur port 9090)

Ensuite, vous exposerez ces Pods via des Services Kubernetes adaptés à chaque besoin :
- `back-service` de type **ClusterIP** exposant `back-pod` sur le port 80 (redirige vers 8080 du Pod)
- `db-service` de type **NodePort** exposant `db-pod` sur le port 9090, accessible via `nodePort` 31090
- `front-service` de type **LoadBalancer** exposant `front-pod` sur le port 80

- Tester la communication :
    - Accéder au back-end via le service clusterIP : `curl http://back-service:80` depuis un Pod du cluster.
    - Accéder au db-service via le NodePort depuis l’extérieur : `curl http://$(minikube ip):31090`.
    - Accéder au front-service via le load balancer (minikube tunnel) : `curl $(minikube service front-service --url)`.

***

# Solution :

```
+--------------------------------------------------------+
|                       Kubernetes Cluster               |
|                                                        |
|  +----------------+    +----------------+    +----------------+    Node (role: front)   
|  |   front-pod    |    |   back-pod     |    |    db-pod      |    Node (role: back)    
|  | container:     |    | container:     |    | container:     |    Node (role: db)      
|  | alpine:latest  |    | alpine:latest  |    | alpine:latest  |                         
|  |                |    | (nc :8080 http)|    | (nc :9090 http)|                         
|  +-------+--------+    +--------+-------+    +---------+------+                         
|          |                      |                      |                                
|          |                      |                      |                                
|          |                      |                      |                                
|          |                      |                      |
|          |                 (ClusterIP)                 |         
|          |                 back-service                |                                
|          |       port 80 => targetPort (port) 8080     |
|          |           curl http://back-service          |
|          |                                             |
|          |                                          (NodePort)                           
|          |                                          db-service 
|          |                            nodePort 31090  => targetPort (pod) 9090
|          |                               curl http://$(minikube ip):31090
|          |
|   (LoadBalancer)                                                                                  
|   front-service                           
|   port 80 => targetPort (pod) 80
|   curl $(minikube service front-service --url)
|                                                                                          
+--------------------------------------------------------+
```

```yaml
---
# Pod Front-end (alpine avec curl, pod passif pour test)
apiVersion: v1
kind: Pod
metadata:
  name: front-pod
  labels:
    app: front
spec:
  nodeSelector:
    role: front
  containers:
    - name: front-container
      image: alpine:latest
      command:
        - sh
        - -c
        - |
          apk add --no-cache netcat-openbsd && \
          while true; do echo -e "HTTP/1.1 200 OK\n\nHello from Front-end" | nc -l -p 80; done

---
# Pod Back-end avec serveur HTTP simple (alpine avec nc)
apiVersion: v1
kind: Pod
metadata:
  name: back-pod
  labels:
    app: back
spec:
  nodeSelector:
    role: back
  containers:
    - name: back-container
      image: alpine:latest
      command:
        - sh
        - -c
        - |
          apk add --no-cache netcat-openbsd && \
          while true; do echo -e "HTTP/1.1 200 OK\n\nHello from Back-end" | nc -l -p 8080; done

---
# Pod Base de données simulée avec serveur HTTP (alpine avec nc)
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  labels:
    app: db
spec:
  nodeSelector:
    role: db
  containers:
    - name: db-container
      image: alpine:latest
      command:
        - sh
        - -c
        - |
          apk add --no-cache netcat-openbsd && \
          while true; do echo -e "HTTP/1.1 200 OK\n\nHello from DB" | nc -l -p 9090; done

---
# Service ClusterIP exposant Back-end (port 8080 -> 80)
apiVersion: v1
kind: Service
metadata:
  name: back-service
spec:
  type: ClusterIP
  selector:
    app: back
  # 80 (port) => 8080 (targetPort)
  ports:
    - port: 80
      targetPort: 8080
---
# Service NodePort exposant la DB (port 9090, nodePort 31090)
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  type: NodePort
  selector:
    app: db
  # 31090 => 9090 (port) => 9090 (targetPort) 
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 31090
---
# Service LoadBalancer exposant Front (port 80)
apiVersion: v1
kind: Service
metadata:
  name: front-service
spec:
  type: LoadBalancer
  selector:
    app: front
  # 80 (port) => 80 (targetPort)
  ports:
    - port: 80
      targetPort: 80
```

***

# Test des Services Kubernetes (ClusterIP, NodePort, LoadBalancer) sur Minikube

Ce guide explique comment tester que chacun des services exposés dans l'exemple YAML fonctionne correctement dans un cluster Minikube multi-nœuds.

## Préparation du cluster Minikube

Lancer un environnement Minikube avec 3 nœuds workers :
```bash
minikube start --driver=docker --nodes 4
kubectl get nodes
```
**Résultat attendu :**

| NAME        | STATUS | ROLES          | AGE | VERSION |
|-------------|--------|----------------|-----|---------|
| minikube    | Ready  | control-plane  | 17m | v1.33.1 |
| minikube-m02| Ready  | <none>         | 16m | v1.33.1 |
| minikube-m03| Ready  | <none>         | 16m | v1.33.1 |
| minikube-m04| Ready  | <none>         | 16m | v1.33.1 |

*Grade:* Validé si tous les nœuds sont `Ready` sans erreur.

***

## Labelisation des nœuds

Assigner un label unique à chaque nœud pour forcer le scheduling selon `nodeSelector` :
```bash
kubectl label node minikube-m02 role=front
kubectl label node minikube-m03 role=back
kubectl label node minikube-m04 role=db
```
*Grade:* Validé si la commande termine sans erreurs, labels visibles via `kubectl get nodes --show-labels`.

***

## Déploiement des pods et services

Appliquer le manifeste YAML des pods et services :
```bash
kubectl apply -f deployment.yaml
```
Verifier que les pods sont programmés sur les nœuds assignés :
```bash
kubectl get pod -o wide
```
**Résultat attendu :**

| NAME      | READY | STATUS  | NODE         | IP          |
|-----------|-------|---------|--------------|-------------|
| front-pod | 1/1   | Running | minikube-m02 | 10.244.1.x  |
| back-pod  | 1/1   | Running | minikube-m03 | 10.244.2.x  |
| db-pod    | 1/1   | Running | minikube-m04 | 10.244.3.x  |

*Grade:* Validé si chaque pod tourne bien sur le nœud labelisé correspondant.

***

## 1. Test du service ClusterIP (`back-service`)

Tester l’accès au pod `back-pod` via le service interne ClusterIP depuis `front-pod` :
```bash
kubectl exec -it front-pod -- sh
apk add curl
curl http://back-service
```
**Résultat attendu:**
```
Hello from Back-end
```

Tester aussi depuis `db-pod` idem pour valider la communication interne.

*Grade:*

- **Excellent** : Réponse correcte reçue sur tous les pods testés.
- **Moyen** : Pas de réponse depuis certains pods, vérifier réseau du cluster.
- **Échec** : Pas de réponse, vérifier que le service ClusterIP est bien créé et que le pod back-pod est sain.

*Conclusion:* Le service ClusterIP assure une communication stable et privée à l’intérieur du cluster.

***

## 2. Test du service NodePort (`db-service`)

Lister les services pour obtenir le port NodePort exposé :
```bash
kubectl get svc db-service
```
**Exemple sortie:**
```
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
db-service   NodePort   10.96.251.12    <none>        9090:31090/TCP   35m
```

Tester l’accès externe via Minikube :
```bash
minikube service db-service --url
curl http://$(minikube ip):31090
```
**Résultat attendu :**
```
Hello from DB
```

*Grade :*

- **Excellent** : Réponse correcte sur l’URL retournée par minikube, le pod est joignable depuis l’extérieur.
- **Moyen** : Réponse intermittente, vérifier la connectivité réseau/port.
- **Échec** : Pas de réponse, assembler pod/node/port ou configuration réseau.

*Conclusion :* Le NodePort expose correctement le pod à l’extérieur via un port fixe.

***

## 3. Test du service LoadBalancer (`front-service`)

Note importante : Minikube ne fournit pas un vrai LoadBalancer cloud, mais simule ce type.

Lister le service front-service :
```bash
kubectl get svc front-service
```
**Sortie attendue:**
```
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
front-service  LoadBalancer   10.102.195.171  <pending>     80:<port>/TCP  35m
```

Tester grâce à la commande Minikube (qui crée un tunnel vers le service LoadBalancer simulé) :
```bash
minikube service front-service --url
curl $(minikube service front-service --url)
```
**Résultat attendu :**
```
Hello from Front-end
```
Tu peux vérifier que la commande `minikube service front-service` ouvre ou affiche bien l’URL sans erreur.

*Grade :*

- **Excellent** : URL accessible sans erreur, les connexions sont bien redirigées.
- **Moyen** : L'URL s'affiche, mais la connexion ne répond pas (pod passif).
- **Échec** : Problème dans le tunnel ou service non joignable.

*Conclusion :* Le service LoadBalancer est simulé correctement en local par Minikube pour l’exposition externe.

***

# Résumé des résultats attendus

| ServiceType | Ressource testée   | Test                   | Résultat attendu           | Note d’évaluation                |
|-------------|--------------------|------------------------|---------------------------|---------------------------------|
| ClusterIP   | back-service       | curl depuis front-pod   | "Hello from Back-end"     | Excellent si réponse correcte    |
| NodePort    | db-service         | curl via minikube URL   | "Hello from DB"           | Excellent si accessible externe  |
| LoadBalancer| front-service      | minikube service + curl | Accès URL sans erreur     | Excellent si URL accessible      |


***

### Analyse de la commande `kubectl describe service back-service`

``` 
$ kubectl describe service back-service
Name:                     back-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=back
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.98.74.102
IPs:                      10.98.74.102
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
Endpoints:                10.244.2.4:8080
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>

$ kubectl get pod -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
back-pod    1/1     Running   0          58s   10.244.2.4   minikube-m03   <none>           <none>
db-pod      1/1     Running   0          58s   10.244.3.5   minikube-m04   <none>           <none>
front-pod   1/1     Running   0          59s   10.244.1.5   minikube-m02   <none>           <none>
```

- Le service `back-service` est de type **ClusterIP** avec une adresse IP interne : `10.98.74.102`.
- Il écoute sur le port 80 et redirige vers le `targetPort` 8080 des pods ciblés.
- Le sélecteur du service est `app=back`, ce qui signifie qu'il cible les Pods ayant ce label.
- Sous la ligne **Endpoints**, on voit `10.244.2.4:8080`.

### Analyse de la commande `kubectl get pod -o wide`

- Le pod `back-pod` a l'IP `10.244.2.4` et tourne sur le node `minikube-m03`.
- Les autres pods (db-pod, front-pod) ont des IP différentes, mais ne sont pas ciblés par ce service, car ils ont d'autres labels (`app=db`, `app=front`).

***

### Signification des Endpoints en Kubernetes

- L'**Endpoint** dans le contexte d'un Service Kubernetes est l'adresse IP (et port) des Pods derrière ce service.
- Kubernetes crée automatiquement une ressource appelée **Endpoints** (ou **EndpointSlices** dans les versions récentes), qui liste les IPs et ports des Pods qui correspondent au `selector` du Service.
- Ici, `back-service` pointe vers l'adresse `10.244.2.4:8080`, qui est l’IP et port du pod `back-pod`.
- Quand un client envoie une requête à `back-service` (via l’IP 10.98.74.102:80), Kubernetes, avec l’aide de `kube-proxy`, redirige ce trafic vers l’un des endpoints réels (ici 10.244.2.4:8080).

***

### En résumé

| Élément           | Description                                                 |
|-------------------|-------------------------------------------------------------|
| Service back-service | Objet stable avec IP virtuelle (ClusterIP) exposant le backend |
| Selector          | Filtre qui associe le service aux pods avec label `app=back` |
| Endpoints         | Liste des IPs et ports des Pods correspondant au service (ici `10.244.2.4:8080`) |
| Pod back-pod      | Pod ciblé par le service, avec l'IP `10.244.2.4`             |
| kube-proxy        | Proxy réseau sur chaque Node qui route le trafic vers les Endpoints |
