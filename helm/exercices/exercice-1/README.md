## Helm

Nous allons convertir un ancien projet, [services/ingress-networking/example-2](../../../services/ingress-networking/example-2/README.md), en une application **Helm**.

L’objectif est de **reproduire la même architecture**, puis d’y **intégrer les éléments** de l’ancienne version dans les fichiers équivalents :

```
ingress-arch/
├─ Chart.yaml
├─ values.yaml
├─ templates/
│  ├─ namespace.yaml
│  ├─ ingressclass.yaml
│  ├─ clusterrole.yaml
│  ├─ clusterrolebinding.yaml
│  ├─ serviceaccount.yaml
│  ├─ configmap.yaml
│  ├─ deployment-edge.yaml
│  ├─ service-edge.yaml
│  ├─ apps-app1.yaml
│  ├─ apps-app2.yaml
```

***

### Installation

Pour installer Helm sur une machine Debian, suivez les [instructions officielles](https://helm.sh/docs/intro/install/) :

```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

***

### Lancer le projet avec Helm

Démarrez le **tunnel Minikube** :

```bash
minikube tunnel
```

Puis installez le chart :

```bash
helm install ingress-arch ./ingress-arch
```

Exemple de sortie :

```
NAME: ingress-arch
LAST DEPLOYED: Wed Oct 22 11:59:20 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

***

### Comprendre le fonctionnement de Helm

Lors de la **création d’un chart** :

* vous rédigez des **templates YAML** contenant des variables (`{{ ... }}`),
* vous définissez des **valeurs** dans `values.yaml`,
* mais **aucune ressource Kubernetes réelle** n’est encore déployée.

La commande :

```bash
helm install ingress-arch ./ingress-arch
```

réalise trois étapes clefs :

1. **Rendu (templating)**  
   Helm fusionne tous vos templates avec les valeurs pour générer du YAML Kubernetes pur.  
   Équivalent à :
   ```bash
   helm template ingress-arch ./ingress-arch
   ```
   (mais avec application automatique).

2. **Déploiement dans le cluster**  
   Helm applique le YAML rendu dans le namespace cible (par défaut `default`).

3. **Gestion du release**  
   Helm trace cette release (`ingress-arch`) avec une révision et un état (deployed, failed, etc.), ce qui permet :
    * de faire un `upgrade`,
    * de revenir en arrière (`rollback`),
    * ou de supprimer (`uninstall`).

***

### Vérification et test de déploiement

```bash
kubectl get svc -n ingress-edge
```

Exemple :

```
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
svc-ingress-nginx-controller   LoadBalancer   10.103.57.89   10.103.57.89   80:31801/TCP,443:32021/TCP   32s
```

Testez ensuite vos endpoints :

```bash
curl http://10.103.57.89/app1/red
curl http://10.103.57.89/app2/red
```

***

## Commandes Helm essentielles

| Action principale        | Commande clé                                       | Description courte |
|--------------------------|----------------------------------------------------|--------------------|
| Installer un chart       | `helm install ingress-arch ./ingress-arch`        | Déploie la première version du chart |
| Mettre à jour une release| `helm upgrade ingress-arch ./ingress-arch`        | Applique les changements sans recréer |
| Supprimer une release    | `helm uninstall ingress-arch`                     | Supprime les objets déployés |
| Voir les releases actives| `helm list`                                       | Liste toutes les releases Helm |
| Consulter l’historique   | `helm history ingress-arch`                       | Affiche les révisions précédentes |
| Rollback vers une version| `helm rollback ingress-arch 1`                    | Restaure une version antérieure |
| Inspecter les valeurs    | `helm get values ingress-arch`                    | Montre les valeurs utilisées au déploiement |
| Vérifier le rendu YAML   | `helm template ./ingress-arch`                    | Affiche les manifests Kubernetes générés |
| Linter un chart          | `helm lint ./ingress-arch`                        | Vérifie la validité du chart avant déploiement |


