# Kubernetes CKAD Roadmap

***

## Core Concepts
1. **[Docker & Containerd](core-concepts/README-docker-containerd.md)**
2. **[Architecture](core-concepts/README-architecture.md)**
3. **[Pod](core-concepts/pod/README.md)**
4. **[ReplicaSets](core-concepts/replicaset/README.md)**, 
   1. [Exercice](core-concepts/replicaset/exercices/README.md)
5. **[Deployment](core-concepts/deployment/README.md)**
   1. [Exercice](core-concepts/deployment/exercices/README.md)
6. **[Namespace](core-concepts/namespace/README.md)**
   1. [Exercice](core-concepts/namespace/exercices/README.md)
7. **[Exercices](core-concepts/README-exercices.md)**

## Config
1. **[Command & Args](config/command-args/README.md)**
   1. [Exercice](config/command-args/README.md)
2. **[Env configMap secret](config/env-configmap-secret/README-env-configmap.md)**
   1. [Exercice](config/env-configmap-secret/deployment-env-configmap.yaml)
3. **[Secret](config/env-configmap-secret/README-secret.md)**
   1. [Exercice](config/env-configmap-secret/deployment-secret.yaml)
4. **[Encryption configuration](config/env-configmap-secret/README-security-etcd.md)**
5. **[Security context](config/securityContext/README.md)**
   1. [Exercice](config/securityContext/pod-security-context.yaml)
6. **[Resource](config/resources/README.md)**
7. **[Service account](config/serviceAccount/README.md)**
   1. [Exercice](config/serviceAccount/exercices/service-account-config.yaml)
8. **[Taints and Tolerations](config/taints-tolerations/README.md)**
   1. [Exercice](config/taints-tolerations/exercices/README.md)
9. **[NodeSelector and NodeAffinity](config/nodeSelector-nodeAffinity/README.md)**
   1. [Exercice 1](config/nodeSelector-nodeAffinity/exercices/README-simple.md)
   2. [Exercice 2](config/nodeSelector-nodeAffinity/exercices/README-advanced.md)
10. **[TaintsTolerations VS NodeAffinity](config/taintsTolerations-nodeAffinity/README.md)**
    1. [Exercice](config/taintsTolerations-nodeAffinity/exercices/README.md)

## Multi Container
1. **[Multi container](./multi-container/README-multi-container.md)**

## Observability
1. **[Readiness probe](observability/readiness-liveness/README-readiness.md)**
2. **[Liveness probe](observability/readiness-liveness/README-liveness.md)**
3. **[Logging](observability/logging/README.md)**
4. **[Monitoring](observability/monitoring/README.md)**
5. **[Labels, Selectors and Annotations](pod-desing/labels-selectors-annotations/README.md)**
6. **[RollingUpdates Rollbacks Deployment](pod-desing/rollingUpdates-rollbacks-deployment/README.md)**
   1. [Exercice](pod-desing/rollingUpdates-rollbacks-deployment/exercices/README.md)
7. **[Jobs](pod-desing/jobs/README.md)**
8. **[CronJob](pod-desing/cronJob/README.md)**
   1. [Exercice](pod-desing/cronJob/exercices/README.md)

## Service & Networking
1. **[NodePort](services/nodePort/README.md)**
2. **[ClusterIp](services/clusterIp/README.md)**
   1. [Exercice](services/clusterIp/exercices/README.md)
3. **NetworkPolicy**
   1. **[NetworkPolicy Introduction](services/networkPolicy/README.md)**
   2. **[Exercice](services/networkPolicy/exercices/README.md)**
   3. **[NetworkPolicy Ingress/Egress](services/networkPolicy/README-2.md)**
4. **[Ingress Networking](services/ingress-networking/README.md)**
   1. **[Exercice 1](services/ingress-networking/example-1/README.md)**
   2. **[Exercice 2](services/ingress-networking/example-2/README.md)**

## Persistence
1. **[Storage in Docker](persistence/storage-docker/README.md)**
2. **[Volumes](persistence/volumes/README.md)**
3. **[PV & PVC](persistence/pv-pvc/README.md)**
   1. **[Exercice 1](persistence/pv-pvc/exercices/README.md)**
   2. **[Exercice 2](persistence/pv-pvc/exercices/README-exercice-2.md)**
4. **[Provisioning Static/Dynamic](persistence/static-dynamic-provisioning/README.md)**
   1. **[Exercice](persistence/static-dynamic-provisioning/exercices/README-exercice-1.md)**
5. **[StatefulSet](persistence/statefulset/README.md)**
   1. **[Exercice TODO](persistence/statefulset/exercices/README.md)**

## Security
1. **[Authentification](security/authentification/README.md)**
2. **[KubeConfig](security/kubeConfig/README.md)**
   1. **[Exercice 1](security/kubeConfig/exercices/README.md)**
3. **[ApiGroups](security/apiGroups/README.md)**
4. **[Authorization](security/authorization/README.md)**
5. **[RBAC](security/RBAC/README.md)**
   1. **[Exercice 1](security/RBAC/exercices/README.md)**
6. **[ClusterRoles](security/clusterRoles/README.md)**
7. **[Admission Controllers](security/admissionControllers/README.md)**
   1. **[Exercice 1](security/admissionControllers/exercices/README.md)**
   2. **[Exercice 2](security/admissionControllers/exercices/README-2.md)**
8. **[Validation Mutation AdmissionControllers](./security/validation-mutation-admissionControllers/README.md)**
   1. **[Exercice 1](security/validation-mutation-admissionControllers/exercices/README.md)**


## (TODO) Service mesh Istio Kubernetes
## (TODO) Elasticsearch and Kibana on Kubernetes (logs, métriques, événements)

