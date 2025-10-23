# Helm

## 1. Ce que Helm **n’est pas**

* Helm **ne collecte pas de metrics**.
  Il ne remplace **ni Prometheus**, **ni Grafana**, ni aucun outil d’observabilité.
  → Son rôle n’est **pas de monitorer**, mais de **déployer et gérer** des applications Kubernetes.

---

## 2. Ce que tu fais aujourd’hui sans Helm

Tu as sûrement :

* Des fichiers YAML : `deployment.yaml`, `service.yaml`, `ingress.yaml`, `pvc.yaml`, etc.
* Tu les déploies avec `kubectl apply -f .`
* Tu changes les valeurs à la main (ou avec `envsubst`, `kustomize`, etc.)
* Tu fais des rollback manuels en réappliquant d’anciennes versions de YAML.

C’est parfaitement valide !
Mais… dès que ton projet grandit, ça devient vite **difficile à maintenir et à versionner**.

---

## 3. Ce que Helm **apporte réellement**

### a. Un moteur de **templating puissant**

Helm te permet de transformer tes YAML statiques en **modèles dynamiques**.
Tu définis des variables dans `values.yaml` (version, nombre de replicas, taille de PV, image Docker, etc.) et Helm les injecte dans les templates.

Exemple :

```yaml
# deployment.yaml (template Helm)
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Puis dans ton `values.yaml` :

```yaml
replicaCount: 3
image:
  repository: myapp
  tag: v1.2.0
```

Avec ça, tu peux facilement changer l’environnement :

* Dev : `replicaCount: 1`
* Prod : `replicaCount: 5`
  → sans modifier le YAML de base.

---

### b. Une **gestion de versions et de releases**

Helm crée un **release** à chaque installation ou mise à jour.
Tu peux ensuite :

* `helm upgrade myapp .` → mettre à jour ton déploiement
* `helm rollback myapp 1` → revenir à une version précédente
  (Helm garde l’historique dans un **Secret** Kubernetes)

→ Tu n’as plus besoin de réappliquer manuellement des anciens YAML.

---

### c. Un système de **packaging et de partage**

Helm te permet de **packager** toute ton appli dans un **chart** (`.tgz`) et de la publier dans un **Helm repository**.

Exemple :

* Tu crées ton chart `myapp`
* Tu le packages : `helm package myapp`
* Tu le pousses dans un repo interne ou public

N’importe qui peut ensuite faire :

```
$ helm repo add myrepo https://myrepo.example.com
$ helm install myapp myrepo/myapp
```

En un seul `helm install`, tout est déployé proprement.

---

### d. Une **gestion des dépendances**

Si ton application dépend d’autres services (ex: Redis, PostgreSQL, Nginx Ingress…),
Helm peut automatiquement les déployer via un simple fichier `Chart.yaml` :

```yaml
dependencies:
  - name: postgresql
    version: 12.1.2
    repository: https://charts.bitnami.com/bitnami
