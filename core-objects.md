## Labels, Annotations et Selectors

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
  annotations:
    ...
```

Sélectors dans les Job, Deployment, ReplicaSet, et DaemonSet :

```yaml
spec:
  selector:
    matchLabels:
      component: redis
  template:
```

Pourquoi il y a un selector dans les replicaset/deployment/... ? Parceque qu'ils peuvent aussi gérer des objets qui ne sont pas dans le template mais qui sont déjà  labelisés.   

Labelisation des objets (impératif) :
````shell
kubectl label nodes <node-name> label-key=label-value
````

Pour supprimer un label (impératif) :
````shell
kubectl label nodes <node-name> label-key=label-value-
````


## Rolling updates

```shell
# différentes commandes :
kubectl rollout [status|history|undo] deployment/<deployment_name>
# permet d'afficher un rollout donné :
kubectl rollout history deployment nginx --revision=1
# sauve la commande et la rend visible ds l'historique (au passage : <container_name>=<new_image>)
kubectl set image deployment nginx nginx=nginx:1.17 --record
```

### deployments

```yaml
minReadySeconds: 5
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

- minReadySeconds tells Kubernetes how long it should wait until it creates the next pod. 
    - This property ensures that all application pods are in the ready state during the update.
- maxSurge specifies the maximum number (or percentage) of pods above the specified number of replicas. 
    - In the example above, the maximum number of pods will be 5 since 4 replicas are specified in the yaml file.
- maxUnavailable declares the maximum number (or percentage) of unavailable pods during the update. If maxSurge is set to 0, this field cannot be 0

## Jobs

Un pod classique s'il rend la main sera relancé automatiquement par kube sauf si l'on met la ```restartPolicy``` à  ```Never```.
Les jobs permettent de gérer le parallelisme avec plusieurs containers qui s'executent en parallèle.

```yaml
spec:
  completions: 3 # nb de pod à  execute et qui doivent sortir en succès
  parallelism: 3 # nb de pod en // (par défault on les execute l'un après l'autre)
  backoffLimit: 3 # nombre d'essais max avant de condiderer que l'on est en ko (default : 6)
  template :
```

## CronJobs

```yaml
spec:
  schedule: "* * * * *"
  jobTemplate : // Recoit le job
    completions:
    parallelism :
    templace :
       spec :
         containers :
```

Format de ```schedule``` (cron \* \* \* \* \*) :
   
- Minutes (*: toutes les minutes, */2 toutes les 2 minutes, 2 à  la 2e minute de chaque heure)
- Heure
- Jour du mois
- Mois
- Jour de la semaine

## Readiness et Liveness probes

- pod state : pending, containerCreating (quand affecté à  un node), running
- pod conditions (array of boolean) : podSchedules (le pod est sur un noeud), initialized (initContainers), containersReady, ready (peuvent être mappé à  un service, prêt à  servir des requetes)

```yaml
containers:
  - name: container1
    readynessProbe:
      exec: # process dans le conteneur
        command:
          - cat
          - /tmp/healthy
      # ou
      httpGet: # appel http
        path: /healthz
        port: 8080
      # ou
      tcpSocket:
        port: 80
      # avec :
      intialDelaySeconds: # on attend x seconds avant de commencer
      periodSeconds: # on teste toute les x secondes 
      failureTresholds: # nombre de fois ou on essaye, default 3
```

readyness -> liveness

A etudier : startupProbe vs readyness ?

## SecurityContext

Peut etre appliqué au niveau du **pod** ou au niveau du **container**.

```yaml
spec:
  securityContext:
      runAsUser: 1000
      capabilities:
          add: ["MAC_ADMIN"]
```

```runAsNonRoot: true``` balance une erreur si le container est lancer en root

## ServiceAccount

2 types de services :
   
- user account (utilisé par un humain) 
- service account (utilisé par des machines)

```shell
kubectl create serviceaccount nom_du_sa
```
avant la 1.23 : créé automatiquement un token stocké dans un secret qui est lié à  ce serviceaccount

pour inclure les infos d'un serviceAccoun dans un pod : 

```yaml
spec.serviceAccountName: nom_du_sa
```

on peut demander à  k8s de ne pas automatiquement monter par défaut le service account : 

```yaml
spec.automountServiceAccountToken: false
```

