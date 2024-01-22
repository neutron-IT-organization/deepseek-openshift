# Installation de conjur cyberark sur openshift

# Prerequisites 

Installez le conjur cli.
Avoir metal Lb ou un autres service de loadbalancing disponible. 
NOTE: Il est surement possible de la faire via une route ,mais cela n'est pas traité dans ce guide.

## Installation de cyberark

Pour commencer nous allons installer conjur sur Openshift dans un namespace dédié.

```shell
oc new-project florian-ns
```

Pour cela un Helm chart est mis à disposition. Cloner le repo ci-dessous.

```shell
git clone TOCOMPLETE
```
Puis installez conjur 

```shell
DATA_KEY=$(docker run --rm cyberark/conjur data-key generate)
helm install conjur ./conjur-oss --set dataKey="${DATA_KEY}" --set authenticators="authn\,authn-k8s/florian-ns" --version 2.0.0 --set openshift.enabled=true --namespace florian-ns
#Exemple 
#helm install conjur ./conjur-oss --set dataKey="8QYoky5p9szd9iXxSRmoOqGClOebfBPNCEpBl3Bn7jo=" --set authenticators="authn\,authn-k8s/conjur" --version 2.0.0 --set openshift.enabled=true --namespace florian-ns
```

On configure ici 2 authenticators : ```authn``` pour une authentification en login/password standard et ```authn-k8s``` pour l'authentification des pods.

On créer maintenant un compte par défaut :

```shell
oc get po # On récupere ici le nom du pods
oc exec -it conjur-conjur-oss-6466669767-zbk6n -c conjur-oss -- conjurctl account create default
```

**NOTE: Conserver l'ouput de la commande précédente. Le mot de passe sera utilisé dans la suite de ce guide.**

Dans notre helm chart le certificat est configuré pour le nom de domaine conjur.myorg.com. On va donc éditer le local base domain.

```shell
oc get svc conjur-conjur-oss-ingress -ojsonpath='{.status.loadBalancer.ingress[0].ip}'
#10.253.234.69
```
```shell
sudo vi /etc/hosts
# add conjur.myorg.com 10.253.234.69
```

On va ensuite utiliser la commande ```conjur init``` pour créer un fichier de configuration (.conjurrc) qui contient les détails de connexion à Conjur. Ce fichier se trouvera alors  sous le répertoire racine de l'utilisateur.

```shell
conjur init --url=https://conjur.myorg.com --account=default -s

#The Conjur server's certificate SHA-1 fingerprint is:
#XX:XX:XX

#To verify this certificate, we recommend running the following command on the Conjur server:
#openssl x509 -fingerprint -noout -in ~conjur/etc/ssl/conjur.pem

#Trust this certificate? yes/no (Default: no): yes
#File /home/youruser/conjur-server.pem exists. Overwrite? yes/no (Default: yes): yes
#Certificate written to /home/feven/conjur-server.pem

#File /home/youruser/.conjurrc exists. Overwrite? yes/no (Default: yes): yes
#Configuration written to /home/youruser/.conjurrc

#Successfully initialized the Conjur CLI
#To start using the Conjur CLI, log in to the Conjur server by running `conjur login`
```

Vous pouvez maintenant vous connectez au serveur conjur. Le mot de passe est l'API key for admin généré lors des étapes précédentes.

```shell
conjur login
#Enter your username: admin
#Enter your password or API key (this will not be echoed):
#WARNING:root:No supported keystore found! Saving credentials in plaintext in '/home/feven/.netrc'. #Make sure to logoff after you have finished using the CLI
#Successfully logged in to Conjur
```

## Gestion des token pour Openshift

Pour automatiser la gestion des token conjur plusieurs méthodes sont possibles. Dans ce guide nous allons ajouter à notre pod un container qui s'authentifiera au serveur conjur à l'aide du certificat généré avec la commande ```conjur init```. Ce token sera ensuite placé dans un dossier partager avec le container applicatif.

Pour cela nous allons tout d'abord créer une configmap contenant le certificat.

```shell
oc create cm conjur-cert --from-file=ssl-certificate=/home/feven/conjur-server.pem
```

Nous allons maintenant créer une policy dans cyberark. Permettant au container **TODO Partie à developper**. Explique ce que fait la policy.

