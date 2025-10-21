## Validation / Mutation Admission controllers

### Introduction

Dans cette section, nous allons étudier en détail les différents types **d’Admission Controllers** disponibles dans Kubernetes, ainsi que la manière de configurer notre **propre Admission Controller** personnalisé.

Un **Admission Controller** est un composant du **kube-apiserver** qui intercepte les requêtes avant que les objets Kubernetes ne soient stockés dans **etcd**.
Il peut **valider**, **modifier** ou **rejeter** une requête selon certaines règles.

---

### 1. Types d’Admission Controllers

#### a. Validating Admission Controllers

Un **Validating Admission Controller** vérifie la validité d’une requête et peut la **rejeter** si certaines conditions ne sont pas respectées.

**Exemple :**
Le plugin `NamespaceLifecycle` valide si le **Namespace** mentionné dans une requête existe déjà.
Si le Namespace n’existe pas, la requête est rejetée.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx
```

➡️ Si le Namespace `dev` n’existe pas, la création du Pod échouera à cause du contrôleur `NamespaceLifecycle`.

---

#### b. Mutating Admission Controllers

Un **Mutating Admission Controller** peut **modifier la requête** avant qu’elle ne soit enregistrée dans etcd.

**Exemple :**
Le plugin `DefaultStorageClass` s’applique lorsqu’une requête de **PersistentVolumeClaim (PVC)** ne mentionne pas de **StorageClass**.
Le contrôleur ajoute automatiquement la **StorageClass par défaut** configurée dans le Cluster.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

➡️ Même si la `storageClassName` n’est pas indiquée, le contrôleur ajoute par exemple :

```yaml
storageClassName: standard
```

Ainsi, le PVC est créé avec la **classe de stockage par défaut**.

---

### 2. Ordre d’exécution des Admission Controllers

Les **Mutating Admission Controllers** sont exécutés **avant** les **Validating Admission Controllers**.
Cela permet à la phase de validation de prendre en compte les modifications éventuelles effectuées sur la requête.

**Exemple :**

* `NamespaceAutoProvision` (mutating) crée automatiquement un Namespace s’il n’existe pas.
* `NamespaceExists` (validating) vérifie que le Namespace existe.

➡️ Si la validation était exécutée avant la mutation, la création échouerait systématiquement.

---

### 3. Webhooks externes : Admission Webhooks

Kubernetes permet également de définir **des Admission Controllers personnalisés** à l’aide de **webhooks**.

Il existe deux types :

* **MutatingAdmissionWebhook**
* **ValidatingAdmissionWebhook**

Ces webhooks permettent d’appeler un **serveur externe** (interne ou externe au Cluster) pour appliquer notre propre logique de validation ou de mutation.

---

### 4. Fonctionnement du Admission Webhook

1. Une requête utilisateur (ex : création d’un Pod) passe les étapes d’**authentification**, d’**autorisation**, puis les **Admission Controllers intégrés**.
2. Si un **Admission Webhook** est configuré, le **kube-apiserver** envoie la requête au serveur webhook sous forme d’un **AdmissionReview** (objet JSON).
3. Le serveur webhook analyse la requête et renvoie une réponse avec un champ `allowed: true` ou `allowed: false`.

**Exemple de flux :**

```
kubectl apply -f pod.yaml
   ↓
Authentification
   ↓
Admission Controllers intégrés
   ↓
Webhook externe
   ↓
etcd (si accepté)
```

---

### 5. Création d’un Admission Webhook personnalisé

#### Étape 1 : Déploiement du serveur Webhook

Le serveur peut être écrit dans n’importe quel langage (Go, Python, etc.) tant qu’il :

* Accepte les requêtes **/validate** ou **/mutate**,
* Et renvoie une réponse **AdmissionReview** au format JSON.

**Exemple (simplifié en Python)** :

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/validate', methods=['POST'])
def validate():
    review = request.get_json()
    username = review["request"]["userInfo"]["username"]
    object_name = review["request"]["object"]["metadata"]["name"]

    allowed = username != object_name
    return jsonify({
        "response": {"allowed": allowed}
    })

@app.route('/mutate', methods=['POST'])
def mutate():
    review = request.get_json()
    username = review["request"]["userInfo"]["username"]

    patch = [
        {"op": "add", "path": "/metadata/labels/user", "value": username}
    ]

    return jsonify({
        "response": {
            "allowed": True,
            "patchType": "JSONPatch",
            "patch": patch
        }
    })
```

---

#### Étape 2 : Héberger le serveur

* Soit **en dehors du Cluster** avec une **URL publique**,
* Soit **dans le Cluster** comme un **Deployment + Service** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
  namespace: webhook-system
spec:
  selector:
    app: webhook-server
  ports:
    - port: 443
      targetPort: 8443
```

---

#### Étape 3 : Configuration du Webhook dans le Cluster

**Exemple de ValidatingWebhookConfiguration :**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-policy
webhooks:
  - name: pod-policy.example.com
    clientConfig:
      service:
        name: webhook-service
        namespace: webhook-system
        path: "/validate"
      caBundle: <base64-certificat>
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

➡️ À chaque création de Pod, le **kube-apiserver** enverra une requête au webhook, qui décidera d’accepter ou de rejeter la requête.

---

## 🔹 Résumé concis

* Un **Admission Controller** intercepte les requêtes avant leur enregistrement dans **etcd**.
* Il existe deux types principaux :

    * **Mutating Admission Controller** : modifie la requête (ex : `DefaultStorageClass`).
    * **Validating Admission Controller** : valide ou rejette la requête (ex : `NamespaceLifecycle`).
* L’ordre d’exécution : **Mutating → Validating**.
* Les **Admission Webhooks** permettent de définir une logique personnalisée via :

    * **MutatingAdmissionWebhook**
    * **ValidatingAdmissionWebhook**
* Pour les utiliser :

    1. Déployer un **serveur Webhook** (Go, Python, etc.)
    2. L’héberger (interne ou externe au Cluster)
    3. Créer un objet **WebhookConfiguration** pour l’enregistrer dans Kubernetes.

---

