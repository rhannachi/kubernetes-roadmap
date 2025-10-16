## Kubeconfig

### Introduction
Dans cette section, nous allons explorer le fonctionnement du **Kubeconfig** dans **Kubernetes**.
Jusqu’à présent, nous avons vu comment générer un certificat pour un utilisateur et comment utiliser ce certificat (ainsi que sa clé privée) pour interagir avec l’**API Server** via une requête **curl**.

Par exemple, pour récupérer la liste des Pods d’un cluster nommé `minikube-playground`, on peut exécuter :

```
$ curl https://<API_SERVER_ADDRESS>:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

L’API Server valide ensuite la requête en vérifiant le certificat client.

### Utilisation avec kubectl
Plutôt que d’utiliser `curl`, nous utilisons l’outil **kubectl**, qui interagit lui aussi avec le **kube-apiserver**.\
Saisir à chaque fois les options `--server`, `--client-key`, `--client-certificate` et `--certificate-authority` serait fastidieux.

Pour simplifier cela, Kubernetes stocke ces informations dans un fichier de configuration appelé **kubeconfig**.\
Par défaut, `kubectl` cherche ce fichier dans le chemin suivant :

```
~/.kube/config
```

Si le fichier est présent à cet emplacement, il n’est pas nécessaire de le spécifier avec l’option `--kubeconfig`.

***

### Structure du fichier kubeconfig
Le fichier **kubeconfig** est au format **YAML** et contient trois sections principales :

- **clusters** → décrit les clusters accessibles.
- **users** → définit les identités (certificats, clés, tokens, etc.) permettant l’accès.
- **contexts** → associe un utilisateur à un cluster et, facultativement, à un namespace.

Il contient également deux champs globaux :

```yaml
apiVersion: v1
kind: Config
```

#### Exemple complet de kubeconfig

```yaml
apiVersion: v1
kind: Config

clusters:
  - name: mykube-playground
    cluster:
      server: https://192.168.49.2:8443
      certificate-authority: /home/user/.minikube/ca.crt

users:
  - name: mykube-admin
    user:
      client-certificate: /home/user/.minikube/profiles/minikube/client.crt
      client-key: /home/user/.minikube/profiles/minikube/client.key

contexts:
  - name: mykube-admin@mykube-playground
    context:
      cluster: mykube-playground
      user: mykube-admin
      namespace: default

current-context: mykube-admin@mykube-playground
```

#### Explication
- Le cluster `mykube-playground` correspond à notre environnement Kubernetes (ici, un cluster Minikube).
- L’utilisateur `mykube-admin` représente notre identité locale avec ses certificats.
- Le context `mykube-admin@mykube-playground` relie l’utilisateur au cluster.
- Le champ `current-context` indique le context actif par défaut utilisé par `kubectl`.

***

### Gestion des contextes
Si vous avez plusieurs clusters ou utilisateurs (par exemple : dev, test, prod), vous pouvez basculer entre différents contextes avec :

```
$ kubectl config use-context prod-admin@production
```

Le changement est directement reflété dans le fichier `~/.kube/config` sous le champ `current-context`.

Pour afficher le contenu du kubeconfig actuel :

```
$ kubectl config view
```

***

### Configuration du namespace par défaut
Chaque cluster Kubernetes contient plusieurs **Namespaces**. Vous pouvez définir le namespace par défaut d’un context :

```yaml
contexts:
  - name: dev-user@staging
    context:
      cluster: staging-cluster
      user: dev-user
      namespace: frontend
```

Ainsi, toute commande `kubectl` exécutée sous ce context ciblera automatiquement le namespace `frontend`.

***

### Gestion des certificats
Les chemins vers les certificats peuvent être spécifiés de deux manières :

1. **Par chemin de fichier** (recommandé pour la clarté) :
   ```yaml
   certificate-authority: /path/to/ca.crt
   ```

2. **En ligne via des données encodées en Base64** :
   ```yaml
   certificate-authority-data: <donnees_base64>
   ```

Pour encoder un certificat en Base64 :

```bash
base64 ca.crt -w 0
```

Pour le décoder :

```bash
echo "<donnees_base64>" | base64 --decode > ca.crt
```

***

### Exemple : plusieurs clusters et utilisateurs
Supposons que vous travailliez sur deux clusters (`dev` et `prod`) avec deux utilisateurs (`dev-user` et `prod-admin`).

```yaml
contexts:
  - name: dev-user@dev
    context:
      cluster: dev
      user: dev-user
      namespace: dev-namespace

  - name: prod-admin@prod
    context:
      cluster: prod
      user: prod-admin
      namespace: production
```

Pour changer d’environnement :

```bash
kubectl config use-context dev-user@dev
kubectl config use-context prod-admin@prod
```

***

### Points clés à retenir
- Le **kubeconfig** centralise toutes les informations d’accès (clusters, utilisateurs, contextes).
- Il évite de ressaisir les certificats et adresses serveur à chaque commande.
- Vous pouvez définir un context par défaut ou en changer à la volée.
- Le **namespace par défaut** peut être configuré par context.
- Les certificats peuvent être référencés par fichier ou encodés en Base64.

***

### Résumé concis
Le fichier **kubeconfig** permet à `kubectl` de se connecter à un ou plusieurs clusters Kubernetes sans avoir à saisir manuellement les certificats et adresses de serveur.

Il contient trois sections principales :
- **clusters** : la liste des clusters accessibles ;
- **users** : les identités (certificats, tokens, etc.) ;
- **contexts** : l’association entre un utilisateur, un cluster et éventuellement un namespace.

Le champ `current-context` indique le context actif.

**Commandes utiles :**
```bash
kubectl config view                     # Voir le contenu du kubeconfig
kubectl config use-context <nom>        # Changer de context
kubectl config set-context <nom> ...    # Créer ou modifier un context
```

**Exemple :**
```yaml
current-context: dev-user@minikube
```
→ Toutes les commandes `kubectl` s’exécuteront avec ce contexte par défaut.


