---
title: Déploiement de Deepseek sur OpenShift
---

# Introduction

Cet article explique comment déployer **Deepseek** sur OpenShift, une plateforme de conteneurisation open-source largement utilisée pour l'orchestration de microservices. Nous allons détailler les étapes nécessaires pour déployer **Deepseek**, ainsi que l'installation de **Dify**, un outil pour l'intégration et la gestion des applications.

## Prérequis

Avant de commencer, assurez-vous d'avoir les éléments suivants :
- Un cluster **OpenShift** fonctionnel.
- **kubectl** configuré pour accéder à votre cluster OpenShift.
- L'image Docker **Deepseek** disponible pour l'exécution sur OpenShift.

## Étape 1 : Installer Dify

Pour installer **Dify**, exécutez la commande suivante dans votre terminal. Cette commande appliquera le fichier de déploiement de **Dify** :

```
kubectl apply -f https://raw.githubusercontent.com/Winson-030/dify-kubernetes/main/dify-deployment.yaml
```


Cela déploiera **Dify** sur votre cluster OpenShift. Une fois l'installation terminée, vous pouvez vérifier l'état du déploiement avec la commande suivante :

```
kubectl get pods -n dify
```


## Étape 2 : Déployer Deepseek sur OpenShift

### Création du fichier de déploiement

Créez un fichier de déploiement YAML simplifié pour **Deepseek**. Ce fichier déploiera l'application avec les paramètres nécessaires pour démarrer **Deepseek** sur votre cluster OpenShift.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deepseek
  namespace: dify
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deepseek
  template:
    metadata:
      labels:
        app: deepseek
    spec:
      containers:
        - name: deepseek
          image: deepseek/deepseek-v2:latest
          ports:
            - containerPort: 11434
              protocol: TCP
          command: ["ollama", "run", "deepseek-v2"]
          resources: {}
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext:
        runAsUser: 0
```

Application du fichier YAML
Pour appliquer ce fichier de déploiement sur OpenShift, exécutez la commande suivante :

```
kubectl apply -f deepseek-deployment.yaml
```

Cela déploiera Deepseek sur OpenShift avec la commande ollama run deepseek-v2 au démarrage.

Étape 3 : Vérifier le déploiement
Une fois le déploiement effectué, vous pouvez vérifier l'état des pods et des déploiements avec les commandes suivantes :

```
kubectl get pods -n dify
kubectl get deployments -n dify
```

Cela vous permettra de vous assurer que Deepseek est correctement déployé et fonctionne comme prévu.

Conclusion
En suivant ces étapes, vous avez déployé Deepseek sur votre cluster OpenShift avec la commande ollama run deepseek-v2, et vous avez installé Dify pour une gestion centralisée des applications. Vous pouvez maintenant commencer à utiliser Deepseek dans votre environnement OpenShift pour vos besoins d'intelligence artificielle.

