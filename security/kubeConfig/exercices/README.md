## Exercice : Gérer plusieurs environnements dans un cluster Minikube

Tu disposes d’un **cluster Kubernetes local** tournant sur **Minikube**.  
L’objectif de cet exercice est de **créer trois environnements distincts** — développement, test et production — et de les relier à trois utilisateurs différents dans ton fichier de configuration Kubernetes (`~/.kube/config`).

#### Objectifs :
1. Créer trois **namespaces** :
    - `dev` pour le développement
    - `test` pour les tests
    - `prod` pour la production

2. Créer trois **utilisateurs Kubernetes** :
    - `dev-user` associé au namespace `dev`
    - `test-user` associé au namespace `test`
    - `prod-user` associé au namespace `prod`

3. Configurer ton fichier **kubeconfig** pour que chaque utilisateur soit lié à son namespace à travers un **contexte Kubernetes**.

4. Être capable de **passer facilement d’un environnement à un autre** avec la commande :
   ```
   $ kubectl config use-context <nom_du_contexte>
   ```

***

## Solution :

### Sauvegarder le kubeconfig actuel
Avant tout changement :
```
$ cp ~/.kube/config ~/.kube/config.backup
```
Cela te permet de revenir à la version d’origine à tout moment.

### 1. Créer les namespaces
```
$ kubectl create namespace dev
$ kubectl create namespace test
$ kubectl create namespace prod
```
Cela crée les trois environnements isolés dans ton cluster.

### 2. Créer les utilisateurs
Avec Minikube, tu peux créer des utilisateurs en générant des certificats client :
```
# Exemple pour dev-user
$ openssl genrsa -out dev-user.key 2048
$ openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=dev"
$ openssl x509 -req -in dev-user.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out dev-user.crt -days 365
```
Répète ces commandes pour test-user et prod-user.

### 3. Ajouter les utilisateurs dans le kubeconfig
```
$ kubectl config set-credentials dev-user --client-certificate=dev-user.crt --client-key=dev-user.key
$ kubectl config set-credentials test-user --client-certificate=test-user.crt --client-key=test-user.key
$ kubectl config set-credentials prod-user --client-certificate=prod-user.crt --client-key=prod-user.key
```
Cette commande ajoute les utilisateurs à ton fichier kubeconfig

### 4. Créer les contextes
```
$ kubectl config set-context dev --cluster=minikube --namespace=dev --user=dev-user
$ kubectl config set-context test --cluster=minikube --namespace=test --user=test-user
$ kubectl config set-context prod --cluster=minikube --namespace=prod --user=prod-user
```
Les contextes lient chaque utilisateur et namespace au cluster existant.

### 5. Basculer entre les environnements
```
$ kubectl config use-context dev
$ kubectl config use-context test
$ kubectl config use-context prod
```
Cette commande te permet de changer d’environnement selon le contexte actif.

### 6. Vérifier ta configuration
```
$ kubectl config get-contexts
$ kubectl config current-context
```

Ton fichier `.kube/config` sera automatiquement mis à jour avec ces contextes et utilisateurs.\
Il n’est pas recommandé de l’éditer manuellement, car `kubectl config set-*` gère la structure YAML proprement.

