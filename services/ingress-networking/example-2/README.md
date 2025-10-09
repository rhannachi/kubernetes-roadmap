## Déployer & tester sur **minikube**

1. Démarre minikube :
   ```
   minikube start --driver=docker
   ```

2. Obtenir une IP LoadBalancer pour le service edge :
    * Exécute `minikube tunnel` dans un terminal (nécessaire pour services `LoadBalancer`) :

      Laisse ce tunnel up pendant le test.

3. Appliquer le manifeste (fichier `deployment.yaml`) :
   ```
   kubectl apply -f deployment.yaml
   ```

4. Vérifier que le service edge a une IP :
   ```
   kubectl get svc -n ingress-edge
   # regarde l'EXTERNAL-IP sur svc-ingress-nginx-controller
   ```

5. Tester les routes (supposons EXTERNAL-IP = 192.168.49.2) :
   ```
   # app1 red
   curl http://<EXTERNAL-IP>/app1/red
   # app1 blue
   curl http://<EXTERNAL-IP>/app1/blue
   # app2 red
   curl http://<EXTERNAL-IP>/app2/red
   # app2 blue
   curl http://<EXTERNAL-IP>/app2/blue
   ```
