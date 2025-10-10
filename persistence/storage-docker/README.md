## Gestion du stockage dans Docker

### Structure par défaut du stockage Docker

Lorsque vous installez Docker sur un hôte, il crée par défaut le répertoire suivant :  
`/var/lib/docker`

À l’intérieur, on trouve plusieurs sous-répertoires, par exemple :

- `aufs/` ou `overlay2/` : stockage interne des couches d’images.
- `containers/` : données spécifiques aux containers.
- `image/` : métadonnées des images.
- `volumes/` : données persistantes créées via les volumes.

Exemple :
- Les fichiers liés aux containers sont stockés dans `/var/lib/docker/containers/`.
- Les images se trouvent sous `/var/lib/docker/image/`.
- Les volumes persistants sont placés dans `/var/lib/docker/volumes/`.

***

### Architecture en couches (Layered architecture)

Docker adopte une **architecture en couches** pour les images.  
Chaque instruction dans un `Dockerfile` crée une nouvelle couche contenant *uniquement les changements* par rapport à la couche précédente.

Exemple concret :

```Dockerfile
FROM ubuntu:20.04         # Base image (~120MB)
RUN apt-get install -y python3-pip   # Packages APT (~300MB)
RUN pip install flask     # Dépendances Python
COPY src/ /app            # Code source applicatif
ENTRYPOINT ["python3", "/app/app.py"]
```

Lors de la construction (`docker build`), Docker :

1. Crée une couche pour **Ubuntu**.
2. Ajoute une couche pour les **packages APT**.
3. Ajoute une couche pour **Flask et dépendances Python**.
4. Copie le **code source** (nouvelle couche).
5. Met à jour **l’ENTRYPOINT** (dernière couche).

**Avantages :**
- Si un autre projet utilise exactement les trois premières couches, Docker les réutilise depuis le cache, ne reconstruisant que les nouvelles couches.
- Gain de temps et d’espace disque.

***

### Couches en lecture seule et Copy-on-Write

Les couches d’**image** sont **read-only**.  
Lorsqu’un container est démarré (`docker run`), Docker :

- Monte toutes les couches de l’image.
- Ajoute au-dessus **une couche writable (container layer)**.

Si vous modifiez un fichier du container provenant d’une couche read-only, Docker applique le mécanisme **Copy-on-Write** :

1. Le fichier original est copié dans la couche writable.
2. Les modifications s’effectuent uniquement sur cette copie.

**Durée de vie** : La couche writable disparaît avec le container. Toutes les modifications sont perdues si elles ne sont pas externalisées dans un volume.

Exemple :
```
$ docker run -it ubuntu bash
$ touch /tmp/test.txt   # Fichier créé dans la couche writable
```

Une fois le container supprimé (`docker rm`), le fichier disparaît.

***

### Volumes et persistance des données

Pour **préserver les données au-delà de la durée de vie d’un container**, on utilise les **volumes**.

#### Création et montage d’un volume
```
$ docker volume create data_volume
$ docker run -v data_volume:/var/lib/mysql mysql:8
```
- Les données MySQL sont alors stockées dans `/var/lib/docker/volumes/data_volume/_data/`.

#### Création implicite d’un volume
Si le volume n’existe pas :
```
$ docker run -v data_volume2:/var/lib/mysql mysql:8
```
Docker le crée automatiquement.

***

### Bind mounts

Un **bind mount** permet de monter un répertoire spécifique de l’hôte dans un container.

Exemple :
```
$ docker run -v /data/mysql:/var/lib/mysql mysql:8
```
- Les données sont directement stockées dans `/data/mysql` sur l’hôte.

***

### Syntaxe moderne : `--mount`

L’usage de `-v` est ancien.  
Préférez `--mount` pour plus de clarté :
```
$ docker run --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql:8
```

***

### Storage drivers

Les **storage drivers** gèrent l’architecture en couches et les opérations Copy-on-Write.  
Parmi les plus fréquents :

- `aufs` (ancien, surtout sur Ubuntu)
- `overlay2` (par défaut sur Linux récents)
- `devicemapper`
- `btrfs`
- `zfs`

Docker choisit automatiquement le meilleur driver disponible selon l’OS.

***

### Volume drivers

Les **volumes** ne sont pas gérés par les storage drivers, mais par les **volume driver plugins**.  
Par défaut : `local`.

Autres exemples :
- AWS EBS
- Azure File Storage
- Google Persistent Disk
- NetApp
- RexRay (permet stockage sur AWS EBS, S3, EMC, OpenStack Cinder...)

Exemple d’utilisation avec AWS EBS via RexRay :
```
$ docker run \
  --volume-driver=rexray/ebs \
  -v myebsvolume:/var/lib/mysql \
  mysql:8
```

***

## Résumé essentiel

- Docker stocke ses données par défaut dans `/var/lib/docker`.
- Les **images** sont construites en **couches read-only**, et les containers ajoutent une **couche writable temporaire**.
- Le mécanisme **Copy-on-Write** permet de modifier un fichier sans altérer l’image originale.
- Les **volumes** permettent de rendre les données persistantes au-delà du cycle de vie du container.
- **Mount types** :
    - Volume mount : `docker run -v nom_volume:/path/container`
    - Bind mount : `docker run -v /path/host:/path/container`
    - Syntaxe moderne : `--mount type=...`
- Les **storage drivers** (overlay2, aufs, btrfs...) gèrent le stockage interne des images et containers.
- Les **volume drivers** (local, RexRay, etc.) permettent de stocker sur l’hôte ou sur des solutions externes comme AWS, Azure, Google Cloud.


