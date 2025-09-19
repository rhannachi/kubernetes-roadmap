# Exercices

- Crée un Pod nommé **nginx-pod** avec l’image **nginx:alpine** :
  ```
  $ kubectl run nginx-pod --image=nginx:alpine --restart=Never
  ```

- Crée un Pod **redis** avec l’image **redis:alpine** et un label `tier=db` :
  ```
  $ kubectl run redis --image=redis:alpine --labels=tier=db --restart=Never
  ```

- Crée un service nommé **redis-service** pour exposer le Pod existant **redis** dans le cluster sur le port 6379 :
  ```
  $ kubectl expose pod redis --name=redis-service --port=6379 --target-port=6379 --type=ClusterIP
  ```

- Crée un déploiement nommé **webapp** utilisant l’image **kodekloud/webapp-color** avec 3 réplicas (en deux étapes car `--replicas` n’existe pas directement sur create deployment) :
  ```
  $ kubectl create deployment webapp --image=kodekloud/webapp-color
  $ kubectl scale deployment webapp --replicas=3
  ```

- Crée un nouveau Pod appelé **custom-nginx** avec l’image **nginx** et expose le container sur le port 8080 :
  ```
  $ kubectl run custom-nginx --image=nginx --port=8080 --restart=Never
  ```

- Crée un nouveau namespace appelé **dev-ns** :
  ```
  $ kubectl create namespace dev-ns
  ```

- Crée un déploiement nommé **redis-deploy** dans le namespace **dev-ns** avec l’image **redis** et 2 réplicas (idem, 2 commandes) :
  ```
  $ kubectl create deployment redis-deploy --image=redis -n dev-ns
  $ kubectl scale deployment redis-deploy --replicas=2 -n dev-ns
  ```

- Crée un Pod nommé **httpd** avec l’image **httpd:alpine** dans le namespace par défaut, puis crée un service de type ClusterIP du même nom exposant ce Pod sur le port 80 :
  ```
  $ kubectl run httpd --image=httpd:alpine --restart=Never
  $ kubectl expose pod httpd --name=httpd --port=80 --target-port=80 --type=ClusterIP
  ```