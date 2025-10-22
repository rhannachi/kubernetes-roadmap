# Webhook - ValidatingAdmissionWebhook (TODO)

**Exemple complet, prêt à tester sur Minikube**, d’un **Admission Webhook** fonctionnel déployé **dans le Cluster**.
Cet exemple illustre un **ValidatingAdmissionWebhook** qui **rejette** la création d’un Pod si son nom contient le mot “test”.

## Objectif du Webhook

> Rejeter la création de tout Pod dont le nom contient le mot **“test”**.

---

## Structure des fichiers

Crée un dossier :

```bash
mkdir admission-webhook-demo && cd admission-webhook-demo
```

Dedans, crée les fichiers suivants :

```
admission-webhook-demo/
├── server.py
├── Dockerfile
├── manifests/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── webhook-config.yaml
│   ├── certificate.yaml
│   └── rbac.yaml
```

---

## 1. Code Python du serveur Webhook (`server.py`)

Ce serveur Flask écoute sur `/validate` et renvoie `allowed: false` si le nom du Pod contient "test".

```python
from flask import Flask, request, jsonify
import base64
import json

app = Flask(__name__)

@app.route("/validate", methods=["POST"])
def validate():
    review = request.get_json()
    pod_name = review["request"]["object"]["metadata"]["name"]

    allowed = "test" not in pod_name
    message = "Pod name contains forbidden word 'test'" if not allowed else "Pod accepted"

    response = {
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": review["request"]["uid"],
            "allowed": allowed,
            "status": {"message": message}
        }
    }
    return jsonify(response)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8443, ssl_context=("/certs/tls.crt", "/certs/tls.key"))
```

---

## 2. Dockerfile

```Dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY server.py /app
RUN pip install flask
CMD ["python", "server.py"]
```

---

## 3. Générer un certificat TLS auto-signé

Kubernetes **exige TLS** pour les webhooks.
Minikube facilite cela avec `openssl`.

### Exécute sur ton terminal :

```bash
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt -days 365 -nodes -subj "/CN=webhook-service.webhook-system.svc"
kubectl create namespace webhook-system
kubectl create secret tls webhook-server-tls \
  --cert=tls.crt --key=tls.key \
  -n webhook-system
```

---

## 4. Manifeste du Deployment (`manifests/deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-server
  namespace: webhook-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook-server
  template:
    metadata:
      labels:
        app: webhook-server
    spec:
      containers:
      - name: webhook-server
        image: webhook-server:latest
        imagePullPolicy: Never   # car on va utiliser une image locale Minikube
        ports:
          - containerPort: 8443
        volumeMounts:
          - name: tls-certs
            mountPath: /certs
            readOnly: true
      volumes:
        - name: tls-certs
          secret:
            secretName: webhook-server-tls
```

---

## 5. Service (`manifests/service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
  namespace: webhook-system
spec:
  ports:
  - port: 443
    targetPort: 8443
  selector:
    app: webhook-server
```

---

## 6. ValidatingWebhookConfiguration (`manifests/webhook-config.yaml`)

⚠️ Remplace le champ `caBundle` par la valeur Base64 du certificat **tls.crt**.

Tu peux l’obtenir ainsi :

```bash
cat tls.crt | base64 | tr -d '\n'
```

Puis colle cette valeur dans `caBundle`.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-validator
webhooks:
  - name: pod-validator.webhook-system.svc
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service:
        name: webhook-service
        namespace: webhook-system
        path: "/validate"
      caBundle: <BASE64_CERT>
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
```

---

## 7. RBAC (`manifests/rbac.yaml`)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webhook-server-sa
  namespace: webhook-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: webhook-server-role
  namespace: webhook-system
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: webhook-server-binding
  namespace: webhook-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: webhook-server-role
subjects:
  - kind: ServiceAccount
    name: webhook-server-sa
    namespace: webhook-system
```

---

## 8. Construction et déploiement sur Minikube

### Étape 1 — Charger l’image dans Minikube :

```bash
eval $(minikube docker-env)
docker build -t webhook-server:latest .
```

### Étape 2 — Appliquer les manifests :

```bash
kubectl apply -f manifests/
```

### Étape 3 — Vérifier :

```bash
kubectl get pods -n webhook-system
kubectl logs -n webhook-system deploy/webhook-server
```

---

## 9. Test du Webhook

### ✅ Pod accepté :

```bash
kubectl run valid-pod --image=nginx
```

> Le Pod est créé sans erreur.

### ❌ Pod rejeté :

```bash
kubectl run test-pod --image=nginx
```

> Erreur :

```
Error from server: admission webhook "pod-validator.webhook-system.svc" denied the request: Pod name contains forbidden word 'test'
```

---

## Résumé du fonctionnement

| Étape | Description                                                            |
| ----- | ---------------------------------------------------------------------- |
| 1     | L’utilisateur crée un Pod (`kubectl run`)                              |
| 2     | La requête est interceptée par le `ValidatingWebhookConfiguration`     |
| 3     | Le **kube-apiserver** envoie un **AdmissionReview** au serveur webhook |
| 4     | Le serveur analyse le nom du Pod                                       |
| 5     | Si le nom contient “test”, il renvoie `allowed: false`                 |
| 6     | Le Pod est rejeté, et l’erreur s’affiche côté client                   |

---
