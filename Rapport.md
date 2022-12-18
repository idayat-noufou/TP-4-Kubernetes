# Orchestration de conteneurs avec Kubernetes (Rapport)

## Déployer l'application sur Kubernetes

ID du cluster : /subscriptions/62229cd0-2ef4-4900-9cc3-0780b46d6178/resourcegroups/rg-tp4-kubernetes/providers/Microsoft.ContainerService/managedClusters/aks-tp4

### Déploiement de MongoDB
Expliquer les commandes avec Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
avec cette commande on installe Helm

```bash
helm version
```
On vérifie que Helm est bien installé (cette commande nous donne la version installée de Helm)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
On télécharge le repo Bitnami contenant le Chart MongoDB sur le cluster

On installe ensuite le chart MongoDB via cette commande
```bash
helm install mongodb bitnami/mongodb -n mongodb --create-namespace
```
bitnami/mongodb : emplacement du fichier
-n : permet de spécifier le nom du namespace, ici mongodb


```bash
kubectl get all,secrets,cm -n mongodb
```
permet de lister les différentes resources créées par la commande ci-dessus, notamment les secrets et configMap contenant le nom MongoDB. 


```bash
kubectl port-forward --namespace mongodb svc/mongodb 27017:27017
```
Permet de rediriger les requêtes des ports de la machine locale vers le cluster

```bash
kubectl get secret --namespace mongodb mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 -d
```
Cette commande permet de récupérerer le mot de passe de l'utilisateur root

On utilise ensuite le mot de passe dans cette commande permettant de se connecter à la base de données MongoDB
*mongodb://root:<PASSWORD>@localhost:27017*
içi, le mot de passe est "aRYIgmjckS"; on se connecte donc via un client mongodb à cette URI :
**mongodb://root:aRYIgmjckS@localhost:27017**


### 1) Déploiement de l'api sur le cluster

On crée un fichier api-deployment.yaml, en indiquant notamment :
- le nombre de replicas(nombre de pod créés) : 1
- le nom de l'image : yrez/ynov-api:latest
- le port du container : 3000

Une fois le fichier complété, on lance la création de la ressource via la commande
```bash
kubectl apply -f api-deployment.yaml
```
La ressource a son tour crée le pod

```bash
kubectl get all #permet d'afficher les ressources
kubectl describe deployment #permet d'inspecter deployment
kubectl describe pods
```

Pour accéder aux logs du pod, on tape cette commande 
```bash
kubectl logs -n default deployment/api-deployment -f
```

avec 
- *api-deployment* étant le nom du déploiement

On comprend grâce aux logs que l'api n'arrive pas à se connecter à la base de donnée ici MongoDB, l'api a besoin d'une connexion à une base de donnée pour démarrer, hors on ne lui a pour l'instant pas communiqué l'URL


### 2) Connexion de l'API à MongoDB

```bash
kubectl create secret generic mongodb-uri \
    --from-literal=mongodb-uri=mongodb://root:aRYIgmjckS@mongodb.mongodb:27017 
```
Permet de sauvegarder mongodb-uri dans un secret.

l'api devra ensuite accéder au service mongodb crée précédemment grâce à helm dans le namespace mongodb. 
le pod de l'api peut communiquer avec celui de mongodb grâce au nom du service de mongodb qui est dans nôtre cas *mongodb* l'uri de connection devrait donc ressembler à ça : *mongodb://root:<PASSWORD>@mongodb:27017* cependant, le pod de l'api se trouve dans le namespace *default* tandis que celui de mongodb se trouve dans le namespace *mongodb* d'où le fait que l'uri de connection ressemble plutôt à ça : **mongodb://root:<PASSWORD>@mongodb.mongodb:27017**. Le sens de *mongodb.mongodb* est *le service mongodb du namespace mongodb* 

On ajoute ensuite le secret dans le fichier de déploiement api-deployment.yaml dans la partie env,
on éxécute ensuite de nouveau la commande permettant de créer la ressource
```bash
kubectl logs -n default deployment/api-deployment -f
```
Cette fois, le Pod run correctement

### 3) Exposer l'API à l'intérieur du cluster à l'aide d'un service

On va ici créer un service permettant d'accèder aux pods à partir d'un nom unique, pour ce faire on crée un fichier de configuration du service api-service.yaml
On renseigne notamment :
- l'app : api : qui correspond à l'app spécifiée dans le fichier api-deployment.yaml
- le protocole de connexion : TCP
- le port permettant de communiquer : 3000

On crée ensuite le service avec la commande
```bash
kubectl apply -f api-service.yaml
```

### 4) Augmenter le nombre de replicas de l'api

Dans le fichier api-deployment.yaml on change la valeur de replicas de 1 à 5
On lance ensuite la commande permettant d'éxécuter le déploiement
```bash
kubectl apply -f api-deployment.yaml
```

### 5) Installer l'Ingress Controller NGINX

On ajoute le repo avec la commande suivante
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

On installe ensuite le Chart avec la commande
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx
```
La commande crée :
- un deployment
- un service
- deux secrets
- un pod
- une configmap

on ne crée pas de namespace içi donc toutes les ressources crée sont dans le même namespace que l'api (default).


### 6) Exposer l'API publiquement en utilisant un Ingress
Pour exposer l'api, on crée un fichier api-ingress.yaml

On éxécute ensuite la commande
```bash
kubectl apply -f api-ingress.yaml
```

L'api est désormais accessible depuis le hostname configuré dans le fichier api-ingress.yaml


Pour le schéma : 
Ingress Controller > ServiceApi > Requete Pod > Mongodb