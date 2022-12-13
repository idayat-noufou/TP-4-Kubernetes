# Orchestration de conteneurs avec Kubernetes

## Introduction

Le but de ce TP sera de déployer l'API utilisée dans les autres TP dans un cluster Kubernetes.

## Prérequis

Ce TP nécessite d'avoir : 

- Un environnement Linux/Unix. Si vous utilisez Windows, assurez-vous d'avoir WSL2 d'installé et configuré.
- Git installé dans l'environnement Linux/Unix.
- Un compte étudiant Ynov

Pour la suite de ce TP, vous utiliserez exclusivement votre environnement Linux/Unix préalablement configuré. **Veillez à ne pas utiliser votre environnement Windows ainsi que Powershell**
 

## Création d'un cluster Kubernetes sur Azure avec AKS.

Dans cette partie vous allez créer un cluster Kubernetes sur le Cloud Provider Azure en utilisant le service Azure Kubernetes Service (AKS).

AKS est un service d'Azure qui permet de déployer un cluster Kubernetes managé de manière très simplifiée. Dans AKS, le Control Plan (les nodes Masters) est entièrement géré par Azure. De ce fait, il n'est pas possible d'accéder aux nodes Masters.

En tant d'étudiant Ynov, vous avez accès à $100 de crédit sur Azure. Si ce n'est pas déjà fait, vous pouvez activer votre compte à partir de [cette page](https://azure.microsoft.com/fr-fr/free/students/) en cliquant sur "Démarrer Gratuitement"

### Installation de l'Azure CLI

Azure CLI est un outil qui vous permettra d'interagir avec les services d'Azure.

Lancez cette commande pour installer Azure CLI : 

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Tester qu'Azure CLI est bien installé :

```bash
az version 
```

### Se connecter à Azure avec Azure CLI

Connectez vous à Azure avec Azure CLI en utilisant le commande suivante. Rentrez ensuite vos identifiants Ynov dans le navigateur.

```bash
az login
```

Si tout s'est bien passé, vous devriez avoir une sortie équivalente à celle-là :

```json
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "38e72bba-3c22-4382-9323-ac1612931297",
    "id": "829b1430-82e2-473b-990f-07f7ea8ce009",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Azure for Students",
    "state": "Enabled",
    "tenantId": "38e72bba-3c22-4382-9323-ac1612931297",
    "user": {
      "name": "alexis.bel@ynov.com",
      "type": "user"
    }
  }
]
```

### Création des ressources sur Azure

Maintenant que vous êtes connectés, vous allez pouvoir créer différentes ressources afin de créer le cluster Kubernetes.

Créer un groupe de ressources. Dans Azure chaque ressources doit être localisé dans un groupe de ressources : 

```bash
az group create --name rg-tp4-kubernetes --location francecentral
```

Créer un cluster AKS. Cette commande va créer un cluster AKS d'un node. La création du cluster prendra entre 5 et 10min. N'annulez pas la commande tant qu'elle n'est pas arrivé au bout ! 

```bash
az aks create -g rg-tp4-kubernetes -n aks-tp4 --enable-managed-identity --node-count 1 -s "Standard_B2ms" --generate-ssh-keys
```

Une fois créer vous allez recevoir une liste d'informations au format JSON identifiant le cluster.

Vérifiez que le cluster s'est bien créé en lançant la commande suivante qui permet de lister les clusters AKS dans vous souscription Azure :

```bash
az aks list -o table
```

### Se connecter au cluster 

Une fois créé, vous allez pouvoir récupérer les credentials qui vous permettront de vous connecter au cluster avec cette commande :

```bash
az aks get-credentials -g rg-tp4-kubernetes -n aks-tp4
```

Télécharger le client `kubectl` (Kube control) qui vous permet d'interagir avec un cluster Kubernetes (qu'il soit sur Azure ou non) avec la commande suivante :

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Valider que `kubectl` est bien installé :

```bash
kubectl version
```

Afficher la liste des nodes du cluster que vous venez de créer afin de valider votre authentication au cluster.

```bash
kubectl get nodes
```

Félicitation, vous avez créer votre premier cluster Kubernetes sur Azure et vous y êtes maintenant connecté.

### Ajouter le formateur en tant qu'administrateur sur le cluster 

Ces commandes vont permettrent de m'ajouter sur le cluster en tant qu'administrateur afin d'y accéder (nécessaire pour l'évaluation du TP)

```bash
RG_ID=$(az group show -n rg-tp4-kubernetes --query id -o tsv)

AKS_ID=$(az aks show -g rg-tp4-kubernetes -n aks-tp4 --query id -o tsv)

az role assignment create --assignee cc556a56-5d94-4e0a-aa73-3fd5efeec66c --role "Azure Kubernetes Service Cluster Admin Role" --scope $AKS_ID

az role assignment create --assignee cc556a56-5d94-4e0a-aa73-3fd5efeec66c --role "Contributor" --scope $RG_ID
```