```

→ Tu n’as pas besoin d’écrire ou maintenir leurs manifests toi-même.

---

### e. Intégration CI/CD et facilité de mise à jour

Helm est parfait dans un pipeline CI/CD :

* `helm upgrade --install` → déploie ou met à jour sans casser l’existant
* Tu peux passer des variables via `--set` ou un fichier values spécifique (`values-prod.yaml`)

---

## 4. Donc pourquoi Helm ?

Tu peux très bien tout faire sans Helm, **mais Helm te simplifie la vie** dès que :

* Tu as plusieurs environnements (dev, staging, prod)
* Tu veux automatiser les déploiements
* Tu veux versionner proprement tes releases
* Tu veux partager facilement ton application

Helm = **YAML + Variables + Versioning + Reproducibilité + Simplicité**

---

## 5. Si j’ai déjà mon architecture Kubernetes ? :

> « Si j’ai déjà mon architecture Kubernetes, puis-je faire un `helm install`, `upgrade` ou `rollback` dessus ? »

👉 Oui, mais **tu dois d’abord la transformer en chart Helm**.
Tu ne peux pas directement faire un `helm install` sur des YAML bruts.
Il faut que ces fichiers soient **dans la structure Helm** :

```
mychart/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
```

Après, tu pourras faire :

```
$ helm install myapp ./mychart
$ helm upgrade myapp ./mychart
$ helm rollback myapp 1
```

---

## En résumé :

| Sans Helm                    | Avec Helm                       |
| ---------------------------- | ------------------------------- |
| YAML statiques               | YAML dynamiques (templating)    |
| `kubectl apply`              | `helm install/upgrade/rollback` |
| Pas de versioning de release | Versioning automatique          |
| Difficile à partager         | Packaging (chart) facile        |
| Tout manuel                  | Automatisation et réutilisation |

---

# Commande utile 

## 1. `helm search repo wordpress`

**But :**
Chercher une chart disponible dans les dépôts Helm configurés localement.
En d'autres termes : “Montre-moi les charts qui contiennent le mot *wordpress*.”

**Exemple de commande :**

```bash
helm search repo wordpress
```

**Exemple de sortie :**

```
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/wordpress       18.0.7          6.6.2           WordPress is the world’s most popular blogging and CMS platform.
```

**Explication :**

* `NAME` : nom complet de la chart (`repo/chartname`)
* `CHART VERSION` : version du modèle Helm
* `APP VERSION` : version réelle de l’application
* `DESCRIPTION` : description courte de la chart

---

## 2. `helm repo list`

**But :**
Lister les dépôts Helm configurés localement (comme les dépôts APT ou YUM sous Linux).

**Exemple de commande :**

```bash
helm repo list
```

**Exemple de sortie :**

```
NAME   	URL
bitnami	https://charts.bitnami.com/bitnami
```

**Explication :**

* `NAME` : nom du dépôt
* `URL` : adresse du dépôt Helm

Tu peux ajouter un dépôt avec :

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

et mettre à jour les index avec :

```bash
helm repo update
```

---

## 3. `helm install bravo bitnami/drupal`

**But :**
Installer la chart **Drupal** depuis le dépôt **bitnami** et lui donner le nom de release **bravo**.

**Exemple de commande :**

```bash
helm install bravo bitnami/drupal
```

**Exemple de sortie :**

```
NAME: bravo
LAST DEPLOYED: Thu Oct 23 12:01:15 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: drupal
CHART VERSION: 15.2.6
APP VERSION: 10.2.2
** Please be patient while the chart is being deployed **
Drupal can be accessed through the following DNS name from within your cluster:
    bravo-drupal.default.svc.cluster.local (port 80)
```

**Explication :**

* Helm télécharge la chart `bitnami/drupal`
* Il rend les templates YAML
* Il applique les manifests Kubernetes dans ton cluster
* Il enregistre cette installation sous le nom `bravo`

---

## 4. `helm list`

**But :**
Lister toutes les releases Helm installées dans ton cluster.

**Exemple de commande :**

```bash
helm list
```

**Exemple de sortie :**

```
NAME   	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART         	APP VERSION
bravo  	default  	1        	2025-10-23 12:01:15.123456 +0000 UTC  	deployed	drupal-15.2.6 	10.2.2
```

**Explication :**

* `NAME` : nom de la release (`bravo`)
* `NAMESPACE` : namespace dans lequel elle est installée
* `REVISION` : numéro de version de la release
* `STATUS` : état actuel (`deployed`, `failed`, etc.)

Tu peux voir toutes les releases dans tous les namespaces avec :

```bash
helm list -A
```

---

## 5. `helm uninstall bravo`

**But :**
Supprimer proprement la release `bravo` et toutes les ressources Kubernetes qu’elle a créées.

**Exemple de commande :**

```bash
helm uninstall bravo
```

**Exemple de sortie :**

```
release "bravo" uninstalled
```

**Explication :**

* Helm supprime tous les objets Kubernetes associés à la release (`Deployment`, `Service`, `ConfigMap`, etc.)
* Il efface aussi les informations de la release de son registre interne

---

## Résumé

| Commande                            | Action                                         | Exemple de sortie                            | Description                                  |
| ----------------------------------- | ---------------------------------------------- | -------------------------------------------- | -------------------------------------------- |
| `helm search repo wordpress`        | Recherche une chart dans les dépôts configurés | `bitnami/wordpress`                          | Trouve la chart WordPress disponible         |
| `helm repo list`                    | Liste les dépôts Helm locaux                   | `bitnami https://charts.bitnami.com/bitnami` | Vérifie les dépôts ajoutés                   |
| `helm install bravo bitnami/drupal` | Installe la chart Drupal sous le nom `bravo`   | `STATUS: deployed`                           | Déploie les ressources Kubernetes            |
| `helm list`                         | Liste les releases installées                  | `bravo  deployed  drupal-15.2.6`             | Vérifie les déploiements Helm actifs         |
| `helm uninstall bravo`              | Supprime la release `bravo`                    | `release "bravo" uninstalled`                | Désinstalle proprement toutes les ressources |

---
