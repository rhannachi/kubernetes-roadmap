### 1. Qu’est-ce qu’un nodeSelector et dans quel cas simple l’utiliser ?
Le **nodeSelector** est un mécanisme basique de Kubernetes qui permet de contraindre un Pod à s’exécuter uniquement sur des Nodes possédant un label spécifique (clé=valeur). Il est simple à configurer et adapté quand on veut imposer un placement strict sur un Node unique ou un groupe de Nodes identiques sans conditions complexes.

***

### 2. Qu’est-ce qu’une nodeAffinity et en quoi elle est plus flexible ?
La **nodeAffinity** est une fonctionnalité avancée qui permet de définir des contraintes de placement plus complexes en combinant plusieurs labels, en utilisant des opérateurs riches (In, NotIn, Exists, etc.) et en formulant des règles logiques (OU/ET). Elle permet aussi des règles préférentielles.

***

### 3. Différence fondamentale entre nodeSelector et nodeAffinity ?
**nodeSelector** ne supporte qu’une correspondance simple clé=valeur, tandis que **nodeAffinity** accepte plusieurs conditions combinées et des opérateurs multiples. nodeAffinity permet aussi exclusion et préférences, ce qui est impossible avec nodeSelector.

***

### 4. Différence entre requiredDuringSchedulingIgnoredDuringExecution et preferredDuringSchedulingIgnoredDuringExecution ?
- **requiredDuringSchedulingIgnoredDuringExecution** : règle obligatoire, le Pod ne sera programmé que sur des Nodes satisfaisant la règle. Si aucun Node ne correspond, le Pod reste Pending. Le changement de label en cours d’exécution n’affecte pas le Pod.
- **preferredDuringSchedulingIgnoredDuringExecution** : règle préférentielle, Kubernetes essaie de respecter la contrainte, mais si aucun Node ne correspond, le Pod sera lancé quand même sur un autre Node disponible.

***

### 5. Que se passe-t-il si aucun Node ne correspond à une règle obligatoire ?
Le Pod reste en état "Pending" et n’est pas lancé tant qu’un Node respectant la contrainte n’est pas disponible. Cela empêche le Pod de fonctionner dans un environnement inadapté.

***

### 6. Pourquoi requiredDuringSchedulingRequiredDuringExecution est-il mentionné et non disponible ?
Cette option stricte garantit que le Pod doit toujours être sur un Node respectant la règle, même si le label du Node change après démarrage. Si la règle cesse d’être respectée, le Pod est éjecté puis reprogrammé. Elle est en développement car cela nécessite une gestion fine du cycle de vie du Pod et peut provoquer des interruptions.

***

### 7. Quand privilégier nodeSelector par rapport à nodeAffinity ?
Privilégier **nodeSelector** pour des besoins simples et statiques, quand on veut juste forcer un Pod sur un Node avec un label unique, sans logique multiple. Utiliser **nodeAffinity** quand des règles complexes, des préférences ou exclusions sont nécessaires.

***

### 8. Quand utiliser une affinité obligatoire plutôt qu’une affinité préférentielle ?
Utiliser une affinité obligatoire quand le placement sur un Node avec certaines caractéristiques est impératif (ex. Pod nécessitant un GPU spécifique). Utiliser une affinité préférentielle quand on souhaite favoriser un certain type de Node sans bloquer l’exécution si ce type n’est pas disponible (ex. preferer un Node SSD mais pas obligatoire).

***

### 9. Comment demander un Pod sur Node avec gpu=nvidia ET disktype=ssd avec nodeAffinity ?
On utilise plusieurs expressions dans `matchExpressions` sous un seul `nodeSelectorTerms` avec les deux conditions :

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: gpu
      operator: In
      values:
      - nvidia
    - key: disktype
      operator: In
      values:
      - ssd
```
Cela impose que les deux conditions soient remplies simultanément (ET logique).

***

### 10. Quel mécanisme choisir pour un Pod critique GPU, puis un Pod préférant SSD ?
- Pour le Pod critique nécessitant absolument un GPU spécifique : utiliser **nodeAffinity** avec `requiredDuringSchedulingIgnoredDuringExecution` pour forcer le placement strict.
- Pour le Pod préférant un Node SSD mais pouvant être flexible : utiliser **nodeAffinity** avec `preferredDuringSchedulingIgnoredDuringExecution` pour privilégier le SSD sans bloquer le lancement.
