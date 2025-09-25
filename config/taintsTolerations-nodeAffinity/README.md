# Taints and Tolerations VS nodeAffinity

La différence simplifiée est :

- **NodeAffinity** sert à **attirer ou orienter les Pods vers certains Nodes** grâce à des règles basées sur les labels des Nodes.  
  C’est un moyen pour un Pod de dire « je préfère (ou je dois absolument) aller sur un Node avec telle ou telle caractéristique ».

- **Taints and Tolerations** servent à **protéger les Nodes** en disant « ce Node ne veut pas accepter certains Pods, sauf s’ils ont la clé pour déverrouiller (tolérer) ce taint ».  
  Ici, ce sont les Nodes qui posent une contrainte stricte pour repousser les Pods non autorisés.

En résumé, **NodeAffinity, c’est le Pod qui exprime son désir d’aller sur certains Nodes** tandis que **Taints and Tolerations, c’est le Node qui verrouille l’accès aux Pods, permettant uniquement aux Pods tolérants de s’y poser**.

C’est pourquoi souvent on utilise les deux ensemble :
- On tainte un Node pour qu’il repousse par défaut tous les Pods non autorisés (protection des Nodes).
- Puis on utilise nodeAffinity sur les Pods autorisés pour s’assurer qu’ils ciblent bien ces Nodes (orientation des Pods).

Cette combinaison permet de garantir que seuls les Pods souhaités s’exécutent sur des Nodes spécifiques, par exemple des nœuds GPU, des nœuds à haute performance, ou des nœuds dédiés à un usage particulier.

Ainsi, taints/tolerations protègent les Nodes, nodeAffinity guide les Pods.

***

### Exemples simples de NodeAffinity

1. **Placer un Pod uniquement sur des Nodes avec un label spécifique**  
   On veut que le Pod tourne uniquement sur un Node avec le label `disktype=ssd` :

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

2. **Préférer un type de Node mais accepter d'autres types**  
   On souhaite privilégier les Nodes avec `zone=us-west` mais si aucun disponible, accepter d'autres Nodes :

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-west
```

***

### Exemples simples de Taints & Tolerations

1. **Tainter un Node pour qu'il refuse les Pods non tolérants**  
   On applique un taint sur un Node qui empêche tout Pod sans la bonne tolérance à y être planifié :

```
$ kubectl taint nodes node1 key=value:NoSchedule
```

Ici, le Node `node1` est marqué pour ne pas accueillir les Pods sans tolérance à `key=value`.

2. **Définir une tolérance dans un Pod pour accepter un Node tainté**

```yaml
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

Ce Pod pourra être programmé sur un Node tainté avec `key=value:NoSchedule`.

***

Ces deux mécanismes répondent à des besoins complémentaires :
- **NodeAffinity** guide où les Pods veulent aller.
- **Taints & Tolerations** imposent une restriction stricte côté Node pour protéger ce Node.


