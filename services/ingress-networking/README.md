## Ingress dans Kubernetes

Dans cette partie, nous allons aborder **Ingress** dans Kubernetes, comprendre son rôle par rapport aux **Services**, et apprendre à le configurer avec des exemples concrets.

***

### Rappel : Services dans Kubernetes

Un **Service** permet de rendre accessible un **Pod** ou un groupe de Pods (via un **Deployment**) au sein du **Cluster** ou vers l’extérieur, selon son type :
- **ClusterIP** : Expose le Service uniquement à l’intérieur du Cluster.
- **NodePort** : Expose le Service sur un port spécifique de chaque **Node** (port entre 30000 et 32767).
- **LoadBalancer** : Crée automatiquement un **Load Balancer** externe via le fournisseur cloud (par ex. AWS, GCP, Azure).

#### Exemple pratique

Supposons que vous déployez une application d’e-commerce :

1. Vous construisez votre image Docker :
   ```
   $ docker build -t myonlinestore:1.0 .
   ```

2. Vous la déployez dans Kubernetes :
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-app
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: web-app
     template:
       metadata:
         labels:
           app: web-app
       spec:
         containers:
         - name: web-app
           image: myonlinestore:1.0
           ports:
           - containerPort: 80
   ```

3. Vous créez un Service **ClusterIP** pour la base de données MySQL :
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql-service
   spec:
     selector:
       app: mysql
     ports:
     - port: 3306
       targetPort: 3306
     type: ClusterIP
   ```

4. Pour votre application web, vous exposez le Service :
   ```yaml
   kind: Service
   apiVersion: v1
   metadata:
     name: web-service
   spec:
     type: NodePort
     selector:
       app: web-app
     ports:
       - port: 80
         targetPort: 80
         nodePort: 30080
   ```

Vos utilisateurs peuvent maintenant accéder à l’application via :  
`http://<node-ip>:30080`

***

### Limites des Services de type NodePort et LoadBalancer

- Les **NodePort** ne permettent pas d’utiliser les ports standards (80, 443).
- Les **LoadBalancer** coûtent cher sur le cloud : chaque Service de ce type crée un nouveau Load Balancer.
- Chaque modification de routage (nouvelle application, nouveau chemin, etc.) nécessite des reconfigurations manuelles.
- La gestion du SSL/TLS peut devenir complexe lorsqu’il faut le configurer pour chaque Service séparément.

***

### Introduction d’Ingress

**Ingress** agit comme un **reverse proxy** et un **load balancer** de niveau 7 (HTTP/HTTPS) géré directement par Kubernetes.  
Il permet :

- De centraliser le routage du trafic externe vers plusieurs Services.
- D’utiliser une seule IP externe et un seul point d’entrée (DNS).
- De configurer le routage par **nom d’hôte (host)** ou **chemin (path)**.
- D’ajouter des certificats SSL/TLS pour HTTPS.

#### Exemple visuel

| URL                      | Service cible   | Description                    |
|---------------------------|-----------------|--------------------------------|
| myonlinestore.com/        | web-service     | Application principale         |
| myonlinestore.com/watch   | video-service   | Service de streaming vidéo     |

***

### Déploiement d’un Ingress Controller

Kubernetes **ne déploie pas d’Ingress Controller par défaut**.  
Vous devez en installer un, comme **NGINX**, **HAProxy**, **Traefik**, **Contour**, ou **Istio**.  
Dans notre exemple, nous utiliserons **NGINX Ingress Controller**.

#### Étapes d’installation

1. **Déployer le contrôleur NGINX** :
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-ingress-controller
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nginx-ingress
     template:
       metadata:
         labels:
           app: nginx-ingress
       spec:
         containers:
         - name: controller
           image: registry.k8s.io/ingress-nginx/controller:v1.10.1
           ports:
           - containerPort: 80
           - containerPort: 443
   ```

2. **Exposer le contrôleur** :
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-ingress
   spec:
     type: NodePort
     selector:
       app: nginx-ingress
     ports:
     - name: http
       port: 80
       targetPort: 80
     - name: https
       port: 443
       targetPort: 443
   ```

3. **Créer une ConfigMap** pour ses paramètres (optionnelle mais recommandée) :
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: nginx-config
   data: {}
   ```

4. **Ajouter un ServiceAccount avec les droits nécessaires** pour qu’il surveille les ressources **Ingress** et adapte sa configuration NGINX automatiquement.

***

### Création d’une ressource Ingress

Une **ressource Ingress** définit les règles de routage du trafic.

#### Exemple simple (un seul Service)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-web
spec:
  defaultBackend:
    service:
      name: web-service
      port:
        number: 80
```

#### Exemple avancé : routage par chemin

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-rules
spec:
  rules:
  - host: myonlinestore.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 80
```

#### Exemple de routage par nom d’hôte

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-ingress
spec:
  rules:
  - host: myonlinestore.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: watch.myonlinestore.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 80
```

***

## Résumé concis

- **Service** expose des Pods au sein du Cluster (ClusterIP) ou vers l’extérieur (NodePort, LoadBalancer).
- **Ingress** centralise le routage HTTP/HTTPS avec un seul point d’entrée et des règles flexibles (host/path).
- Il fonctionne via un **Ingress Controller**, généralement **NGINX**, déployé dans le Cluster.
- Les configurations de sécurité (SSL, certificats TLS) et de routage sont gérées dans les **ressources Ingress**.
- Cela simplifie considérablement la mise à l’échelle, les déploiements multi-services et la maintenance.


