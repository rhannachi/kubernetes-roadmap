# 🧠 Objectif général

> Apprendre à **identifier où se trouve le problème** :
>
> * Est-ce un souci d’objet non créé ?
> * D’accès API (RBAC) ?
> * De routage NGINX ?
> * De service ou pod backend ?
> * D’exposition externe ?

---

## 🩺 1. Vérifier la base : Namespace

### 🔍 Commande :

```bash
kubectl get ns
```

### 🧠 Explication :

* Liste tous les namespaces du cluster.
* Tu dois voir `web-apps` dans la liste.
  → S’il n’apparaît pas, rien ne s’est déployé au bon endroit.

### 🧰 Pour aller plus loin :

```bash
kubectl describe ns web-apps
```

* Affiche les détails du namespace (annotations, status, etc.).
* Permet de vérifier qu’il est bien *Active* (et non *Terminating*).

---

## 🔒 2. Vérifier les permissions RBAC

### 🔍 Commande :

```bash
kubectl get clusterrole cr-ingress-nginx-controller
kubectl get clusterrolebinding crb-ingress-nginx-controller
```

### 🧠 Explication :

* Vérifie que le **ClusterRole** et le **ClusterRoleBinding** existent.
* Ces deux objets doivent être créés pour permettre au contrôleur NGINX de lire les ressources Ingress et Services.

### 🧰 Diagnostic avancé :

```bash
kubectl describe clusterrole cr-ingress-nginx-controller
kubectl describe clusterrolebinding crb-ingress-nginx-controller
```

* Vérifie que le `ServiceAccount` `sa-ingress-nginx-controller` est bien lié au ClusterRole.

---

## 👤 3. Vérifier le ServiceAccount du contrôleur

### 🔍 Commande :

```bash
kubectl get sa -n web-apps
```

### 🧠 Explication :

* Vérifie que le ServiceAccount `sa-ingress-nginx-controller` est bien créé.
* C’est lui qui donne une identité au pod du contrôleur.

---

## ⚙️ 4. Vérifier le ConfigMap

### 🔍 Commande :

```bash
kubectl get configmap -n web-apps
kubectl describe configmap ingress-nginx-config -n web-apps
```

### 🧠 Explication :

* Le contrôleur NGINX lit ce ConfigMap pour sa configuration.
* Si le ConfigMap est absent ou mal nommé, NGINX risque de démarrer avec des erreurs.

---

## 🧭 5. Vérifier l’IngressClass

### 🔍 Commande :

```bash
kubectl get ingressclass
kubectl describe ingressclass ingress-nginx-class
```

### 🧠 Explication :

* Confirme que le contrôleur NGINX est bien **enregistré** comme gestionnaire d’Ingress.
* Si ton `Ingress` fait référence à une classe inexistante, le contrôleur **ignorera** cet Ingress (aucun routage ne sera fait).

---

## 🚀 6. Vérifier le déploiement du contrôleur Ingress NGINX

### 🔍 Commande :

```bash
kubectl get deploy -n web-apps
```

### 🧠 Explication :

* Vérifie que `deploy-ingress-nginx-controller` est bien créé.
* Si le statut n’est pas “AVAILABLE”, tu dois creuser.

### 🔧 Pour aller plus loin :

```bash
kubectl describe deploy deploy-ingress-nginx-controller -n web-apps
```

* Vérifie les événements (Events) à la fin :

    * ❌ ImagePullBackOff → problème d’image
    * ❌ CrashLoopBackOff → conteneur plante
    * ❌ Unauthorized → problème RBAC

---

## 🧩 7. Vérifier le pod du contrôleur NGINX

### 🔍 Commande :

```bash
kubectl get pods -n web-apps -l app=ingress-nginx-controller
```

### 🧠 Explication :

* Vérifie si le pod est `Running` et `Ready`.
* Le label `app=ingress-nginx-controller` permet de le filtrer facilement.

### 🔧 Si problème :

```bash
kubectl describe pod <nom-du-pod> -n web-apps
kubectl logs <nom-du-pod> -n web-apps
```

