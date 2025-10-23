# Déploiement et test de WordPress via le [chart Bitnami/WordPress](https://artifacthub.io/packages/helm/bitnami/wordpress) sur Minikube
[chart Bitnami/WordPress](https://artifacthub.io/packages/helm/bitnami/wordpress)

## Architecture Kubernetes du Chart Bitnami/Wordpress

```ascii
+-------------------------------------------------------------------------------------+
|                                 CLUSTER KUBERNETES                                  |
+-------------------------------------------------------------------------------------+
                                            |
                                            v
+------------------------------------+ (Accès Externe)
|         Utilisateur / Client         |
+------------------------------------+
            |
            v
       +----------+
       | INGRESS  | <----- (Optionnel, pour HTTP/S et nom de domaine)
       +----------+
            | (Route Traffic)
            v
+-------------------------------------------------------------------------------------+
|                                COMPOSANT WORDPRESS                                  |
+-------------------------------------------------------------------------------------+
|    +--------------------+       +-------------------------+
|    | WORDPRESS SERVICE  | <-----| WORDPRESS DEPLOYMENT    |
|    | (LoadBalancer/CP)  |       +-------------------------+
|    +--------------------+                | (Gère les Pods/Réplicas)
|             |                            v
|             |              +-------------+-------------+
|             v              |  WORDPRESS POD (Container)|
|        (Traffic)           +-------------+-------------+
|                                    | (Utilise)
|     +-------------------------+----+----+-------------------------+
|     | SECRET (Mots de Passe)  | ConfigMap | PVC (Volume Persistant) |
|     +-------------------------+-----------+-------------------------+
+-------------------------------------------------------------------------------------+
            | (Se Connecte à la BD)
            v
+-------------------------------------------------------------------------------------+
|                                COMPOSANT MARIADB (BD)                               |
+-------------------------------------------------------------------------------------+
|    +--------------------+       +-------------------------+
|    | MARIADB SERVICE    | <-----| MARIADB DEPLOYMENT/SS   |
|    | (ClusterIP)        |       +-------------------------+
|    +--------------------+                | (Gère les Pods/Réplicas)
|             |                            v
|             |              +-------------+-------------+
|             v              |  MARIADB POD (Container)  |
|        (Traffic Interne)   +-------------+-------------+
|                                    | (Utilise)
|                     +------------+-------------+---------------------------+
|                     | SECRET (BD Credentials)  | PVC (Volume Persistant)   |
|                     +------------+-------------+---------------------------+
+-------------------------------------------------------------------------------------+
```

### Explication des Objets Kubernetes

Voici les objets Kubernetes (ressources) clés déployés par le chart Bitnami WordPress:

* **Deployment** ou **StatefulSet** (pour MariaDB) : Définit l'état souhaité pour les applications (WordPress et MariaDB), y compris le nombre de **Pods** répliqués.
* **Pod** : L'unité de déploiement la plus petite, contenant le ou les conteneurs (l'application WordPress, le serveur MariaDB).
* **Service** :
    * **WordPress Service** : Fournit un point d'entrée stable pour l'application. Il utilise souvent le type `LoadBalancer` pour exposer l'application à l'extérieur du cluster ou `NodePort`/`ClusterIP`.
    * **MariaDB Service** : Un service de type `ClusterIP` qui permet aux Pods WordPress de se connecter à la base de données de manière stable, en utilisant un nom DNS interne.
* **Ingress** : Un objet optionnel qui gère l'accès externe HTTP et HTTPS, en fournissant le routage basé sur le nom d'hôte ou le chemin vers le Service WordPress.
* **Secret** : Stocke les informations sensibles, principalement les mots de passe de l'administrateur WordPress et de l'utilisateur MariaDB. Ces secrets sont montés ou injectés comme variables d'environnement dans les Pods.
* **PersistentVolumeClaim (PVC)** : Une demande de stockage persistant. Le chart crée deux PVC :
    1.  Un pour stocker les fichiers de WordPress (thèmes, plugins, médias uploadés).
    2.  Un pour stocker les données de la base de données MariaDB.
* **ConfigMap** : Peut être utilisé pour stocker des données de configuration non sensibles, comme des fichiers de configuration PHP ou Nginx/Apache.

---

## 1. Prérequis

Avant de commencer, assurez-vous que les outils suivants sont installés et configurés sur votre machine :

* **Minikube** — pour exécuter un cluster Kubernetes local
* **kubectl** — pour interagir avec le cluster
* **Helm** — pour gérer les charts Kubernetes

Démarrez ensuite votre cluster Minikube :

```bash
minikube start
```

---

## 2. Installation du chart Bitnami/WordPress

### Étape 1 — Ajouter le dépôt Bitnami

Ajoutez le dépôt Helm officiel de Bitnami, puis mettez-le à jour :

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Étape 2 — Déployer WordPress

Installez le chart **Bitnami/WordPress** en précisant la version souhaitée (ici : `27.1.3`) :

```bash
helm install my-wordpress bitnami/wordpress --version 27.1.3
```

### Étape 3 — Adapter le type de service à Minikube

Par défaut, le chart crée un service de type **LoadBalancer**.
Cependant, Minikube ne fournit pas d’adresse IP externe par défaut.
Pour rendre le site accessible, modifiez le type du service en **NodePort** :

```bash
helm upgrade my-wordpress bitnami/wordpress --set service.type=NodePort
```

### Étape 4 — Obtenir l’URL de WordPress

Récupérez l’URL publique permettant d’accéder à WordPress depuis votre navigateur :

```bash
minikube service my-wordpress
```

Exemple de sortie :

```
|-----------|--------------|-------------|---------------------------|
| NAMESPACE |     NAME     | TARGET PORT |            URL            |
|-----------|--------------|-------------|---------------------------|
| default   | my-wordpress | http/80     | http://192.168.49.2:32738 |
|           |              | https/443   | http://192.168.49.2:31014 |
|-----------|--------------|-------------|---------------------------|
```

---

## 3. Vérification du déploiement

Assurez-vous que tous les composants sont correctement déployés et en cours d’exécution :

```bash
kubectl get pods
```

Vous devriez observer au minimum les pods suivants :

* `my-wordpress-wordpress-*`
* `my-wordpress-mariadb-*`

Vérifiez ensuite la création des services associés :

```bash
kubectl get svc
```

Vous devriez obtenir une liste semblable à :

* `my-wordpress`
* `my-wordpress-mariadb`
* `my-wordpress-mariadb-headless`

---

## 4. Accès à l’interface d’administration WordPress

Une fois WordPress accessible, connectez-vous à l’interface d’administration via :

```
http://<minikube-ip>/wp-admin
```

Pour récupérer le mot de passe administrateur généré automatiquement :

```bash
kubectl get secret --namespace default my-wordpress -o jsonpath="{.data.wordpress-password}" | base64 -d
```

L’utilisateur par défaut est :

```
user
```

---

## 5. (Optionnel) Personnalisation du chart

Si vous souhaitez ajuster la configuration du chart Bitnami, vous pouvez le télécharger localement :

```bash
helm pull bitnami/wordpress --untar
```

Cela vous permettra de modifier les valeurs par défaut avant déploiement.

---

## 6. Nettoyage et maintenance

Pour supprimer complètement le déploiement WordPress (pods, services et ressources associées) :

```bash
helm uninstall my-wordpress
```

