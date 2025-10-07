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
kubectl run curlpod --image=curlimages/curl -it --rm -n web-apps -- \
  curl http://svc-ingress-nginx-controller.web-apps.svc.cluster.local/red
```

### 🧠 Explication :

* Lancer un pod temporaire (`curlpod`) dans le même namespace.
* Tester le chemin `/red` ou `/blue`.
* Tu devrais obtenir :

  ```
  <h1>RED APP</h1>
  ```

  ou

  ```
  <h1>BLUE APP</h1>
  ```

### 💡 Si tu obtiens une erreur 404 :

* Vérifie la règle `nginx.ingress.kubernetes.io/rewrite-target`.
* Vérifie que le contrôleur NGINX voit bien l’Ingress (via ses logs).

---

## 🌎 13. Tester l’accès externe (depuis ta machine)

### 🔍 Commande :

```bash
kubectl get svc svc-ingress-nginx-controller -n web-apps
```

➡️ Note le `NODE-PORT` (ex : 30080)

### Puis :

```bash
curl http://<NODE_IP>:<NODE_PORT>/red
curl http://<NODE_IP>:<NODE_PORT>/blue
```

### 🧠 Explication :

* Permet de tester le flux complet depuis ton poste jusqu’aux pods.
* Si cela échoue :

    * Vérifie que tu peux atteindre le nœud (`ping <NODE_IP>`).
    * Vérifie qu’aucun pare-feu ne bloque le NodePort.

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

***
***

# 🧩 Kubernetes Ingress NGINX – Diagnostic Guide

## 0️⃣ Préambule : contexte et namespace

```bash
# Lister tous les namespaces
kubectl get ns

# Si le namespace web-apps n’existe pas → rien ne fonctionnera
kubectl describe ns web-apps
```

---

## 1️⃣ Vérification de la base RBAC

```bash
# Vérifie que les rôles sont présents
kubectl get clusterrole cr-ingress-nginx-controller
kubectl get clusterrolebinding crb-ingress-nginx-controller

# Vérifie le lien entre le ServiceAccount et le ClusterRole
kubectl describe clusterrolebinding crb-ingress-nginx-controller
```

🧠 *Si le ServiceAccount n’est pas référencé ici → le contrôleur NGINX n’aura pas accès aux Ingress.*

---

## 2️⃣ Vérifier le ServiceAccount du contrôleur

```bash
kubectl get sa -n web-apps
kubectl describe sa sa-ingress-nginx-controller -n web-apps
```

🧠 *Il doit exister et être associé à ton déploiement `deploy-ingress-nginx-controller`.*

---

## 3️⃣ Vérifier la configuration du contrôleur

```bash
# Vérifie le ConfigMap (paramètres NGINX)
kubectl get configmap -n web-apps
kubectl describe configmap ingress-nginx-config -n web-apps

# Vérifie la classe Ingress utilisée
kubectl get ingressclass
kubectl describe ingressclass ingress-nginx-class
```

🧠 *Si le `ingressClassName` de ton Ingress ne correspond pas à la classe du contrôleur → NGINX ignorera ton Ingress.*

---

## 4️⃣ Vérifier le déploiement du contrôleur Ingress NGINX

```bash
# Vérifie que le déploiement existe
kubectl get deploy -n web-apps

# Vérifie qu’il est en cours d’exécution
kubectl describe deploy deploy-ingress-nginx-controller -n web-apps

# Vérifie le pod associé
kubectl get pods -n web-apps -l app=ingress-nginx-controller

# Si un pod est en erreur :
kubectl describe pod <nom-du-pod> -n web-apps
kubectl logs <nom-du-pod> -n web-apps
```

🧠 *Les logs du contrôleur sont essentiels — ils montrent s’il détecte les Ingress, s’il a des erreurs de config, etc.*

---

## 5️⃣ Vérifier le Service du contrôleur (exposition)

```bash
kubectl get svc -n web-apps
kubectl describe svc svc-ingress-nginx-controller -n web-apps
```

🧠 *Type attendu : NodePort (pour exposer vers l’extérieur).
Vérifie que le selector `app=ingress-nginx-controller` correspond bien à ton pod.*

---

## 6️⃣ Vérifier les applications backend

```bash
# Vérifie les déploiements
kubectl get deploy -n web-apps
kubectl describe deploy deploy-web-red -n web-apps
kubectl describe deploy deploy-web-blue -n web-apps

# Vérifie les pods correspondants
kubectl get pods -n web-apps --show-labels
kubectl logs -n web-apps -l app=web-red
kubectl logs -n web-apps -l app=web-blue
```

🧠 *Si un déploiement est "0/1 READY", il y a un souci d’image ou de démarrage.*

---

## 7️⃣ Vérifier les Services backend

```bash
kubectl get svc -n web-apps
kubectl describe svc svc-web-red -n web-apps
kubectl describe svc svc-web-blue -n web-apps

# Vérifie si le service pointe vers des endpoints valides
kubectl get endpoints svc-web-red -n web-apps
kubectl get endpoints svc-web-blue -n web-apps
```

🧠 *Si un endpoint est vide → le Service ne trouve aucun pod → problème de label (selector).*

---

## 8️⃣ Vérifier l’Ingress

```bash
kubectl get ingress -n web-apps
kubectl describe ingress ingress-web-apps -n web-apps
```

🧠 *L’adresse (ADDRESS) doit être renseignée, et les règles `/red` et `/blue` doivent pointer vers les bons Services.*

---

## 9️⃣ Tester le routage interne (depuis le cluster)

```bash
# Lance un pod temporaire avec curl
kubectl run curlpod --image=curlimages/curl -it --rm -n web-apps -- \
  curl -v http://svc-ingress-nginx-controller.web-apps.svc.cluster.local/red

kubectl run curlpod --image=curlimages/curl -it --rm -n web-apps -- \
  curl -v http://svc-ingress-nginx-controller.web-apps.svc.cluster.local/blue
```

🧠 *Si tu obtiens `<h1>RED APP</h1>` ou `<h1>BLUE APP</h1>`, le routage interne fonctionne parfaitement.*

---

## 🔟 Tester le routage externe (depuis ta machine)

```bash
kubectl get svc svc-ingress-nginx-controller -n web-apps
```

➡️ Note le **NODE-PORT** (ex : 30080)

Puis :

```bash
curl http://<NODE_IP>:<NODE_PORT>/red
curl http://<NODE_IP>:<NODE_PORT>/blue
```

🧠 *Si cela échoue → vérifie le firewall ou le NodePort.*

---

## 1️⃣1️⃣ Résumé visuel du namespace

```bash
kubectl get all -n web-apps
```

🧠 *Vue d’ensemble rapide de tous les objets et de leur état (Pods, Services, Ingress…).*

---

## 1️⃣2️⃣ Nettoyage (facultatif)

```bash
kubectl delete ns web-apps
```

