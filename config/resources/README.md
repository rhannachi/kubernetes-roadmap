# Resources Kubernetes

Commençons par examiner un Cluster Kubernetes composé de trois Nodes. Chaque Node possède une certaine quantité de CPU et de mémoire.\
De leur côté, les Pods ont également besoin de ressources pour fonctionner. Par exemple, un Pod peut demander 2 vCPU et 1 Gi de mémoire.

Lorsqu’un Pod doit être programmé, c’est le Scheduler de Kubernetes qui décide sur quel Node il sera placé. Le Scheduler prend en compte les ressources demandées par le Pod et celles disponibles sur chaque Node.\
Si un Node ne dispose pas de ressources suffisantes, le Scheduler évite d’y placer le Pod. Dans le cas où aucune ressource n’est disponible sur l’ensemble des Nodes, le Pod reste en état *Pending*.\
La commande `kubectl describe pod` permet alors d’afficher les événements et de constater, par exemple, une insuffisance de CPU.

## Requests et Limits
Lors de la création d’un Pod, il est possible de définir des **Requests** (demande minimale garantie) et des **Limits** (quantité maximale utilisable) pour CPU et mémoire à l’intérieur de chaque conteneur.

- Une **Request** représente la quantité minimale de ressources garantie pour un conteneur. Elle est utilisée par le Scheduler afin de trouver un Node adéquat.
- Une **Limit** fixe le maximum que le conteneur peut consommer. Si ce maximum est dépassé :
    - pour le CPU, Kubernetes régule et empêche l’usage au-delà de la limite,
    - pour la mémoire, le conteneur peut dépasser temporairement, mais s’il consomme durablement plus que la limite, il sera terminé avec une erreur **OOMKilled** (*Out Of Memory*).

Concernant le CPU :
- 1 CPU correspond à 1 vCPU (par exemple un vCPU AWS, un core GCP ou Azure).
- Une valeur peut être exprimée en fraction, ex. `0.1` CPU = `100m` (milliCPU).

Concernant la mémoire :
- Les suffixes diffèrent : `G` (1000 Mo, gigaoctet) vs. `Gi` (1024 Mi, gibibyte). Il est donc important de bien choisir l’unité adaptée.

### Exemple (Pod avec Requests et Limits)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: web
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```
Ici :
- Kubernetes réserve **250m CPU** et **256Mi mémoire** pour ce conteneur.
- Le conteneur pourra monter jusqu’à **500m CPU** et **512Mi mémoire**, mais pas plus.

***

### 1. Pas de Request, pas de Limit
#### Comportement
- Aucun Pod n’a une garantie de ressources.
- Tous les Pods peuvent consommer autant de CPU/mémoire que possible.
- Le Scheduler ne sait pas combien chaque Pod a besoin.

#### Exemple
- Pod A fait un calcul intensif et prend **90% du CPU**.
- Pod B et Pod C n’ont plus assez de cycles CPU → latence élevée.
- Si c’est de la mémoire, Pod A peut prendre *toute la RAM*, et Kubernetes devra tuer aléatoirement d’autres Pods (*OOMKilled*).

#### Avantages
- Simple configuration.
- Les Pods “opportunistes” utilisent toute la capacité disponible.

#### Inconvénients
- Très risqué : aucun isolement, risque de famine (*starvation*) pour les autres Pods.
- Pas adapté à un environnement multi-équipes.

***

### 2. Limit mais pas de Request
#### Comportement
- Kubernetes fixe automatiquement la **Request = Limit**.
- Chaque Pod a une allocation fixe de ressources, sans flexibilité.

#### Exemple
- Pod A a `Limit = 1 CPU`.
- Même si Pod B et Pod C ne consomment rien, Pod A ne pourra jamais dépasser **1 CPU**.

#### Avantages
- Ressources plus prévisibles.
- Permet de contenir la consommation d’un Pod “gourmand”.

#### Inconvénients
- **Peu flexible** : les ressources libres ne peuvent pas être utilisées par un Pod qui en aurait besoin.
- Peut empêcher une application d’exploiter le Node efficacement.

***

### 3. Requests et Limits définies
#### Comportement
- Chaque Pod dispose d’une **quantité garantie** (Request).
- Mais ne peut pas dépasser sa Limit.

#### Exemple
- Pod A : `Request=500m`, `Limit=1000m`
- Pod B : `Request=200m`, `Limit=500m`
- Pod C : `Request=200m`, `Limit=500m`  
  → Le Scheduler garantit que ces Pods auront au moins 900m de CPU disponibles au total.  
  → Aucun Pod ne peut dépasser ses limites fixées.

#### Avantages
- Bon compromis entre isolation et prévisibilité.
- Les applications sensibles ont une garantie de performance minimale.
- Pas de risque qu’un Pod affame les autres.

#### Inconvénients
- Comme pour le scénario précédent : si un Pod a besoin de plus de ressources et qu’elles sont disponibles, il ne pourra pas les utiliser au-delà de sa Limit.

***

### 4. Requests définies, pas de Limits
#### Comportement
- Chaque Pod a une garantie minimale de ressources (Request).
- Mais pas de plafond : un Pod peut consommer davantage si des ressources libres existent.

#### Exemple
- Pod A : `Request=500m`
- Pod B : `Request=200m`
- Pod C : `Request=200m`  
  → Chaque Pod a une garantie minimale de CPU.  
  → Si Pod A a besoin de plus de CPU et que Pod B/C ne consomment pas tout, Pod A peut exploiter la capacité restante.

#### Avantages
- Très flexible et efficace : les Pods utilisent toutes les ressources disponibles sans gaspillage.
- Bonne utilisation de la puissance du Node.

#### Inconvénients
- Si un Pod devient trop gourmand, il peut impacter négativement les autres (sauf leur Request minimale qui reste garantie).
- Pour la **mémoire**, c’est dangereux : comme il n’y a pas de Limit, un Pod peut consommer toute la RAM → Kubernetes devra le tuer (OOMKilled).

***
***

## LimitRange : Définir des standards et contraintes par conteneur dans un Namespace

### Qu’est-ce que LimitRange ?
- Objet Kubernetes appliqué **au niveau d’un Namespace**.
- Permet de définir des **valeurs par défaut** pour les **Requests** (demandes) et **Limits** (limites) CPU/mémoire des conteneurs si elles ne sont pas précisées dans les Pods.
- Permet aussi d’imposer des **bornes minimales et maximales** sur ces valeurs pour éviter des configurations extrêmes ou erronées.
- Agit au moment de la création du Pod dans le Namespace.

### Exemple LimitRange
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev-team
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    max:
      cpu: 1
      memory: 1Gi
    min:
      cpu: 100m
      memory: 128Mi
```
- Si un Pod est créé sans spécifier ses ressources, il héritera de ces valeurs par défaut.
- Un conteneur ne pourra pas demander moins de 100m CPU ni plus de 1 CPU, et pareil pour la mémoire.

