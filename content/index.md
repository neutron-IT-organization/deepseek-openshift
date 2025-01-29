---
title: Home
---

```markdown
# Déploiement de DeepSeek sur OpenShift


Cette documentation explique comment déployer DeepSeek sur un cluster OpenShift, en créant un namespace, en attribuant des privilèges au service account, en installant les composants nécessaires, et en exposant l'application via une route.

## 1. Créer un Namespace `deepseek`

Pour commencer, créez un namespace dédié à DeepSeek sur OpenShift :

```bash
oc create namespace deepseek
```

## 2. Ajouter la SCC `privileged` au service account `default`

Ensuite, attribuez la SCC (Security Context Constraint) `privileged` au service account `default` dans le namespace `deepseek` :

```bash
oc adm policy add-scc-to-user privileged -z default -n deepseek
oc adm policy add-scc-to-user privileged -z dify-redis -n deepseek
oc adm policy add-scc-to-user privileged -z dify-postgres -n deepseek
oc adm policy add-scc-to-user privileged -z dify-weaviate -n deepseek

```

## 3. Installer le déploiement d'Ollama

Déployez Ollama dans le namespace `deepseek` avec le fichier de configuration suivant :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: deepseek
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      securityContext:
        runAsUser: 0
      containers:
        - name: ollama
          image: ollama/ollama:latest
          ports:
            - containerPort: 11434
              protocol: TCP
          resources: {}
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
```

Appliquez cette configuration avec la commande suivante :

```bash
oc apply -f ollama-deployment.yaml
```

## 4. Exécuter `ollama run deepseek-v2`

Une fois le déploiement d'Ollama effectué, exécutez la commande suivante à l'intérieur du pod pour lancer `deepseek-v2` :

```bash
oc exec -it <pod-name> -- ollama run deepseek-v2
```

Remplacez `<pod-name>` par le nom du pod Ollama que vous pouvez obtenir avec la commande `oc get pods`.

## 5. Installer Dify dans le namespace `deepseek`

Ajoutez le repo Helm de Dify et installez-le dans le namespace `deepseek` :

```bash
oc apply -f manifest/dify.yaml
```

## 6. Exposer Dify avec une Route

Enfin, exposez le service Dify via une route OpenShift pour rendre l'application accessible :

```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: route-original-dify
  namespace: deepseek
spec:
  path: /
  to:
    kind: Service
    name: dify-nginx
    weight: 100
  port:
    targetPort: http-dify
  tls:
    termination: edge
  wildcardPolicy: None
```

Appliquez cette route avec la commande suivante :

```bash
oc apply -f dify-route.yaml
```

## Conclusion

Vous avez maintenant déployé DeepSeek sur OpenShift. Vous pouvez accéder à l'application via la route exposée et utiliser les différents services qu'elle offre.
```

Cette documentation couvre toutes les étapes pour déployer et configurer DeepSeek sur OpenShift. Si vous avez des ajustements à faire, n'hésitez pas à le mentionner !