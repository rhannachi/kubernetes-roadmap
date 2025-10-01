# Le Service Kubernetes de type "ClusterIP"

Une application web complète contient souvent plusieurs types de **Pods** hébergeant différentes parties de l’application. Par exemple :
- un groupe de Pods pour le **front-end** (serveur web),
- un autre groupe pour le **back-end**,
- un ensemble de Pods pour une base clé-valeur comme **Redis**,
- et enfin un groupe pour une base de données persistante comme **MySQL**.

Dans cette architecture, le front-end doit pouvoir communiquer avec le back-end, qui à son tour communique avec la base de données et le service Redis.

### Pourquoi un Service ClusterIP ?

Chaque Pod possède une adresse IP, mais ces adresses sont **éphémères** : lorsqu’un Pod est recréé, son IP change. Il est donc impossible de s’appuyer sur ces IP pour la communication interne.\
Par ailleurs, lorsqu’un Pod front-end veut atteindre un service back-end composé de plusieurs Pods, il faut une manière simple de sélectionner lequel joindre.

Un **Service** Kubernetes de type **ClusterIP** permet de :
- regrouper un ensemble de Pods (via un selector basé sur labels),
- fournir une adresse IP unique et stable au sein du Cluster,
- exposer une seule interface pour accéder **aux Pods membres** du Service,
- répartir automatiquement les requêtes reçues entre ces Pods (load balancing interne, basé sur iptables ou IPVS).

***

### Exemple concret avec un Service ClusterIP

Supposons que l’on crée un Service pour le back-end, qui expose ses Pods sur le port 80.

#### Définition YAML du Service :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP  # Type par défaut : communication interne au Cluster
  selector:
    app: my-backend-app
  ports:
    - port: 80         # Port exposé par le Service dans le Cluster
      targetPort: 80   # Port sur lequel les Pods écoutent
```

#### Définition YAML d’un Pod back-end correspondant :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod-1
  labels:
    app: my-backend-app
spec:
  containers:
    - name: backend-container
      image: nginx
      ports:
        - containerPort: 80
```

***

### Déploiement et utilisation

1. Créer les Pods via `kubectl apply -f backend-pod.yaml` (ou via un Deployment).
2. Créer le Service via `kubectl apply -f backend-service.yaml`.
3. Le Service obtient une adresse IP ClusterIP (ex : 10.96.0.10).
4. Les autres Pods du Cluster peuvent joindre le back-end via cette IP ou via le nom DNS du Service `backend` (ex : `curl http://backend:80`).

***

### Avantages du Service ClusterIP

- **Stabilité** : l’adresse IP du Service est fixe, même si les Pods changent.
- **Load balancing interne** : Kubernetes répartit de façon équilibrée les appels entre plusieurs Pods correspondant au selector du Service.
- **Découplage** : les composants communiquent via un point stable et unique au lieu de gérer manuellement les adresses IP des Pods.
- **Sécurité** : en étant accessible uniquement dans le Cluster, il Isoler les services internes de toute exposition externe non désirée.

***

### Résumé concis

- Le Service **ClusterIP** est la méthode par défaut et la plus simple pour exposer un groupe de Pods **uniquement à l’intérieur du Cluster**.
- Il fournit une IP virtuelle unique et stable pour accéder à ces Pods.
- La communication inter-Pods se fait via cette IP ou via un nom DNS interne.
- Il assure le load balancing interne, distribuant automatiquement le trafic entre les Pods du Service.
- Idéal pour connecter différentes couches d’une application (front-end, back-end, base de données) sans exposer ces services à l’extérieur.