### 🧠 Lecture :

* `kubectl describe` montre les erreurs de scheduling, probes ou configuration.
* `kubectl logs` montre les logs du contrôleur :

    * S’il lit bien les Ingress.
    * S’il a des erreurs du type “no endpoints found”.
    * Ou encore s’il n’arrive pas à accéder à son service.

---

## 🌐 8. Vérifier le Service du contrôleur

### 🔍 Commande :

```bash
kubectl get svc -n web-apps
```

### 🧠 Explication :

* Vérifie que `svc-ingress-nginx-controller` est bien de type **NodePort**.
* Regarde le **port exposé** (`PORT(S)` colonne).

### 🔧 Pour plus de détails :

```bash
kubectl describe svc svc-ingress-nginx-controller -n web-apps
```

* Vérifie :

    * Les ports (80 et 443)
    * Le selector `app=ingress-nginx-controller` → doit matcher ton pod.

---

## 🔴 9. Vérifier les déploiements applicatifs (RED & BLUE)

### 🔍 Commande :

```bash
kubectl get deploy -n web-apps
```

### 🧠 Explication :

* Doit afficher `deploy-web-red` et `deploy-web-blue` avec 1/1 réplicas disponibles.

### 🔧 Si problème :

```bash
kubectl describe deploy deploy-web-red -n web-apps
kubectl logs -n web-apps -l app=web-red
```

* `describe` → montre si les pods échouent à démarrer.
* `logs` → vérifie que le NGINX interne tourne bien (tu devrais voir les logs HTTP classiques).

---

## 💡 10. Vérifier les services applicatifs (RED & BLUE)

### 🔍 Commande :

```bash
kubectl get svc -n web-apps
```

### 🧠 Explication :

* Doit lister :

    * `svc-web-red`
    * `svc-web-blue`
* Type `ClusterIP`
* Vérifie que le port `80` est exposé.

### 🔧 Pour valider le mapping label → pod :

```bash
kubectl get endpoints svc-web-red -n web-apps
kubectl get endpoints svc-web-blue -n web-apps
```

### 🧠 Lecture :

* Si la sortie est vide (`<none>`), le **Service ne trouve aucun pod**.
  → Vérifie les labels :

  ```bash
  kubectl get pods -n web-apps --show-labels
  ```

  Les labels `app=web-red` et `app=web-blue` doivent correspondre.

---

## 🚦 11. Vérifier l’objet Ingress

### 🔍 Commande :

```bash
kubectl get ingress -n web-apps
kubectl describe ingress ingress-web-apps -n web-apps
```

### 🧠 Explication :

* `get` → vérifie qu’il existe et a bien une `ADDRESS` (IP du contrôleur).
* `describe` → montre les règles (paths `/red`, `/blue`) et les backends associés.

### ⚠️ Diagnostic :

* Si `ADDRESS` est vide → le contrôleur ne gère pas cet Ingress.
  → Vérifie le champ `ingressClassName` et compare avec ton `IngressClass`.

---

## 🌍 12. Tester le routage HTTP (depuis le cluster)

### 🔍 Commandes :

```bash
kubectl exec -n web-apps -it deploy/deploy-web-red -- curl -v http://svc-ingress-nginx-controller.web-apps.svc.cluster.local/red
```

👉 (et pour tester l’autre application)

```bash
kubectl exec -n web-apps -it deploy/deploy-web-blue -- curl -v http://svc-ingress-nginx-controller.web-apps.svc.cluster.local/blue
```

---

### 🧠 Explication pédagogique :

* Cette commande exécute `curl` **à l’intérieur d’un pod déjà existant** (`web-red` ou `web-blue`).
* Cela permet de tester la **connectivité réseau interne** sans créer de pod temporaire (ce qui évite les erreurs de timeout que tu avais eues avec `kubectl run`).
* On vérifie que le pod peut joindre le **Service du contrôleur NGINX** via son nom DNS interne :

  ```
  svc-ingress-nginx-controller.web-apps.svc.cluster.local
  ```