- depuis la **1.23** : les token sont générés à  la demande et expirent, l'api est appelée en auto qd un pod est lancés
- depuis la **1.24** : les secret/token ne sont plus créés automatiquement et il faut lancer la commande : ```k create token nom_du_sa``` (il va être créé avec une date d'expiration contrairement à  avant la 1.24)

Si on veut faire à  l'ancienne mode il faut creer un secret de :
```yaml
type: kubernetes.io/service-account-token
annotation: kubernetes.io/sevice-account.name=nom_du_sa
```

## ResourceRequest

```yaml
containers:
  - name:
    resources: 
      requests: # Ce que le container est assuré de recevoir (donc on essaye de trouver un node dispo)
        memory: "1Gi"
        cpu: "1"
      limits: # ce que le container a le droit d'utiliser
        memory: "" # kubernetes ne controle pas la memoire donc le container peux aller au dessus, si c'est le cas il sera dégagé
        cpu: "2" # kubernetes controle le cpu donc le container ne pourra pas aller au dessus
```

=> si uniquement les limites sont renseignées alors requests prend la valeur de limits
   
```OOMKilled``` : out of memory

## ResourceQuota

Par namespaces

## ConfigMap

````shell
kubectl create configmap <config_name> --from-literal=KEY=VALUE --from-literal:
kubectl create configmap <config_name> --from-file=config.properties
````

````yaml
envFrom:
 - configMapRef: # Injecter toutes les entrées d'une configmap en variable d'envir
   name: config_name

env:
 - name :
   valueFrom:
      configMapKeyRef: # Injecter une entrée de la config map
        name:
        key:
````

## Secret

````shell
k create secret generic secret_name --from-literal=pwd=toto
k create secret generic secret_name --from-file=file
````

Le create encode en auto en base64 :
````shell
# encode value
echo "toto" | base64
# decode value 
echo "kokoko" | base64 --decode
````

### variables d'envir

````yaml
envFrom:
 - secretRef: # Injecter toutes les entrées d'un secret en variable d'envir
   name: secret_name

env:
 - name :
   valueFrom:
      secretKeyRef: # Injecter une entrée de la config map
        name:
        key:
````

### volumes

````yaml
spec:
  containers:
    - name:
      volumeMounts: # va créér un fichier par item du secret
        - name: secret-volume
          mountPath: /test # repertoire contenant les fichiers

volumes:
  name: secret-volume
  secret:
   secretName: mysecret
   items: # optionnel,par exemple si on veut réduire, ce que l'on injecte en variables d'envir)
     - key: key1
       path: path1
````

=> Dans ce cas, crée un volume :  
- Si le répertoire existe déjà  dans le container alors le contenu sera dégagé
- Si le secret est mis à  jour, le contenu du volume sera aussi mis à  jour (le rep pointe vers un lien symbolique)

subPath :

```yaml
spec:
  containers:
    - name:
      volumeMounts:
        - name: secret-volume
          mountPath: /bin/toto #nom du fichier
          subPath: key1 (nom de la clé de secret)
```

=> Dans ce cas, kube va créer uniquement le fichier mais il ne sera pas mis à  jour en cas d'update du secret

## Volumes
- hostPath : Ce type de volume monte un fichier ou un répertoire du système de fichiers du nÅ“ud hà´te dans votre pod.
- emptyDir : Il s'agit d'un type de volume créé lors de la première affectation d'un pod à  un nÅ“ud. Il reste actif tant que le pod s'exécute sur ce nÅ“ud. Le volume est initialement vide et les conteneurs du pod peuvent lire et écrire les fichiers dans le volume emptyDir. Une fois le pod retiré du nÅ“ud, les données du répertoire vide sont effacées.
- secret : Un volume secret est utilisé pour transmettre des informations sensibles, telles que des mots de passe, à  des pods.
- ConfigMap : La ressource ConfigMapfournit un moyen dâ€™injecter des données de configuration dans les pods.
- persistentVolumeClaim : Un volume persistentVolumeClaim est utilisé pour monter un PersistentVolume dans un pod. Les volumes persistants sont un moyen pour les utilisateurs de Â«revendiquerÂ» un stockage durable (tel qu'un disque persistant GCE ou un volume iSCSI) sans connaà®tre les détails de l'environnement cloud en particulier

### Exemple avec un hostPath

````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-srius
spec:
  template:
   spec:
      containers:
      - name: tomcat
        image: softcu-nexus.si.francetelecom.fr/devops/softbuilder-tomcat:0.0.4
        volumeMounts:
        - mountPath: /usr/local/tomcat/filer
          name: filer-volume
          readOnly: true
      volumes:
      - name: filer-volume
        hostPath:
          path: /var/opt/data/flat/33i/MOCK
          type: Directory
````

### Persistent volumes

PV :
````yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  persistentVolumeReclaimPolicy: Retain|Delete|Recycle #default=Retain
  accessModes:
    - ReadWriteOnce
  nfs: # ici, on met le type de volume (ex : hostPath, awsElasticBlockStore...)
    path: /tmp
    server: 172.17.0.2
````

accessModes :    
- ReadWriteOnce : le volume peut être monté en lecture-écriture par un seul nÅ“ud
- ReadOnlyMany : le volume peut être monté en lecture seule par plusieurs nÅ“uds
- ReadWriteMany : le volume peut être monté en lecture-écriture par de nombreux nÅ“uds

=> PersistentVolumes donâ€™t belong to any namespace (see figure 6.7). Theyâ€™re cluster-level resources like nodes.

PVC :
````yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
````
=> va être matché (accessModes, espace dispo) avec un PV      

Dans un Pod :

````yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
````

=> On peut referencer plusieurs fois le même PVC dans plusieurs Pod si accessMode = ReadWriteMany

### Storage class

Un Storage class est un objet qui est référence dans un PVC et qui crée automatiquement un PV
