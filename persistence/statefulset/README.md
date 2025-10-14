# Partie 1 - les StatefulSets dans Kubernetes

---

## 1. Introduction

Avant d’aborder le concept de **StatefulSet**, il est essentiel de comprendre pourquoi il existe et dans quels cas un simple **Deployment** ne suffit pas.

Un **Deployment** gère efficacement des applications *stateless* (sans état persistant), comme des serveurs web.
Mais lorsqu’il s’agit d’applications *stateful* (qui conservent un état entre les redémarrages, comme une base de données), un **StatefulSet** devient indispensable.

---

## 2. Du serveur physique à la base de données distribuée

Imaginons un scénario classique :

> Objectif : Déployer un serveur MySQL sur une machine physique.

1. Nous installons **MySQL** sur un serveur et créons une base de données.
   → Le service est opérationnel, les applications peuvent y écrire des données.

2. Pour assurer la **haute disponibilité (HA)**, nous déployons plusieurs serveurs supplémentaires.
   → Chaque nouveau serveur possède une base vide.

3. Nous devons donc **répliquer les données** du serveur principal vers les serveurs secondaires.

---

## 3. Comprendre la réplication MySQL

Nous utiliserons l’exemple d’une **topologie Master/Slave** :

* **Master** : reçoit toutes les écritures.
* **Slaves** : lisent les données et répliquent en continu depuis le master.

### Étapes typiques :

1. Le **Master** est configuré et contient les données initiales.
2. Le **Slave 1** effectue une **copie initiale** (clone) du Master.
3. Une **réplication continue** est activée entre Master → Slave 1.
4. Le **Slave 2** peut ensuite cloner ses données à partir de **Slave 1** pour réduire la charge sur le Master.
5. Enfin, Slave 2 active à son tour la réplication continue depuis le Master.

> Remarque : chaque Slave doit connaître l’adresse du Master pour se connecter et synchroniser les données.

---

### Exemple concret (en dehors de Kubernetes)

| Serveur     | Rôle   | Action principale                       |
| ----------- | ------ | --------------------------------------- |
| `db-master` | Master | Crée la base initiale                   |
| `db-slave1` | Slave  | Clone depuis le master, puis réplique   |
| `db-slave2` | Slave  | Clone depuis `db-slave1`, puis réplique |

Ainsi, nous obtenons une base distribuée avec **un flux de données unidirectionnel** (Master → Slaves).

---

## 4. Problèmes avec un Deployment Kubernetes

Essayons maintenant de déployer cette architecture dans Kubernetes à l’aide d’un **Deployment**.

* Chaque instance (Master, Slave1, Slave2) serait représentée par un **Pod**.
* Les Pods sont gérés par le **Deployment Controller**.

### Limites d’un Deployment

1. **Pas d’ordre de création garanti** :
   Tous les Pods d’un Deployment sont lancés simultanément.
   → Impossible d’assurer que le Master soit prêt avant les Slaves.

2. **Noms et adresses dynamiques** :
    * Les Pods d’un Deployment reçoivent des **noms aléatoires** (`mysql-5d7f9b87f6-bn2zl`).
    * Les **adresses IP** changent à chaque recréation du Pod.
      → Les Slaves risquent de pointer vers une adresse invalide après un redémarrage.

3. **Pas d’identité persistante** :
   Si un Pod est détruit, son remplaçant est totalement nouveau.
   → Impossible de maintenir un lien stable entre Master et Slaves.

---

## 5. Solution : le StatefulSet

Le **StatefulSet** résout précisément ces problèmes.

Un **StatefulSet** est similaire à un **Deployment**, mais avec des propriétés supplémentaires qui garantissent la **stabilité**, l’**ordre** et l’**identité** des Pods.

### Caractéristiques principales

1. **Création séquentielle des Pods**

    * Les Pods sont créés **un par un** dans l’ordre.
    * Exemple :
        1. `mysql-0` (Master)
        2. `mysql-1` (Slave 1)
        3. `mysql-2` (Slave 2)
    * Chaque Pod suivant n’est déployé que lorsque le précédent est en état **Running** et **Ready**.

2. **Nom et identité persistante (On abordera cela en détail dans la partie 3) **
    * Les Pods d’un StatefulSet ont des **noms prévisibles** :
      `nomDuStatefulSet-<index>` (ex: `mysql-0`, `mysql-1`, `mysql-2`).
    * Si `mysql-0` est supprimé, le Pod recréé gardera **le même nom**.

---

### Exemple complet : StatefulSet MySQL

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql" # => (défini plus bas dans le cours)
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: example
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