* En retour, tu devrais obtenir une réponse HTTP `200 OK` contenant :

  ```
  <h1>RED APP</h1>
  ```

  ou

  ```
  <h1>BLUE APP</h1>
  ```
* Cela confirme que :

    * Le **DNS interne** de Kubernetes fonctionne,
    * Le **Service NGINX** est accessible depuis d’autres pods du cluster,
    * Le **routage Ingress** redirige bien les requêtes vers le bon backend.

---

### 💡 En cas d’erreur

Si la commande renvoie une erreur du type :

```
curl: (7) Failed to connect to ...
```

alors :

1. Vérifie que le Service du contrôleur est bien créé :

   ```bash
   kubectl get svc -n web-apps
   ```
2. Vérifie que le pod du contrôleur est en **Running** :

   ```bash
   kubectl get pods -n web-apps -l app=ingress-nginx-controller
   ```
3. Et consulte les logs :

   ```bash
   kubectl logs deploy/deploy-ingress-nginx-controller -n web-apps
   ```

---

Voici la version **mise à jour et conforme à Minikube**, qui simplifie l’accès externe en utilisant la commande `minikube service --url` au lieu de chercher manuellement le NodePort :

---

## 🌎 13. Tester l’accès externe (depuis ta machine avec Minikube)

### 🔍 Commande :

Pour récupérer l’URL d’accès externe au service NGINX :

```bash
minikube service svc-ingress-nginx-controller -n web-apps --url
```

Exemple de sortie :

```
http://192.168.49.2:31811
http://192.168.49.2:32021
```

---

### 🔍 Tester les applications :

```bash
curl http://192.168.49.2:31811/red
curl http://192.168.49.2:31811/blue
```

⚠️ Note :

* L’une des URL correspond au port HTTP (`80`) et l’autre au port HTTPS (`443`).
* Pour HTTPS, tu peux rencontrer un avertissement de certificat auto-signé :

```bash
curl -k https://192.168.49.2:32021/red
```

Le `-k` permet d’ignorer le certificat auto-signé dans un cluster Minikube.

---

### 🧠 Explication pédagogique :

* Cette méthode teste **le chemin complet depuis ton poste jusqu’aux pods** via le NodePort exposé par Minikube.
* Elle remplace le besoin de connaître manuellement le NodePort ou l’IP du nœud.
* Cela vérifie :

    * Le Service `svc-ingress-nginx-controller` est accessible depuis l’extérieur,
    * Le routage Ingress fonctionne (`/red` → web-red, `/blue` → web-blue),
    * Les pods backend répondent correctement.

---

## 🧹 14. Vérifier la santé globale

### 🔍 Commande :

```bash
kubectl get all -n web-apps
```

### 🧠 Explication :

* Résumé de tout ce qui tourne dans le namespace :

    * Pods, Deployments, Services, Ingress, ReplicaSets...
* Permet de repérer d’un coup d’œil un objet en erreur.

---

## 🔧 15. Nettoyage après test

### 🔍 Commande :

```bash
kubectl delete ns web-apps
```

### 🧠 Explication :

* Supprime tout le namespace, donc toutes les ressources à l’intérieur.
* Utile pour repartir sur une base propre après un test.

---

# 🧭 Résumé — Stratégie de Diagnostic

| Étape | Cible      | Commande clé                                       | Diagnostic principal        |
| ----- | ---------- | -------------------------------------------------- | --------------------------- |
| 1     | Namespace  | `kubectl get ns`                                   | Ressources bien créées      |
| 2     | RBAC       | `kubectl describe clusterrolebinding`              | Accès API valide            |
| 3     | Controller | `kubectl get pods -l app=ingress-nginx-controller` | Pod NGINX up                |
| 4     | Services   | `kubectl get endpoints`                            | Pods connectés aux services |
| 5     | Ingress    | `kubectl describe ingress`                         | Règles bien appliquées      |
| 6     | Routage    | `kubectl run curlpod ...`                          | Trafic interne OK           |
| 7     | Externe    | `curl http://<NODE_IP>:<NODE_PORT>`                | Accès utilisateur OK        |

