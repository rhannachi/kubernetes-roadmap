### Helm comme alternative à Kustomize

Helm est une alternative à **Kustomize** pour la gestion et la personnalisation des manifestes **Kubernetes** selon les environnements (dev, staging, prod)\
Comprendre le fonctionnement de ces deux outils permet de choisir celui le plus adapté à votre projet.

***

### Principe de fonctionnement de Helm

Helm repose sur une logique de **templating** utilisant la syntaxe Go. Au lieu de définir des valeurs statiques dans vos manifestes, vous déclarez des **variables** qui seront remplacées dynamiquement lors du déploiement.

#### Exemple simple de Template Helm

Fichier `templates/deployment.yaml` :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: myapp
          image: nginx:{{ .Values.image.tag }}
```

Fichier `values.yaml` :
```yaml
replicaCount: 1
image:
  tag: "2.4.4"
```

Lors du déploiement, Helm insère automatiquement les valeurs du fichier `values.yaml`, ce qui permet de gérer dynamiquement la configuration par environnement.

***

### Gestion multi‑environnements avec Helm

Pour adapter les valeurs selon l’environnement, on crée plusieurs fichiers `values.yaml` spécifiques :

```
mychart/
├── templates/
│   └── deployment.yaml
├── values.dev.yaml
├── values.staging.yaml
└── values.prod.yaml
```

- `values.dev.yaml` : configuration pour le développement
- `values.staging.yaml` : configuration pour la pré‑production
- `values.prod.yaml` : configuration pour la production

Lors du déploiement, vous indiquez le fichier de valeurs à utiliser :

```bash
helm install myapp . -f values.dev.yaml
```

Cette commande applique le fichier `values.dev.yaml` et génère les manifestes finaux valides YAML avant de les soumettre au **Cluster** Kubernetes.

***

### Helm comme gestionnaire de paquets

Helm ne sert pas uniquement à personnaliser les configurations. C’est un **package manager** pour les applications Kubernetes, comparable à **apt** ou **yum** sur Linux.  
Il permet :

- d’installer, mettre à jour ou supprimer des applications via des **charts** ;
- de gérer les dépendances entre services ;
- d’utiliser des fonctionnalités avancées : **conditionals**, **loops**, **functions**, et **hooks**.

***

### Points forts et limites

| Critère | Helm | Kustomize |
|----------|------|-----------|
| Simplicité de lecture | Moyen (syntaxe templating Go) | Excellente (YAML pur) |
| Puissance et flexibilité | Très élevée (fonctions, hooks, conditionnels) | Moyenne |
| Validation YAML native | Non (avant rendu) | Oui |
| Support multi‑environnement | Excellent (fichiers `values.yaml`) | Excellent (overlays) |

Helm apporte davantage de fonctionnalités mais implique une complexité supérieure. **Kustomize**, plus simple et déclaratif, reste idéal pour des configurations légères ou faciles à lire.

***