### Quand utiliser LimitRange ?
- Pour **imposer une discipline** d’attribution des ressources au sein d’un Namespace.
- Pour **éviter que des Pods sans ressources définies** monopolisent ou saturent un Node.
- Pour **standardiser les configurations** au sein d’une équipe ou d’un environnement.
- Souvent utilisé en complément de ResourceQuota pour une gouvernance fine.

***
***

## ResourceQuota : Limiter la consommation globale des ressources dans un Namespace

### Qu’est-ce que ResourceQuota ?
- Objet Kubernetes appliqué **au niveau d’un Namespace**.
- Sert à fixer des **plafonds globaux** sur la consommation totale des ressources au sein du Namespace, par exemple le total CPU/mémoire demandée ou limitée par tous les Pods confondus.
- Peut aussi limiter le nombre total d’objets (Pods, Services, ConfigMaps, etc.).
- Empêche un Namespace d’outrepasser un quota défini, garantissant une **répartition équitable des ressources** à l’échelle du Cluster.

### Exemple ResourceQuota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: dev-team
spec:
  hard:
    requests.cpu: "2"       # Total CPU minimum demandée dans le Namespace
    requests.memory: 4Gi    # Total mémoire minimum demandée
    limits.cpu: "4"         # Total CPU maximum limitée
    limits.memory: 8Gi      # Total mémoire maximum limitée
    pods: "10"              # Max 10 Pods dans le Namespace
```
- Le Namespace `dev-team` ne pourra pas dépasser 2 CPU prescrits au total et 4 Gi de mémoire demandée par tous ses Pods cumulés.
- Empêche des déploiements ou la création de Pods qui dépasseraient ces limites globales.

### Quand utiliser ResourceQuota ?
- Dans des Cluster partagés par **plusieurs équipes ou projets**.
- Pour **éviter qu’une équipe monopole toutes les ressources** du Cluster.
- Pour imposer une **quarantine des ressources globales** par environnement (ex : test vs prod).
- Souvent combiné avec LimitRange pour assurer une gestion granulaire des ressources (par Pod) et une limite globale (par Namespace).

***

## Résumé concis

- Les Pods demandent des ressources (CPU, mémoire) définies via **Requests** (minimum garanti) et **Limits** (maximum autorisé).
- Le **Scheduler** place les Pods sur les Nodes en fonction des Requests et des ressources disponibles.
- **CPU** : limité par Kubernetes (throttling). **Mémoire** : dépassement → Pod tué (*OOMKilled*).
- Scénarios :
    - Sans Request/Limit → risque de monopolisation des ressources.
    - Avec Limit seule → la Request est alignée sur la Limit.
    - Avec Request et Limit → ressources garanties et plafonnées.
    - Avec Request seule → flexibilité, ressources garantées mais usage libre au-delà.
- **LimitRange** : définit requests/limits par défaut dans un Namespace.
- **ResourceQuota** : fixe une limite globale de consommation CPU/mémoire pour tous les Pods d’un Namespace.
