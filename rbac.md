## Securité

### Users

2 types de users : des personnes et des robots (service accounts). k8s/kubectl ne gère pas nativement les personnes.

Les users de types personnes sont gérés par le kube-apiserver :      

- On peut specifier des user/password ds un fichier csv que l'on passe au kube-apiserver (--basic-auth-file=*.csv)   
- On peut aussi spécifier des token dans un fichier csv (--token-auth-file=*.csv)
- certificats
- systèmes externes
=> les 2 premiers modes sont déconseillés car unsecure

### api

/api et /apis permettent d'acceder aux fonctionnalités du serveur :     

- /api (core group) : ns, pods, rc, events, services... (pas les déploiements!!!!)
      - api groups : vide
- /apis (named group) : 
      - api groups : /apps, /extensions, /networking.k8s.io, /storage*, authentication
      - sous ces api : /apps/v1/deployments
=> Tout à§a correspond au champ apiGroups des Role et ClusterRole (ex vide pour les core groups ou app pour les deployments)
       
pour obtenir une description des api ```curl http://localhost:6443 -k``` (il faut d'abord faire un ```kubectl proxy```)

### authorisations

4 mecanismes d'autorisation :     

- Node : Le node authoriser repond aux kubelet (qui doit avoir un nom du type system:node:node01)
- ABAC (attribute based) : 
      - user ou groupe de users => se fait avec un object policy (du style : michel peux void les pods ou en créer)
      - mais à  chaque fois qu'on veut faire une modif, il faut changer le fichier et restarter le kube api serveur
- RBAC (role based)
      - on définit des rà´les (ex: développeur)
- Webhook : permet de déléguer à  un système tier

2 autres modes : AlwaysAllow (par défaut) & AlwaysDeny (passé en argument du kube api server)       

Les authorisations se font à  la chaine (ex avec un ordre defini en arg du kube api serveur : node -> rbac -> webhook) jusqu'à  que un dise oui

### RBAC : Role based acces control

Activation de RBAC : ```kube-apiserver --authorization-mode=Example,RBAC --other-options --more-options```
     
Création d'une rà´le :
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group / pour les named groups, il faut préciser
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
  resourceNames: ["mypod"] # optionnel, permet de reduire le scope à  certains objets
```

RoleBinding, permet de linker le groupe au user :
```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jane # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

Pour les Role et RoleBinding, il faut forcément spécifier un namespace (alors que ce n'est pas nécessaire pour les ClusterRole et ClusterRoleBinding)

Avec kubectl :
```shell
# resoource peut prendre une liste d'objets séparés par des virgules (l'api groups est récupéré automatiquement)
kubectl create role pod-reader --verb=get --resource=pods --resource-name=readablepod --resource-name=anotherpod
kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme
```

Pour tester ses droits :
```shell
kubectl auth can-i create deployments
kubectl auth can-i delete nodes --as srozange (impersonate srozange)
```


Pour retrouver l'api groups d'un objet ! :
```shell
kubectl api-resources
```

## kubeconfig

le fichier config a 3 parties :      
- clusters
- contexts : quel user est associé au cluster / on peut y préciser un namespace
- users

Comment kubectl sait quel context choisir ? il y a un champ current-context ds le fichier.    

```shell
kubectl config view
kubectl config use-context prod-user@production
kubectl config set-context --current --namespace=default
```

## admission controllers

Verifie l'objet avant de l'admettre. Exemple :     

- AlwaysPullImage
- NamespaceExists (activé par défaut) => remplacé par le NamespaceLifecyle
- NamespaceAutoProvision (non activé par défaut) => remplacé par le NamespaceLifecyle
- DefaultStorageClass

Liste des admission controller :
```shell
k exec -it kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```


Pour ajouter ou supprimer un admission controller (dans /etc/kubernetes/manifests/kube-apiserver.yaml)
```yaml
    - --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
    - --disable-admission-plugins=DefaultStorageClass
```

Types d'admission controller :

- Validating (ex : est-ce que le ns existe ?)
- Mutating (ex : DefaultStorageClass qui ajoute une storage class par défaut)
- Il peut y en avoir qui font les 2

En général les mutating sont exécutés avant les validating.

Pour faire ses propres admission controller, on peut utiliser :

- MutatingAdmissionWebhook
- ValidatingAdmissionWebhook

=> ils appellent tous les 2 une resource à  nous que l'on doit déployer (ds le cluster ou ailleurs)

Ex :
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  rules:
  - apiGroups:   [""] # on peut réduire à  certaines opérations
    apiVersions: ["v1"]
    operations:  ["CREATE"]
    resources:   ["pods"]
    scope:       "Namespaced"
  clientConfig:
    service: # on peut avoir une url à  la place si on veut taper vers l'exterieur
      namespace: "example-namespace"
      name: "example-service"
    caBundle: <CA_BUNDLE>
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

Ils sont paramétrés ds : ```/etc/kubernetes/manifests/kube-apiserver.yaml```

## API versions

Les versions :

- alpha (ex: v1alpha1) => doivent être activées via un flag
- beta (ex: v1beta1) => activées par défaut, peut avoir des bugs mineurs
- stable (ex: v1)

Si l'on a plusieurs versions :

- prefered version : version par defaut (renvoyé par kubectl  par ex)
- storage version : version dans laquelle un objet est stored dans etcd (habituellement la même que la prefered mais peut être overridée)

On peut activer une version d'api via le lancement du kube-apiserver :
```shell
  --runtime-config=batch/v2alpha1
```

```shell
kubectl explain job

# Version préférée d'un objet :
kubectl proxy 8001
curl localhost:8001/apis/authorization.k8s.io

ou

kubectl get --raw /apis/authorization.k8s.io
```

### depreciations

>API elements may only be removed by incrementing the version of the API group.
      
>API objects must be able to round-trip between API versions in a given release without information loss, with the exception of whole REST resources that do not exist in some versions.
 
>Other than the most recent API versions in each track, older API versions must be supported after their announced deprecation for a duration of no less than:
     
- GA: 12 months or 3 releases
- Beta: 9 months or 3 releases
- Alpha: 0 releases

```shell
kubectl convert -f old_file --output-version= apps/v1
```
