## Misc

- image nginx : port 80
- image busybox : pas d'entrypoint, arg=["sh"]
- kubectl : --no-headers -> ne mets pas les headers, on peut donc ajouter ```| wc -l```
- /bin/sh -c 'echo $var' : interprete la chaine de caractères qui suit (attention si on met des guillemets à  la place des quotes, $var sera remplacée)
- root user id = 0
    
### vi
 
- export KUBE_EDITOR=/usr/bin/vim
- :14 (ligne 14)
- selection : maj + V
- déplacement à  droite : >
- undo : u
- redo : .
- copy : y (yank), cut : d, paste : p
- echo "set ai et ts=2 sw=2 number" >> .vimrc

### shell

```shell
export ns=default
alias k='echo "namespace: $ns" && kubectl -n "$ns"'

alias setns="kubectl config set-context --current --namespace $1"
alias getns="kubectl config view | grep namespace"
```

### yaml

- multiligne : |-
- Il faut entourer les nombres et les booléens avec des ""  (même 2.3.4)

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

- shell form : For example, ENTRYPOINT node app.js (lançée dans un shell)   
- exec form : For example, ENTRYPOINT ["node", "app.js"] (non lançée dans un shell)

=> toujours favoriser l'exec form

Dans kube :

```command``` correspond à  ```EntryPoint``` dans Docker.  
```args``` correspond ```Cmd``` dans Docker.