### Fonctionnement :

* Le Pod `mysql-0` joue le rôle de **Master**.
* `mysql-1` et `mysql-2` sont les **Slaves**, configurés pour répliquer à partir du Master (`mysql-0.mysql.default.svc.cluster.local`).
* Même si `mysql-0` est redémarré, son **nom DNS** reste identique → la réplication continue sans rupture.

---

## 6. Avantages des StatefulSets

| Fonctionnalité       | Deployment            | StatefulSet                           |
| -------------------- | --------------------- | ------------------------------------- |
| Ordre de déploiement | Non garanti           | Séquentiel                            |
| Nom fixe du Pod      | Non                   | Oui (`podname-0`, `podname-1`, …)     |
| Adresse stable (DNS) | Non                   | Oui                                   |
| Stockage persistant  | Possible mais partagé | Dédié à chaque Pod                    |
| Idéal pour           | Apps stateless        | Bases de données, systèmes distribués |

---

## 7. Conclusion

Les **StatefulSets** sont essentiels pour déployer des applications *stateful* dans Kubernetes.
Ils garantissent :

* Un ordre strict de déploiement (Master avant Slaves).
* Des noms et adresses DNS stables.
* Des volumes persistants individuels.

C’est cette combinaison qui permet à des systèmes comme **MySQL**, **Cassandra**, ou **MongoDB** de fonctionner correctement dans un environnement Kubernetes.

---

## Résumé concis

| Concept                 | Explication essentielle                                                                                                 |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Problème**            | Les Deployments ne garantissent ni l’ordre ni la stabilité des Pods nécessaires à la réplication d’une base de données. |
| **Solution**            | Le StatefulSet crée des Pods avec des identités stables et un ordre déterministe.                                       |
| **Avantages clés**      | Noms fixes (`mysql-0`, `mysql-1`…), adresses DNS persistantes, stockage dédié.                                          |
| **Cas d’usage typique** | Bases de données (MySQL, MongoDB), clusters distribués (Kafka, Elasticsearch).                                          |
| **Exemple**             | MySQL Master/Slaves : `mysql-0` = Master, `mysql-1` = Slave1, `mysql-2` = Slave2.                                       |

***
***

# Partie 2 — Les Services et la mise en réseau d’un StatefulSet

---

## 1. Pourquoi un Service est essentiel pour un StatefulSet

Dans Kubernetes, les **Pods** sont éphémères et leurs adresses IP changent à chaque recréation.
Pour permettre à d’autres Pods (clients ou Slaves) d’accéder à un Pod **de façon stable**, on utilise un **Service**.

Cependant, tous les Services ne se valent pas pour un **StatefulSet**.

Un **StatefulSet** a besoin de deux types de Services :

1. **Un Headless Service** – pour attribuer un **nom DNS stable à chaque Pod**.
2. (Optionnel) **Un Service standard** – pour exposer l’application de manière globale (vers l’extérieur ou au sein du cluster).

---

## 2. Le Headless Service : fondation d’un StatefulSet

Un **Headless Service** est un Service **sans ClusterIP** (`clusterIP: None`).

Contrairement à un Service standard (qui fait du *load balancing* entre Pods),
le Headless Service **ne distribue pas le trafic** : il permet à chaque Pod d’avoir **sa propre entrée DNS**.

### Exemple concret

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  type: ClusterIP
  ports:
  - port: 3306
    name: mysql
```

Ici :

* `clusterIP: None` → rend le Service “Headless”.
* Chaque Pod aura **son propre nom DNS**, généré automatiquement par Kubernetes.

---

## 3. Résolution DNS dans un StatefulSet

Kubernetes génère un **nom DNS stable** pour chaque Pod selon la structure suivante :

```
<nom_du_pod>.<nom_du_service>.<namespace>.svc.cluster.local
```

### Exemple avec le StatefulSet MySQL

| Pod     | Nom DNS complet                         |
| ------- | --------------------------------------- |
| mysql-0 | mysql-0.mysql.default.svc.cluster.local |
| mysql-1 | mysql-1.mysql.default.svc.cluster.local |
| mysql-2 | mysql-2.mysql.default.svc.cluster.local |

Ainsi :

* `mysql-1` peut contacter le Master via `mysql-0.mysql.default.svc.cluster.local`.
* Même après redémarrage, ce nom reste **valide et inchangé**.

---

## 4. Association d’un StatefulSet à un Headless Service

Le StatefulSet doit **référencer** ce Headless Service via le champ `serviceName`.

### Exemple complet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"   # <--- nom du Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: example
        ports:
        - containerPort: 3306
```

