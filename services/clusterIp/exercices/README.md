# TODO

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
      command: ["sleep", "3600"]
      # curl est dans alpine, on pourra tester via kubectl exec

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
# Pas de réponse spécifique car le pod front est passif (sleep)
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


