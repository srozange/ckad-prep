## Deployment strategies

- blue/green

Passe d'une ferme à  une autre : on a 2 deploiements avec chacun un label, on a un service qui pointe vers la v1, puis on fait pointer le service vers la V2

- canary

2 deploiements avec un label commun et un label de version, un service qui pointe vers ce label commun et on reduit le nombre de pod ds le deploiement canary
On crée un premier deployment, le selector et le pod doivent avoir le label commun et la version, pareil pour le 2e, puis on expose un des 2 deployment et on change le selecteur du svc pour le label commun
