## Taints & Tolerations

- taints -> sur les noeuds
- tolerations -> sur les pods

Pour mettre à  jour les taints :
````yaml
kubectl taints node node_name key=value:taint-effect
````

taint-effect : ce qui arrive au pod s'il ne sont pas tolérés :
    
- NoSchedule : pas mis sur le noeud
- PreferNoSchedule : on va essayer de pas le mettre sur le noeud mais ce n'est pas garanti
- NoExecute : vire aussi les pods existants qui ne tolerent pas la taint

Pour mettre à  jour les tolerations (cà´té pod) :

```yaml
spec:
  containers:
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
```

On utilise pas les taints et tolerations pour dire aux pods d'aller sur tel ou tel noeud, c'est plutot le rà´le du node affinity.
On utilise les taints pour exclure les pods qui ne les tolerent pas.

## Node Selector & Node Affinity

````yaml
spec:
  containers:
  nodeSelector:
    label-key: label-value
````

Si on veut faire des selections plus complexes, il faut passer les node affinity :

````yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
````

- requiredDuringSchedulingIgnoredDuringExecution : si pas de noeud trouvé, non schedulé + si les label changent, restent en place
- preferredDuringSchedulingIgnoredDuringExecution : si pas de noeud trouvé, schedulé + si les label changent, restent en place

## Taints & Tolerations vs Node Affinity

Les taints & nodes ne permettent pas de dire "ces pods iront forcément sur ces noeud", si les autres neuds n'ont pas de taints, les pods pourront y aller.
Avec les affinity,on arrive à  mettre nos pods là  ou on veut mais on ne peut pas garantir que les autres n'y viendront pas
Il faut combiner les 2 pour s'assurer que les pods iront toujours sur les noeuds qu'on veut et que le autres n'y viendront pas.
      
A voir également, l'admission controller : [PodNodeSelector](https://medium.com/geekculture/k8s-admission-controller-podnodeselector-31e940b8b183) 

## Multi containers pod

Patterns :
    
- Sidecar : Collecte les logs et les envoie à  un serveur
- Ambassador : exemple : proxy des requetesDB vers la bonne DB suivant l'envir
- Adapter : exemple : adapte les logs avant de les envoyer à  un serveur

## downwardApi

Permet par exemple de récupérer un label au runtime :

```yaml
- name: APPLICATION_TYPE
  valueFrom:
    fieldRef:
      fieldPath: metadata.labels['app.kubernetes.io/name']
```
