# Exercices :

1. **Lister tous les namespaces existants dans le cluster**  
Utiliser `kubectl get namespaces` pour afficher la liste des namespaces par défaut et personnalisés.

2. **Créer un nouveau namespace**  
Créer un namespace nommé `dev-team` via la commande impérative et vérifier sa création.

3. **Déployer un Pod dans un namespace spécifique**  
Créer un Pod nginx dans le namespace `dev-team` en utilisant `kubectl run` avec le flag `-n`.

4. **Lister les ressources dans un namespace spécifique**  
Afficher tous les Pods dans le namespace `dev-team` avec `kubectl get pods -n dev-team`.

5. **Créer un Deployment dans un namespace personnalisé**  
Créer un Deployment simple nginx dans le namespace `production` et vérifier que le Deployment est bien dans ce namespace.

6. **Déployer un service dans un namespace et l'exposer**  
Créer un Service dans le namespace `dev-team` pour exposer un Pod ou un Deployment.

7. **Changer le namespace par défaut dans le contexte kubectl**  
Utiliser `kubectl config set-context` pour définir un namespace par défaut afin d’éviter la spécification répétée du flag `-n`.

8. **Créer un ResourceQuota dans un namespace**  
Définir un quota limite en nombre de Pods et consommation CPU/mémoire dans le namespace `dev-team`.

9. **Lister toutes les ressources dans tous les namespaces**  
Avec `kubectl get all --all-namespaces`, observer l’organisation des ressources réparties.

10. **Supprimer un namespace et toutes les ressources qu’il contient**  
Supprimer un namespace personnalisé, par exemple `dev-team`, et vérifier la suppression définitive.

***

# Solution :

1. **Lister tous les namespaces :**
```
$ kubectl get namespaces
```

2. **Créer un namespace `dev-team` :**
```
$ kubectl create namespace dev-team
```

3. **Créer un Pod nginx dans le namespace `dev-team` :**
```
$ kubectl run nginx-pod --image=nginx --restart=Never -n dev-team
```

4. **Lister les Pods dans le namespace `dev-team` :**
```
$ kubectl get pods -n dev-team
```

5. **Créer un Deployment nginx dans le namespace `production` :**
```
$ kubectl create deployment nginx-deploy --image=nginx -n production
```

6. **Créer un Service dans le namespace `dev-team` pour exposer un Pod :**
```
$ kubectl expose pod nginx-pod --port=80 --target-port=80 --type=ClusterIP -n dev-team
```

7. **Changer le namespace par défaut pour kubectl dans le contexte actuel :**
```
$ kubectl config set-context --current --namespace=dev-team
```

8. **Créer un ResourceQuota dans le namespace `dev-team` (exemple simple) :**
Création du fichier `quota.yaml` (ou utiliser commande équivalente) puis appliquer :
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-team-quota
  namespace: dev-team
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "6"
    limits.memory: "12Gi"
```

Puis appliquer :
```
$ kubectl apply -f quota.yaml
```

9. **Lister tous les objets dans tous les namespaces :**
```
$ kubectl get all --all-namespaces
```

10. **Supprimer un namespace `dev-team` et toutes ses ressources :**
```
$ kubectl delete namespace dev-team
```


