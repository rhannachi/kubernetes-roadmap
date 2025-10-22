## Helm
On va convertir un ancien projet sur lequel on a travaillé, [services/ingress-networking/example-2](../../services/ingress-networking/example-2/README.md), en une application Helm.

### Installation
Pour installer Helm sur une machine Debian, voici les [méthodes officielles et recommandées](https://helm.sh/docs/intro/install/) :
``` 
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Lancer le projet avec Helm

```bash
$ minikube tunnel

$ helm install ingress-arch ./ingress-arch
NAME: ingress-arch
LAST DEPLOYED: Wed Oct 22 11:59:20 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

$ kubectl get svc -n ingress-edge
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
svc-ingress-nginx-controller   LoadBalancer   10.103.57.89   10.103.57.89   80:31801/TCP,443:32021/TCP   32s

$ curl http://10.103.57.89/app1/red
<h1>RED APP (app1)</h1>

$ curl http://10.103.57.89/app2/red
<h1>RED APP (app2)</h1>
```

***

## **Toutes les commandes Helm les plus utiles et puissantes** pour le projet `ingress-arch`

## 1. Installation / Mise à jour / Suppression

### ➤ **Installer un chart**

```bash
helm install ingress-arch ./ingress-arch
```

Installe ton chart Helm (dans le namespace par défaut ou un autre avec `-n <namespace>`).

Exemple :

```bash
helm install ingress-arch ./ingress-arch -n ingress-edge --create-namespace
```

---

### ➤ **Mettre à jour une release**

```bash
helm upgrade ingress-arch ./ingress-arch
```

Met à jour la release avec les nouveaux fichiers templates ou values.

* Tu peux passer des variables à la volée :

  ```bash
  helm upgrade ingress-arch ./ingress-arch --set app1.replicas=3
  ```
* Ou utiliser un fichier personnalisé :

  ```bash
  helm upgrade ingress-arch ./ingress-arch -f values-prod.yaml
  ```

---

### ➤ **Installer ou mettre à jour automatiquement**

```bash
helm upgrade --install ingress-arch ./ingress-arch
```

💡 Très utilisée en **CI/CD** :
Si la release existe → upgrade
Sinon → install

---

### ➤ **Supprimer une release**

```bash
helm uninstall ingress-arch
```

Supprime tous les objets créés par Helm (mais garde l’historique par défaut).

---

## 2. Gestion des versions et rollback

### ➤ **Lister les releases**

```bash
helm list
```

Affiche toutes les releases Helm installées sur ton cluster.

---

### ➤ **Afficher l’historique des versions**

```bash
helm history ingress-arch
```

Exemple de sortie :

```
REVISION | UPDATED             | STATUS   | CHART           | DESCRIPTION
1         | 2025-10-22 14:35:20| deployed | ingress-arch-0.1.0 | Install complete
2         | 2025-10-22 14:40:50| deployed | ingress-arch-0.1.0 | Upgrade complete
```

---

### ➤ **Faire un rollback**

```bash
helm rollback ingress-arch 1
```

⏪ Revient à la version précédente (`REVISION` 1 ici).
Très utile si une mise à jour casse quelque chose.

---

## 3. Exploration et débogage

### ➤ **Voir les ressources déployées**

```bash
helm get all ingress-arch
```

Affiche tout ce que Helm a appliqué : templates rendus, secrets, chart metadata.

---

### ➤ **Voir uniquement les values utilisées**

```bash
helm get values ingress-arch
```

Tu peux aussi voir les valeurs par défaut :

```bash
helm show values ./ingress-arch
```

---

### ➤ **Voir les manifests Kubernetes générés**

Avant d’appliquer un chart :

```bash
helm template ./ingress-arch
```

💡 Cela ne déploie rien — ça affiche simplement les YAML Kubernetes qui seraient créés.

---

### ➤ **Vérifier le rendu final des valeurs**

```bash
helm get manifest ingress-arch
```

→ Montre les manifests effectivement déployés.

---

### ➤ **Vérifier les différences avant upgrade**

```bash
helm diff upgrade ingress-arch ./ingress-arch
```

💡 Nécessite le plugin [`helm-diff`](https://github.com/databus23/helm-diff)
Super pratique pour voir les changements avant de les appliquer.

---

## 4. Charts, dépendances, et packaging

### ➤ **Créer un nouveau chart**

```bash
helm create mychart
```

Crée une structure de chart standard (comme ton `ingress-arch`).

---

### ➤ **Packager ton chart**

```bash
helm package ./ingress-arch
```

Crée un fichier `.tgz` (chart compressé) → idéal pour publier sur un repo Helm.

---

### ➤ **Vérifier ton chart**

```bash
helm lint ./ingress-arch
```

Analyse syntaxe, structure, et cohérence des templates.
⚠️ À faire **avant tout déploiement**.

---

### ➤ **Installer un chart depuis un repo distant**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mypostgres bitnami/postgresql
```

### ➤ **Lister les repos ajoutés**

```bash
helm repo list
```

### ➤ **Mettre à jour les repos**

```bash
helm repo update
```

---

## 5. Debug & tests avancés

### ➤ **Tester ton chart localement (sans déploiement)**

```bash
helm template ./ingress-arch | kubectl apply --dry-run=client -f -
```

Simule un déploiement complet sans rien appliquer.

---

### ➤ **Tester ton chart avec des tests Helm**

Si tu ajoutes un test dans `templates/tests/`, exécute :

```bash
helm test ingress-arch
```

---

### ➤ **Nettoyer les hooks échoués**

```bash
helm uninstall ingress-arch --keep-history
```

Puis réinstaller proprement.

---

### ➤ **Afficher l’aide intégrée**

```bash
helm help
helm install --help
```

---

## 6. Plugins Helm utiles (bonus)

Tu peux ajouter des fonctionnalités puissantes avec des plugins :

| Plugin            | Commande d’installation                                         | Description                           |
| ----------------- | --------------------------------------------------------------- | ------------------------------------- |
| **helm-diff**     | `helm plugin install https://github.com/databus23/helm-diff`    | Compare les releases avant upgrade    |
| **helm-secrets**  | `helm plugin install https://github.com/jkroepke/helm-secrets`  | Gère les secrets chiffrés (SOPS, GPG) |
| **helm-unittest** | `helm plugin install https://github.com/quintush/helm-unittest` | Tests unitaires sur templates         |
| **helmfile**      | via brew ou apt                                                 | Gestion multi-charts (CI/CD avancé)   |

---

## En résumé : les 10 commandes Helm les plus puissantes

| Action                | Commande                                   |
| --------------------- | ------------------------------------------ |
| Installer un chart    | `helm install ingress-arch ./ingress-arch` |
| Mettre à jour         | `helm upgrade ingress-arch ./ingress-arch` |
| Rollback              | `helm rollback ingress-arch 1`             |
| Voir les releases     | `helm list`                                |
| Voir l’historique     | `helm history ingress-arch`                |
| Inspecter les valeurs | `helm get values ingress-arch`             |
| Voir les manifests    | `helm get manifest ingress-arch`           |
| Tester le rendu       | `helm template ./ingress-arch`             |
| Lint du chart         | `helm lint ./ingress-arch`                 |
| Supprimer la release  | `helm uninstall ingress-arch`              |

---


