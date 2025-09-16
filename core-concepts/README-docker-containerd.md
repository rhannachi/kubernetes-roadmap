# Docker VS Containerd

### Historique et compatibilité
Au début, Kubernetes ne supportait que **Docker** comme runtime de conteneur. Avec la montée en popularité de Kubernetes, d’autres runtimes comme rkt ont voulu être pris en charge. Pour cela, Kubernetes a créé la **Container Runtime Interface (CRI)**, une norme permettant à tout runtime conforme à l’**Open Container Initiative (OCI)** d’être utilisé.
- L’**OCI ImageSpec** définit comment construire une image.
- L’**OCI RuntimeSpec** définit comment exécuter un conteneur.

Docker n’étant pas compatible nativement avec CRI, Kubernetes a créé un composant temporaire appelé **dockershim** pour assurer la compatibilité. Depuis la version 1.24, dockershim a été supprimé, ce qui signifie que Docker n’est plus un runtime pris en charge de manière native. Cependant, toutes les images Docker continuent de fonctionner parfaitement, car elles respectent la norme OCI.

***

### De Docker à containerd
Docker est en réalité un ensemble de composants : une CLI, une API, des outils pour construire des images, la gestion des volumes, la sécurité… Au cœur de cette architecture se trouve **containerd**, responsable de la gestion des conteneurs via **runc**.  
Aujourd’hui, containerd est un projet indépendant, membre de la CNCF, et peut être installé sans Docker. Kubernetes utilise désormais directement containerd (ou d’autres runtimes compatibles CRI) comme moteur d’exécution, sans passer par Docker.

***

### Les outils CLI associés
Avec containerd et Kubernetes, trois outils en ligne de commande principaux sont utilisés :

#### ctr
C’est l’outil livré avec containerd. Il est bas niveau, principalement destiné au débogage et aux tests, avec peu de fonctionnalités. Il n’est pas fait pour un usage en production. Par exemple, on peut l’utiliser avec des commandes comme `ctr images pull` ou `ctr run`.

#### nerdctl
C’est un projet de la communauté containerd, qui propose une CLI très proche de Docker. Elle permet d’utiliser containerd avec une syntaxe familière (`nerdctl run`, `nerdctl logs`, etc.) et offre même des fonctionnalités avancées comme les images chiffrées, la distribution en P2P, la signature d’images, et la gestion des namespaces Kubernetes. C’est l’outil recommandé pour remplacer Docker au quotidien.

#### crictl
Développé par la communauté Kubernetes, crictl fonctionne avec tous les runtimes compatibles CRI, pas seulement containerd. C’est un outil de débogage et d’inspection des conteneurs et pods gérés par Kubernetes. Sa syntaxe ressemble beaucoup à celle de Docker (`crictl ps`, `crictl exec`, etc.). Par contre, il n’est pas conçu pour gérer directement des charges de travail, car kubelet supprime les conteneurs qu’il ne connaît pas.

***

### Comparaison des CLI

| CLI     | Origine         | Compatibilité        | Usage principal              | Particularités                         |  
|---------|-----------------|---------------------|-----------------------------|--------------------------------------|  
| ctr     | containerd      | containerd uniquement | Debug bas niveau            | Fonctionnalités limitées, accès direct à l’API |  
| nerdctl | containerd      | containerd uniquement | Gestion des conteneurs      | CLI Docker-like avec extras          |  
| crictl  | Kubernetes      | Tous runtimes CRI    | Debug & inspection Kubernetes | Support des pods et conteneurs, syntaxe Docker-like |  

***

### En résumé
Docker n’est plus supporté comme runtime par Kubernetes depuis la version 1.24, remplacé par containerd ou d’autres runtimes compatibles CRI.  
Les outils ctr et crictl servent surtout au débogage, tandis que nerdctl est le remplaçant pratique de la CLI Docker pour travailler avec containerd.

***

### Pourquoi parle-t-on encore de Docker ?
Docker a longtemps été le runtime par défaut dans Kubernetes. Il cumule bien plus que le simple runtime de conteneur : CLI, API, outils de construction, gestion des volumes… Le runtime sous-jacent est containerd, qui fonctionne désormais directement avec Kubernetes, sans Docker.

Docker reste très utilisé dans le développement et la création d’images, mais Kubernetes n’en a plus besoin pour exécuter les conteneurs. C’est pourquoi Docker est déprécié comme runtime dans Kubernetes, sans pour autant disparaître.

