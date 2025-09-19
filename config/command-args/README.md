# Command & Args

Dans un fichier YAML Kubernetes, on peut configurer certaines parties de ce qui est défini dans un Dockerfile, mais uniquement côté déploiement et exécution du conteneur, pas la phase de build.

Voici les parties du Dockerfile qu’on peut configurer dans un YAML Kubernetes :

| Partie Dockerfile               | Équivalent / Configuration dans YAML Kubernetes                      |
|-------------------------------|----------------------------------------------------------------------|
| `FROM`                        | Spécification de l'image dans le champ `spec.containers.image`       |
| `CMD`                         | Spécification des commandes par défaut via `args`                    |
| `ENTRYPOINT`                  | Peut être remplacé ou configuré via `command`                        |
| `ENV`                         | Variables d’environnement dans `spec.containers.env`                 |
| `EXPOSE`                      | Ports exposés listés dans `spec.containers.ports`                    |
| `VOLUME`                      | Volumes montés dans `spec.volumes` et `spec.containers.volumeMounts` |
| `WORKDIR`                     | Chemin de travail défini dans `spec.containers.workingDir`           |

### Exemple

Dockerfile :

```dockerfile
FROM nginx:alpine
ENV MY_VAR=1
EXPOSE 80
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

Correspondance Kubernetes YAML (extrait) :

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    env:
    - name: MY_VAR
      value: "1"
    ports:
    - containerPort: 80
    command: ["nginx"]
    args: ["-g", "daemon off;"]
```

### Ce qui ne se configure pas dans YAML Kubernetes :
- Instructions dans un Dockerfile liées à la phase de build (ex: `RUN`, `ARG`, `COPY`, `ADD`). Celles-ci doivent être appliquées lors de la construction de l’image (Dockerfile).

