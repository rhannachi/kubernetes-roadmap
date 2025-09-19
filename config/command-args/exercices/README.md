### Exercice 1 : Créer un Pod avec l'image Ubuntu qui exécute un conteneur pour dormir 5000 secondes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-2
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "5000"]
```

Le conteneur exécutera la commande `sleep 5000` pour rester actif pendant 5000 secondes.

***

### Exercice 2 : Quel commande sera exécutée dans le Pod défini ?

YAML Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-green
  labels:
    name: webapp-green
spec:
  containers:
  - name: simple-webapp
    image: rhannachi/webapp-color
    command: ["--color", "green"]
```

Dockerfile de l'image (rappel) :

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--color", "red"]
```

**Réponse :**

Le Pod exécutera la commande définie dans `command` du YAML, qui remplace entièrement l’ENTRYPOINT dans l’image. Ici, `["--color", "green"]` n'est pas une commande valide seule, donc **cela provoquera probablement une erreur**.

Pour remplacer que les arguments, il faut garder l’ENTRYPOINT et modifier seulement les `args`.

***

### Exercice 3 : Quel commande sera exécutée dans le Pod défini ?

YAML Pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-green
  labels:
    name: webapp-green
spec:
  containers:
  - name: simple-webapp
    image: rhannachi/webapp-color
    command: ["python", "app.py"]
    args: ["--color", "pink"]
```

Dockerfile image :

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--color", "red"]
```

**Réponse :**

Le Pod exécutera

```
python app.py --color pink
```

car `command` remplace l'ENTRYPOINT et `args` remplace le CMD.

***

### Exercice 4 : Créer un Pod Kubernetes qui exécute une application web avec un fond vert.

- Nom du Pod : `webapp-green`
- Image Docker : `rhannachi/webapp-color`
- L’argument ligne de commande : `--color=green`

Commande :

```
$ kubectl run webapp-green --image=rhannachi/webapp-color -- --color green
```