Cela garantit que chaque Pod du StatefulSet sera accessible via son **DNS individuel**, fourni par le Service `mysql`.

---

## 5. Service standard pour l’accès global

Dans certaines architectures, on souhaite que les applications clientes ne se connectent **pas directement à un Pod**, mais à une **adresse unique** (le Service).

Pour cela, on peut créer un **Service standard (non headless)** qui **redige le trafic** vers le Pod principal (souvent le Master).

### Exemple : Service vers le Master

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
spec:
  selector:
    statefulset.kubernetes.io/pod-name: mysql-0
  type: ClusterIP
  ports:
  - port: 3306
    name: mysql
```

Ce Service pointe **uniquement** vers le Pod `mysql-0`, identifié comme le Master.

> Avantage :
> Les clients peuvent se connecter à `mysql-master.default.svc.cluster.local`,
> sans connaître le nom du Pod Master (toujours `mysql-0`).

---

## 6. Exemple complet : StatefulSet MySQL + Services

Voici une configuration complète fonctionnelle pour un cluster MySQL avec un Master et deux Slaves.

```yaml
# 1 Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  type: ClusterIP
  ports:
  - port: 3306
    name: mysql

---

# 2 Service vers le Master "mysql-0"
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
spec:
  selector:
    statefulset.kubernetes.io/pod-name: mysql-0
  type: ClusterIP
  ports:
  - port: 3306
    name: mysql

---

# 3 StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: example
        ports:
        - containerPort: 3306
```

> **Résultat attendu** :
> * `mysql-0` = Master, accessible via `mysql-master` ou `mysql-0.mysql`.
> * `mysql-1` et `mysql-2` = Slaves, accessibles via leurs noms DNS.
> * Les applications internes peuvent utiliser `mysql-master` pour écrire et `mysql` (Headless Service) pour lire.

---

## 7. Bonnes pratiques (Best Practices)

| Bonne pratique                                              | Description                                                                                         |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Utiliser un **Headless Service**                            | Obligatoire pour nommer les Pods de manière stable.                                                 |
| Toujours définir `serviceName` dans le StatefulSet          | Sans cela, la résolution DNS ne fonctionne pas.                                                     |
| Créer un Service dédié au Master                            | Simplifie la connexion des clients applicatifs.                                                     |
| Éviter le *load balancing* sur des Pods de bases de données | Les connexions doivent être dirigées intelligemment selon le rôle (Master/Slave).                   |
| Surveiller les **readinessProbes**                          | Elles garantissent que les Pods Slaves ne reçoivent pas de requêtes avant la fin de la réplication. |

---

## Résumé concis

| Élément                                  | Rôle                                                                                                |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Headless Service (`clusterIP: None`)** | Fournit une entrée DNS unique pour chaque Pod (`podname.servicename.namespace.svc.cluster.local`).  |
| **Service standard**                     | Expose un ou plusieurs Pods via une adresse unique, souvent utilisé pour le Master.                 |
| **DNS stable**                           | Permet aux Pods Slaves de toujours connaître le Master (`mysql-0.mysql.default.svc.cluster.local`). |
| **Lien avec StatefulSet**                | Le champ `serviceName` lie le StatefulSet à son Headless Service.                                   |
| **Exemple typique**                      | MySQL Master (`mysql-0`) + 2 Slaves (`mysql-1`, `mysql-2`) avec Services `mysql` et `mysql-master`. |

***
***

# Partie 3 — Gestion du stockage persistant dans un StatefulSet

---

## 1. Pourquoi le stockage persistant est essentiel

Dans un StatefulSet, chaque Pod représente un **service stateful** (ex : base de données).
Si un Pod est supprimé ou recréé par Kubernetes :

* Son **nom DNS reste stable** (`mysql-0`, `mysql-1`, …)
* Mais si le stockage est éphémère, **toutes les données seraient perdues**

➡️ Pour éviter cela, Kubernetes utilise **PersistentVolume (PV)** et **PersistentVolumeClaim (PVC)**.

---

## 2. Concepts clés

### 2.1 PersistentVolume (PV)

* Ressource dans Kubernetes représentant **un espace de stockage physique ou cloud**.
* Définie par un administrateur ou via provisionnement dynamique.
* Possède des propriétés :
    * `capacity` : taille du volume (ex : 5Gi)
    * `accessModes` : type d’accès (`ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`)
    * `storageClassName` : classe de stockage (SSD, HDD, cloud, etc.)

### 2.2 PersistentVolumeClaim (PVC)

* Demande de stockage faite par un Pod.
* Le Pod ne connaît pas le stockage physique exact ; il réclame **“x Gi de stockage avec telle classe”**.
* Kubernetes lie automatiquement le PVC à un PV disponible.

### 2.3 volumeClaimTemplates (StatefulSet)

* Spécifique aux **StatefulSets**.
* Permet de créer **un PVC unique par Pod automatiquement**.
* Assure que chaque Pod a **son propre volume persistant**.

> Dans un Deployment, les Pods partagent souvent le même volume ou utilisent des volumes éphémères.
> StatefulSet garantit **persistance + identité** par Pod.

---

## 3. Comment StatefulSet gère le stockage

1. Le StatefulSet définit `volumeClaimTemplates`.
2. Pour chaque Pod créé, Kubernetes génère un **PVC unique** basé sur ce template.
3. Chaque Pod monte son **PVC** dans le conteneur.
4. Si un Pod est supprimé ou recréé, **le PVC reste attaché**, garantissant la persistance des données.

---

## 4. Exemple concret : MySQL StatefulSet avec stockage persistant

### 4.1 Headless Service (DNS stable)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless Service
  type: ClusterIP
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

### 4.2 StatefulSet avec volumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"   # Headless Service pour DNS stable
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: example
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
      storageClassName: standard
```

