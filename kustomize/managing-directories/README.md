### Organisation et gestion de manifests Kubernetes avec **Kustomize**

Jusqu’à présent, nous n’avons exploré que les bases d’un fichier **kustomization.yaml**.\
Pourtant, même avec cette connaissance limitée, **Kustomize** nous permet déjà de faire des choses puissantes et pratiques, notamment pour **gérer des manifests Kubernetes répartis sur plusieurs répertoires**.

---

### Exemple initial : structure simple sans Kustomize

Imaginons que nous ayons un répertoire `k8s/` contenant quatre fichiers YAML :

```
k8s/
├── api-deployment.yaml
├── api-service.yaml
├── db-deployment.yaml
└── db-service.yaml
```

Pour déployer ces ressources, il suffit d’exécuter :

```bash
kubectl apply -f k8s/
```

C’est la méthode standard avec Kubernetes, sans intervention de **Kustomize**.

---

### Réorganisation en sous-répertoires

Avec le temps, le nombre de fichiers peut croître (20, 30, 50, etc.), rendant le répertoire difficile à gérer.\
Pour plus de clarté, nous décidons de regrouper les fichiers par composant :

```
k8s/
├── api/
│   ├── deployment.yaml
│   └── service.yaml
└── database/
    ├── deployment.yaml
    └── service.yaml
```

Pour appliquer ces configurations, il faut désormais exécuter deux commandes :

```bash
kubectl apply -f k8s/api/
kubectl apply -f k8s/database/
```

Cela fonctionne, mais devient vite fastidieux lorsque le nombre de sous-répertoires augmente (API, base de données, cache, Kafka, etc.).\
Chaque fois qu’on modifie les manifests, il faut appliquer manuellement chaque sous-dossier, ou adapter les pipelines CI/CD pour le faire automatiquement.

---

### Simplification avec **Kustomize**

Pour résoudre ce problème, on crée un fichier **kustomization.yaml** à la racine du répertoire `k8s/` :

```
k8s/
├── kustomization.yaml
├── api/
│   ├── deployment.yaml
│   └── service.yaml
└── database/
    ├── deployment.yaml
    └── service.yaml
```

#### Exemple de `kustomization.yaml` à la racine :

```yaml
resources:
  - api/deployment.yaml
  - api/service.yaml
  - database/deployment.yaml
  - database/service.yaml
```

Désormais, il suffit d’une seule commande pour déployer toutes les ressources :

```bash
kubectl kustomize k8s/ | kubectl apply -f -
```

ou, directement avec **kubectl** (intègre Kustomize nativement) :

```bash
kubectl apply -k k8s/
```

Kustomize lit le fichier `kustomization.yaml`, récupère tous les manifests listés, les assemble et les applique dans le cluster.
Nous n’avons plus besoin de naviguer dans chaque sous-répertoire.

---

### Structuration avancée avec plusieurs `kustomization.yaml`

Lorsque le projet grandit, le fichier racine peut devenir très long :

```
k8s/
├── api/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── database/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── cache/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── kafka/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── kustomization.yaml
```

Chaque sous-répertoire contient désormais son propre fichier **kustomization.yaml** :

#### Exemple dans `k8s/api/kustomization.yaml` :

```yaml
resources:
  - deployment.yaml
  - service.yaml
```

Et dans `k8s/database/kustomization.yaml` :

```yaml
resources:
  - deployment.yaml
  - service.yaml
```

Le fichier **racine** devient alors plus simple :

```yaml
resources:
  - api/
  - database/
  - cache/
  - kafka/
```

Désormais, **Kustomize** comprend que chaque répertoire contient son propre `kustomization.yaml`.
Lors de l’exécution, il va automatiquement **récupérer et combiner** toutes les ressources listées dans ces fichiers.

#### Déploiement unique :

```bash
kubectl apply -k k8s/
```

Cette approche hiérarchique rend la configuration **plus lisible, modulaire et maintenable**.

---

### Avantages

✅ Déploiement simplifié avec une seule commande\
✅ Structure modulaire et évolutive\
✅ Intégration native avec **kubectl**\
✅ Facilite la maintenance et les pipelines CI/CD

---

## Résumé concis

* **Problème initial :** gérer de nombreux manifests YAML dispersés dans plusieurs répertoires.
* **Solution simple :** création d’un fichier `kustomization.yaml` listant toutes les ressources.
* **Problème suivant :** fichier racine trop long et difficile à maintenir.
* **Solution avancée :** un `kustomization.yaml` par sous-répertoire (API, DB, cache, Kafka…), et un fichier racine qui les référence.
* **Commandes clés :**

  ```bash
  kubectl kustomize k8s/ | kubectl apply -f -
  kubectl apply -k k8s/
  ```
* **Résultat :** configuration hiérarchique, claire, modulaire et facile à maintenir.