```shell
vi policy-autom.yaml
```

```shell
- !policy
  id: conjur/authn-k8s/florian-ns
  body:
  - !webservice
  - !layer
    id: ubi8-app
  - !host
    id: florian-ns/service_account/florian-sa
    annotations:
      authn-k8s/namespace: florian-ns
      authn-k8s/authentication-container-name: ubuntu-auth
  - !grant
    role: !layer ubi8-app
    member: !host florian-ns/service_account/florian-sa
  - !permit
    resource: !webservice
    privilege: [ authenticate ]
    role: !layer ubi8-app
# CA cert and key for creating client certificates
  - !policy
    id: ca
    body:
    - !variable
      id: cert
      annotations:
        description: CA cert for Kubernetes Pods.
    - !variable
      id: key
      annotations:
        description: CA key for Kubernetes Pods.
  - !variable test
  - !permit
    role: !layer ubi8-app
    privileges: [ read, execute ]
    resource: !variable test
```

```shell
conjur policy load -b root -f policy.yaml
```

On peut maintenant ajouter une variable de test :

```shell
 conjur variable set -i conjur/authn-k8s/florian-ns/test -v 1234567890
```

Ajouter maintenant les variable de clef/certificat.

```shell
conjur variable set -i conjur/authn-k8s/conjur/ca/cert -v "$(cat /home/feven/cyberark/ca.key)" 
conjur variable set -i conjur/authn-k8s/conjur/ca/cert -v "$(cat /home/feven/conjur-server.pem)" 
```

Nous pouvons maintenant déployer notre pod constitué d'un pod applicatif et du conjur authenticator.

```shell
vi ubi8.yaml
```

```shell
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ubi8
  namespace: florian-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ubi8
  template:
    metadata:
      labels:
        app: ubi8
    spec:
      containers:
      - name: ubi8
        image: openshift.artifactory.mycloud.intranatixis.com/ubi8
        command: ["/bin/bash", "-c", "tail -f /dev/null"]
        env:
        - name: CONJUR_APPLIANCE_URL
          value: "https://conjur-conjur-oss.florian-ns.svc.cluster.local/authn-k8s/florian-ns"
        - name: CONJUR_ACCOUNT
          value: default
        - name: CONJUR_AUTHN_TOKEN_FILE
          value: /run/conjur/access-token
        - name: CONJUR_SSL_CERTIFICATE
          valueFrom:
            configMapKeyRef:
              name: conjur-cert
              key: ssl-certificate
        volumeMounts:
        - mountPath: /run/conjur
          name: conjur-access-token
          readOnly: true
      - name: ubuntu-auth
        image: openshift.artifactory.mycloud.intranatixis.com/cyberark/conjur-openshift-authenticator
        env:
        - name: MY_POD_NAME
          valueFrom:
           fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
           fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
             fieldPath: status.podIP
        - name: CONJUR_AUTHN_URL
          value: "https://conjur-conjur-oss.florian-ns.svc.cluster.local/authn-k8s/florian-ns"
        - name: CONJUR_ACCOUNT
          value: default
        - name: CONJUR_AUTHN_LOGIN
          value: "host/conjur/authn-k8s/florian-ns/florian-ns/service_account/florian-sa"
        - name: CONJUR_SSL_CERTIFICATE
          valueFrom:
            configMapKeyRef:
              name: conjur-cert
              key: ssl-certificate
        volumeMounts:
        - mountPath: /run/conjur
          name: conjur-access-token
      serviceAccountName: florian-sa
      volumes:
      - name: conjur-access-token
        emptyDir:
          medium: Memory
```

## Validation

Nous pouvons maintenant tester notre serveur conjur.

Pour cela connectez vous au pod ubi8

```shell
oc rsh ubi8-7967957478-tc96s

CONT_SESSION_TOKEN=$(cat /run/conjur/access-token| base64 | tr -d '\r\n')
curl -s -k -H "Content-Type: application/json" -H "Authorization: Token token=\"$CONT_SESSION_TOKEN\"" https://conjur-conjur-oss-ingress.florian-ns.svc.cluster.local/secrets/default/variable/conjur%2Fauthn-k8s%2Fflorian-ns%2Ftest
1234567890
```





