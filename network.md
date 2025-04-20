### Accès inter namespace

````shell
curl <service>.<ns>.svc.cluster.local # service 
curl <pod_ip>.<ns>.pod.cluster.local # pod (ex: 172-17-0-1.default.pod.cluster.local)
````

### Versions 

**Pod** : v1  
**Service** : v1  
**ReplicaSet** : apps/v1  
**Deployment** : apps/v1

### Structure

````yaml
apiVersion: 
kind: 
metadata:
  name:
spec:
````

### cmd vs args

Dans docker :

- shell form : For example, ENTRYPOINT node app.js (lanà§ée dans un shell)   
- exec form : For example, ENTRYPOINT ["node", "app.js"] (non lanà§ée dans un shell)

=> toujours favoriser l'exec form

Dans kube :

```command``` correspond à  ```EntryPoint``` dans Docker.  
```args``` correspond ```Cmd``` dans Docker.

## Services (ClusterIp & NodePort)

Quand on veut acceder à  un ou plusieurs pod, on ne peut pas se baser sur leurs ip car il peuvent être detruits ou recréés, on passe donc par des services.    

Types :      
- NodePort : Ouvre un port sur chacun des noeuds puis redirige vers le bon pod (quelque soit le noeud du pod)    
- ClusterIp : Utilisé au sein du cluster (ex : un ensemble de pods communique avec un autre ensemble de pods)     
- LoadBalancer : Service à  l'extérieur du cluster (pas sur un noeud), (Avantage par rapport au NodePort : sur un NodePort, si le client est habitué à  attaquer un certain noeud et que le noeud tombe, il perdra l'accès alors que le load balancer redirigera vers un noeud valide)    

Exemple de NodePort :

````yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80 # port interne du service (obligatoire)
      targetPort: 80 # port d'attaque du container (si non fourni = port)
      nodePort: 30007 # port sur chacun des noeuds (range : 30000 - 32767 / si non fourni = provisioné)
````

Pourquoi la partie selector des services ne contient pas ```matchLabels``` ?   
=> Pour les services, dans spec.selector, on ne peut identifier les pods cibles uniquement par leurs labels alors que pour les déploiements, on peut avoir ```matchLabels``` ou ```matchExpressions```, exemple :    
````yaml
selector:
  matchExpressions:
    - {key: name, operator: In, values: [payroll, web]}
    - {key: environment, operator: NotIn, values: [dev]}
````

## Services ingress

Permet d'avoir un point d'entrée unique (url) qui va rediriger vers le bon service (suivant le path de l'url).
Permet également de gérer des certificats ssl.

Il faut un ingress controller (k8s n'en a pas un par defaut):
- Istio
- traefik
- HaProxy
- Nginx (maintenu par k8s)
- GCP (maintenu par k8s)

Ingress Resources : la configuration d'un ingress

Imperatif :

````shell
kubectl create ingress <ingress-name> --rule="host/path=service:port"
# Example : (si on une etoile dans le path, c'est un pathType de type prefix)
# si juste ="/wear*...", à§a intercepte tous les hosts
kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"
````

Example de yaml :           
````yaml
 apiVersion: networking.k8s.io/v1
 kind: Ingress
 metadata:
   name: ingress-wildcard-host
 spec:
   rules:
     - host: "foo.bar.com"
       http:
         paths:
           - pathType: Prefix
             path: "/bar"
             backend:
               service:
                 name: service1
                 port:
                   number: 80
     - host: "*.foo.com"
       http:: 
         paths:
           - pathType: Prefix
             path: "/foo"
             backend:
               service:
                 name: service2
                 port:
                   number: 80
````

pathType:
- Exact: Matches the URL path exactly and with case sensitivity.
- Prefix: (si = /aaa/bbb alors /aaa/bbb/ccc match)

si aucune rule ne match, on va vers le default backend (s'il existe)    
si pas de host, ca matche tout

Pour installer un ingress controller :
- Deployment avec l'ingress controller
- Service qui pointe vers le deployment
- ConfigMap référencé dans les args du deployment
- ServiceAccount

- nginx.ingress.kubernetes.io/rewrite-target: /     
=> So we specify the rewrite-target option. This rewrites the URL by replacing whatever is under rules->http->paths->path which happens to be /pay in this case with the value in rewrite-target      
=> on peut aussi avoir par ex : /$2 avec un matching que l'on fait dans path (ex : /something(/|$)(.*) )

- nginx.ingress.kubernetes.io/ssl-redirect: default to true when Ingress contains a certificate

## Network policies

### Basics

Ingress : traffic en entrée          
Egress : traffic en sortie     

Par défaut, tous les pods peuvent communiquer entre eux. Si on veut empecher à§a, il faut faire des NetworkPolicies :     

````yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress: # accès de l'api vers la db
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    # optionnel : permet de définir un namespace d'origine
    - namespaceSelector
        matchLabels:
          name: prod
    # permet de préciser une plage d'adresses ip source
    - ipBlock
        cidr: 192.168.5.10/32
    ports:
      protocol: TCP
     port: 3306
   egress: # accès de la db vers un serveur de backup
   - to:
    - ipBlock
        cidr: 192.168.5.10/32
    ports:
      protocol: TCP
      port: 80
````

Si namespaceSelector sans '-' au debut, alors il faut matcher podSelector + namespaceSelector pour passer (sinon c'est que des ou)

quand on fait un ingress, si la requete passe, la réponse est automatiquement permise (pas d'egress à  définir)
flannel ne supporte pas les NetworkPolicies !   

````shell
kubectl get networkpolicies
kubectl get networkpolicy
kubectl get netpol
````