### Télécharger Lens afin de naviguer sur le cluster.

Lens est une interface graphique qui vous permettra de vous connectez à votre cluster Kubernetes et d'avoir une visualisation graphique de l'ensemble des ressources. 

Lens est pratique afin de visualiser les objets Kubernetes et de faire les liens entre eux lorsque l'on débute sur Kubernetes.

Vous pouvez télécharger Lens à partir de ce [lien](https://k8slens.dev/)


## Déployer l'application sur Kubernetes

Dans le reste du TP, vous allez devoir réaliser les différentes étapes qui vous permettront de rendre l'API accessible depuis le cluster Kubernetes.

### Déploiement de MongoDB

Vous allez déployer MongoDB dans le cluster Kubernetes. Pour cela vous allez utilisez `helm`. Helm est un gestionnaire de paquet permettant de déployer des applications sur Kubernetes.

Installer Helm avec la commande suivante : 

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

 Vérifier que Helm est bien installé avec la commande suivante :

 ```bash
 helm version
```
Dans Helm, une application est packagé sur la forme d'un `chart`. Il est possible de créer ses propres Charts ou d'utiliser ceux déjà existant, maintenus par la communauté. 

Un chart Helm n'est rien de plus qu'une liste de ressources Kubernetes sous la forme de template `go`.

Il existe un chart Helm permettant de déployer MongoDB sur Kubernetes. 

Il faut dans un premier temps configurer le registry où est stocké le Chart avec la commande suivante :

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Vous pouvez ensuite installer une `release` de ce Chart en utilisant la commande suivante :

```bash
helm install mongodb bitnami/mongodb -n mongodb --create-namespace
kubectl port-forward --namespace mongodb svc/mongodb 27017:27017
```

Cette commande va installer le Chart Helm dans le namespace `mongodb` et créer toutes les ressources Kubernetes nécessaires. Dans Kubernetes, un `namespace` est un espace logique permettant de stocker des ressources Kubernetes. C'est pratique pour organiser les ressources et gérer les privilèges. Par défaut, si le namespace n'est pas spécifié, la ressource Kubernetes sera créée dans le namespace `default`.

Vous pouvez lister les ressources Kubernetes qui ont été créées automatiquement par Helm avec la commande suivante (Ou en allant directement les visualiser dans l'interface de Lens) :

```bash
kubectl get all,secrets,cm -n mongodb
```

Vous deviez les différentes ressources créée.

Lancez la commande suivante pour visualiser les logs du pod dans lequel se trouve le conteneur de MongoDB afin de vous assurer que le conteneur fonctionne correctement :

```bash
k logs -n mongodb deployment/mongodb -f
```

### Inspectez les différentes ressources créées

Prenez le temps d'inspecter les différentes ressources générées par Helm. Il est important de comprendre les différentes ressources qui ont été créées ainsi que leur rôle.

### Connection à MongoDB

MongoDB est exposé dans le cluster à l'aide d'un service de type `ClusterIP`. Ce type de service n'est accessible que depuis l'intérieur du cluster (et celui-ci est héberger sur Azure).

Il est cependant possible d'accéder au service en utilisant la technique du port-forwarding afin de rediriger les requêtes vers le service du cluster en utilisant les ports locaux de la machine cliente (votre PC). Attention cette technique n'est ni fiable, ni sécurisée, et ne doit être utilisé que dans le cadre de tests ou de débogage.


Faite un port-forward en utilisant la commande suivante :

```bash
kubectl port-forward --namespace mongodb svc/mongodb 27017:27017
```

Cela redirigera les requêtes du port 27017 de votre machine vers le port 27017 du service Mongodb du cluster. **Assurez vous que vous n'avez pas d'autre service qui tourne sur le port 27017 sur votre machine, sinon la commande de marchera pas.**

**Laissez cette commande active afin de conserver le port-forwarding !** 

Récupérez le mot de passe de l'utilisateur root avec cette commande (dans un autre terminal) :

```bash
kubectl get secret --namespace mongodb mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 -d
```

Vérifier que vous arrivez à vous connectez à MongoDB en utilisant Compass et la Connection String `mongodb://root:<PASSWORD>@localhost:27017`

### 1)  Déploiement de l'API sur le cluster

Maintenant que MongoDB est fonctionnel vous allez pouvoir déployer l'API. La première étape consiste à créer un `deployment` comme vu dans le cours.

Créez un fichier `api-deployment.yaml` et ajoutez la configuration permettant déployer l'API avec un seul replica. Vous pourrez pour cela utiliser l'image déjà packagée et accessible publiquement `yrez/ynov-api:latest` lors de la configuration du `deployment`. N'oubliez pas que l'API tourne sur le port `3000`. Cette information devra être utilisée dans la configuration du deployment.

Utilisez la [documentation sur les deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Une fois le fichier complété, vous pouvez créer la ressource avec la commande suivante : 

```bash
kubectl apply -f api-deployment.yaml
```

Assurez vous que le deployment a bien été créé et qu'il a a son tour créé un `pod`:

```bash
kubectl get all
kubectl describe deployment
kubectl describe pods
```

La commande `kubectl describe` permet d'afficher les détails sur une ressources et notamment les évènement qui y sont associés, très pratique pour déboguer une ressource.

Une fois que le pod est créé, vous allez remarquer qu'il redémarre sans arrêt, et c'est normal ! L'API est en erreur, consultez les logs du pod pour comprendre le problème.


### 2) Connection de l'API à MongoDB

L'API ne démarre pas car la connection à MongoDB n'a pas été réalisé et celle-ci est nécessaire. Afin de sécurisé la connection à l'API, il va falloir créer un `secret` contenant le connection string permettant de se connecter à l'API.

Utilisez la commande `kubectl create secret` pour créer un secret contenant la variable `MONGODB_URI` ainsi que la valeur associée.

Utilisez la [documentation sur les secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

**ATTENTION** : Il faudra utiliser le hostname `mongodb.mongodb` à la place de `localhost`. 

**Expliquez la signification de cette valeur.**

Une fois le secret créé, modifier la configuration du deployment afin d'injecter le secret sous la forme d'une variable d'environnement (il y a plusieurs manières de faire qui sont décrites dans le documentation des secrets).

### 3) Exposer l'API à l'intérieur du cluster à l'aide d'un service.

La prochaine étape consiste à déployer un service qui va permettre d'exposer l'ensemble des pods de l'API à l'intérieur du cluster à partir d'un nom unique.

Pour cela, créez un fichier `api-service.yaml` et ajoutez-y la configuration du service.

Utilisez la [documentation sur les services](https://kubernetes.io/docs/concepts/services-networking/service/).

Créez le service avec la commande suivante :

```bash
kubectl apply -f api-service.yaml
```

Une fois le service créé, assurez vous que pouvez accéder à l'API en faisant un port-forward du service sur l'api.

Essayez d'utiliser l'API :

```bash
# Ajouter un utilisateur
curl --location --request POST 'http://localhost:3000/users' \
--header 'Content-Type: application/json' \
--data-raw '{"email":"alexis.bel@ynov.com", "firstName": "Alexis", "lastName": "Bel"}'

# Lister les utilisateurs 
curl 'http://localhost:3000/users'
```

### 4) Augmenter le nombre de réplicas de l'API

Modifier la configuration du deployment de l'API et passez le nombre de réplicas à 5.

### 5) Installer l'Ingress Controller NGINX

Installer l'Ingress Controller NGINX en utilisant Helm en suivant la [documentation](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx).

Expliquer les différentes ressources qui ont été créé par Helm une fois le chart installé.

Assurez vous que l'Ingress Controller est fonctionnel en essayant de vous connectez à l'adresse IP du service de type LoadBalancer.


### 6) Exposer l'API publiquement en utilisant un Ingress.

Maintenant que l'Ingress Controller est fonctionnel, vous allez exposez l'API à l'extérieur du cluster. 

Pour cela, créez un fichier `api-ingress.yaml` et ajoutez y la configuration nécessaire. 

Utilisez la [documentation sur les ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).

Les ingress fonctionnent avec des noms de domaine. Il est donc nécessaire d'avoir un nom de domaine afin de créer l'ingress. 

Le plus simple est d'utiliser le service [nip.io](https://nip.io/) qui est un wildcard DNS basé sur l'IP. Vous pouvez donc utiliser un hostname de cette forme pour la configuration de l'ingress `api.<INGRESS_CONTROLLER_IP>.nip.io`, exemple : `api.1.2.3.4.nip.io`

Appliquer la configuration une fois celle-ci réalisée afin de créé l'ingress. 

Assurez vous que vous arrivez à accéder à l'API depuis l'hostname configuré dans l'ingress.

### 7) Réalisation du schéma réseau

Réaliser un schéma réseau qui permet d'expliquer le parcours de la requêtes HTTP depuis le navigateur d'un utilisateur jusqu'au pod de l'API.

### Bonus : Configurer de autoscaling sur l'API

Actuellement, le nombre de replicas est statique. Mettez en place de l'autoscaling sur l'API en fonction de la consommation en CPU des pods.








