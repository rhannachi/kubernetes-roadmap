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

