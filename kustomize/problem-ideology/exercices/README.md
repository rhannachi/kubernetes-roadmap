## Exemple complet "kustomize-example"
Voici un exemple complet nommé `kustomize-example`, structuré pour illustrer une base et des overlays pour trois environnements (development, staging, production).

### Structure des dossiers

```
kustomize-example/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

### Commentaire explicatif de l’architecture Kustomize

- **Base**  
  Le dossier `base` contient toutes les ressources Kubernetes **communes à tous les environnements**. Ces fichiers YAML représentent la configuration par défaut, comme un `Deployment`, un `Service`, un `ConfigMap`, etc., qui vont être partagés.  
  Le fichier `kustomization.yaml` dans `base` référence ces ressources via le champ `resources`. Cette base sert de **point de départ unique et réutilisable** pour toutes les autres configurations spécifiques.

- **Overlays**  
  Le dossier `overlays` regroupe des sous-dossiers pour chaque environnement cible (ici : `development`, `staging`, `production`).  
  Chaque environnement a son propre fichier `kustomization.yaml` qui :
    - référence la base via le champ `resources` (le chemin relatif vers `base`)
    - applique des **patches** ou modifications spécifiques à cet environnement (ici, `replica-patch.yaml` modifiant le nombre de replicas) via le champ `patches` sous la forme `{ path: <fichier> }`
    - peut aussi ajouter des ressources spécifiques uniquement à cet environnement (exemple : un service ou configmap propre à la production).

- **Principe**  
  Kustomize applique la configuration commune définie dans la base, puis superpose les modifications définies dans l’overlay, générant ainsi un manifeste final personnalisé pour chaque environnement.  
  Cette méthode **évite la duplication du code**, rend les configurations plus maintenables, et simplifie la gestion multi-environnements.

- **Détail technique**  
  Le fichier `kustomization.yaml` dans chaque overlay utilise le champ `resources:` pour inclure la base (remplaçant l’ancienne notation `bases:`), puis utilise `patches:` avec la syntaxe `- path: fichier.yaml` pour définir les modifications locales, qui peuvent être :
    - Changement de valeurs (ex : nombre de replicas)
    - Ajout ou suppression de ressources
    - Modifications de labels, annotations, images, etc.

- **Flexibilité**  
  La structure base/overlay est très flexible et peut s’adapter à des projets complexes : on peut imbriquer plusieurs bases, créer plusieurs niveaux d’overlays (features, régions, etc.).

***

### Commandes utiles

- Voir la composition finale du manifeste (exemple : staging)

```bash
kubectl kustomize overlays/staging
```

- Appliquer un environnement

```bash
kubectl apply -k overlays/production
```

***


