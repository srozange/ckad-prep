## Mode impératrif

### ReplicaSets

Scaler un replica set :
     
````shell
kubectl scale rs <replicaset-name> --replicas=5
````

### Services

Crée un pod et un service :

```shell
kubectl run nginx --image=nginx --port 80 --expose
```

```shell
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client --type=NodePort -o yaml
```

- va automatiquement utiliser comme selecteur les labels du pod (contrairement à  ```k create svc```)
- ```--port``` met à  jour les champs port et targetPort (impossibilité de préciser le port pour un NodePort, contrairement à  ```k create svc```)
- si pas de type, on est en ```ClusterIP```
- possibilité de creer un NodePort mais impossible de preciser un port
- si pas de port, va prendre le ```containerPort``` du container et le mettre dans ```targetPort```

```shell
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

- ne va pas utiliser comme selecteur celui d'un pod
- permet de préciser un port pour le ```NodePort```
- ```--tcp port:targetPort```

### Pods

````shell
kubectl run pod --image=image -- arg1 arg2
kubectl run pod --image=image --command -- cmd arg1 arg2
````

Mise à  jour d'une valeur immuable d'un pod :
````shell
k replace --force -f pod.yaml
````

## Custome Objects
### CRD : Custom Resource Definition

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

## Helm

```sh
helm pull --untar bitnami/wordpress
helm install release-1 bitnami/wordpress
```

## Pense bête

### Supprimer un namespace reticent
````shell
kubectl get ns ingress-nginx -o json | \
  jq '.spec.finalizers=[]' | \
  curl -X PUT http://localhost:8001/api/v1/namespaces/ingress-nginx/finalize -H "Content-Type: application/json" --data
````