### 4.3 Ce qui se passe

| Pod     | PVC généré            | Mount path     |
| ------- | --------------------- | -------------- |
| mysql-0 | mysql-storage-mysql-0 | /var/lib/mysql |
| mysql-1 | mysql-storage-mysql-1 | /var/lib/mysql |
| mysql-2 | mysql-storage-mysql-2 | /var/lib/mysql |

* Chaque Pod reçoit **son propre volume dédié**.
* Même si `mysql-0` est supprimé, son volume `mysql-storage-mysql-0` reste attaché au nouveau Pod.
* Garantit **persistance des données MySQL**, indépendamment du cycle de vie des Pods.

---

## 5. Bonnes pratiques pour le stockage persistant

| Pratique                                           | Explication                                                              |
| -------------------------------------------------- | ------------------------------------------------------------------------ |
| Utiliser **volumeClaimTemplates** dans StatefulSet | Chaque Pod a son propre PVC                                              |
| Définir `storageClassName`                         | Pour choisir SSD/HDD ou stockage cloud dynamique                         |
| `accessModes: ReadWriteOnce`                       | Les bases de données doivent écrire uniquement depuis un Pod à la fois   |
| Vérifier la capacité (`requests`)                  | Assurez-vous qu’elle correspond aux besoins de stockage de l’application |
| Surveiller l’état du PVC                           | `kubectl get pvc` pour vérifier que tous les volumes sont provisionnés   |

---

## 6. Visualisation du flow stockage

```
StatefulSet mysql
 ├─ Pod mysql-0 ──> PVC mysql-storage-mysql-0 ──> PersistentVolume (5Gi)
 ├─ Pod mysql-1 ──> PVC mysql-storage-mysql-1 ──> PersistentVolume (5Gi)
 └─ Pod mysql-2 ──> PVC mysql-storage-mysql-2 ──> PersistentVolume (5Gi)
```

> Chaque Pod a **son volume indépendant**.
> Les données sont **persistantes même si les Pods sont recréés**.
> Les noms des volumes suivent le **nom du Pod**, ce qui facilite le suivi et la réplication.

---

## 7. Résumé concis

1. **PV** = ressource physique/virtuelle de stockage dans le cluster.
2. **PVC** = demande de stockage faite par un Pod.
3. **volumeClaimTemplates** = template pour créer **automatiquement un PVC unique par Pod dans un StatefulSet**.
4. Garantit :
    * stockage **persistant** par Pod
    * **ré-association automatique** après recréation du Pod
    * **isolation des données** entre Pods (Master vs Slaves).

***
***

## Résumé concis et global du fonctionnement d’un StatefulSet

1. **Objectif d’un StatefulSet**
    * Déployer des applications *stateful* (bases de données, clusters distribués) avec :
        * **Ordre de création garanti** (`mysql-0` → `mysql-1` → `mysql-2`)
        * **Identité persistante** (nom DNS et nom Pod stable)
        * **Stockage persistant** par Pod

2. **Pods et noms stables**
    * Chaque Pod obtient un **nom prévisible** : `<statefulset-name>-<index>`
    * Kubernetes ajoute un label automatique :
      ```yaml
      statefulset.kubernetes.io/pod-name: mysql-0
      ```

3. **Headless Service (`clusterIP: None`)**
    * Permet la **résolution DNS individuelle** pour chaque Pod :
      ```
      <pod-name>.<service-name>.<namespace>.svc.cluster.local
      ```
    * Pas de load balancing interne, chaque Pod est accessible directement.

4. **Service pour Master**
    * Service classique (`ClusterIP`) pointant uniquement vers le Pod Master (`mysql-0`)
    * Permet aux applications de se connecter facilement au Master sans connaître le nom exact du Pod.

5. **Flow de réplication (MySQL exemple)**
    1. `mysql-0` (Master) démarre et s’initialise.
    2. `mysql-1` (Slave1) clone les données du Master et active la réplication.
    3. `mysql-2` (Slave2) clone les données du Slave1 et active la réplication.
    4. Tous les Pods Slaves connaissent le Master via DNS (`mysql-0.mysql.default.svc.cluster.local`) ou Service (`mysql-master`).

6. **VolumeClaimTemplates**
    * Crée un **PersistentVolumeClaim unique par Pod**
    * Garantit la **persistance des données** même si le Pod est recréé ou déplacé.
    * Spécifique aux StatefulSets, **non disponible dans un Deployment**.

7. **Résumé DNS et accès**
    * `mysql` (Headless Service) → DNS individuel pour chaque Pod
    * `mysql-master` (Service ClusterIP) → accès stable au Master

***
***

# Architecture Yaml

```yaml
# 1 Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  type: ClusterIP
  ports:
  - port: 3306
    name: mysql

---

# 2 Service vers le Master "mysql-0"
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
spec:
  selector:
    statefulset.kubernetes.io/pod-name: mysql-0
  type: ClusterIP
  ports:
  - port: 3306
    name: mysql

---

# 3 StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: example
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
      storageClassName: standard
```

```
                     +------------------------+
                     |       Cluster DNS      |
                     |------------------------|
                     | mysql-0.mysql          |
                     | mysql-1.mysql          |
                     | mysql-2.mysql          |
                     +-----------+------------+
                                 |
               +-----------------+-----------------+
               | Headless Service: mysql           |
               | ClusterIP: None                   |
               +-----------------+-----------------+
                                 |
       ------------------------------------------------------
       |                        |                           |
+---------------+        +---------------+           +---------------+
| Pod: mysql-0  |        | Pod: mysql-1  |           | Pod: mysql-2  |
|---------------|        |---------------|           |---------------|
| Container:    |        | Container:    |           | Container:    |
| mysql:8       |        | mysql:8       |           | mysql:8       |
| Port: 3306    |        | Port: 3306    |           | Port: 3306    |
| Env: ROOT_PW  |        | Env: ROOT_PW  |           | Env: ROOT_PW  |
| VolumeMount:  |        | VolumeMount:  |           | VolumeMount:  |
| /var/lib/mysql|        | /var/lib/mysql|           | /var/lib/mysql|
+-------+-------+        +-------+-------+           +-------+-------+
        |                        |                           |
        | PVC: mysql-storage-0   | PVC: mysql-storage-1      | PVC: mysql-storage-2
        | 5Gi, RWO               | 5Gi, RWO                  | 5Gi, RWO
        v                        v                           v
+----------------+      +----------------+           +----------------+
| PersistentVol  |      | PersistentVol  |           | PersistentVol  |
| mysql-storage-0|      | mysql-storage-1|           | mysql-storage-2|
+----------------+      +----------------+           +----------------+

       +-------------------------+
       | Service: mysql-master   |
       | ClusterIP -> mysql-0    |
       +-------------------------+
                     |
                     v
                 (Accès Master)
```

### Explications :

1. **Headless Service `mysql`**
    * Permet aux Pods d’être adressables via DNS comme `mysql-0.mysql`, `mysql-1.mysql`…
    * Pas de load balancing, juste du DNS.

2. **StatefulSet `mysql`**
    * Crée 3 Pods avec noms stables : `mysql-0`, `mysql-1`, `mysql-2`.
    * Chaque Pod a son **PVC unique** (`mysql-storage-0`, etc.) → stockage persistant.

3. **Service `mysql-master`**
    * Pointera uniquement vers `mysql-0` (master).
    * Utile pour que les applications écrivent sur le master.

4. **DNS dans le Cluster**
    * Permet aux Pods de se retrouver entre eux via `mysql-0.mysql`, etc.

5. **Volumes persistants (PVCs)**
    * Chaque Pod a son propre volume → pas de conflit sur les données.

---