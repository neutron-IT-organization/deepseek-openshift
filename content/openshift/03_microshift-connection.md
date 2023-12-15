# Connexion sur MicroShift 

## Accéder localement à MicroShift

Copiez le fichier kubeconfig d'accès local généré dans le répertoire ~/.kube/ en exécutant la commande
commande suivante :

```shell
mkdir -p ~/.kube/
sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
```

## Vérification


Vous devriez maintenant avoir accès à l'API.
Pour vérifier cela :

```shell
oc get no
```

```
[feven@scw-friendly-austin ~]$ oc get no
NAME                  STATUS   ROLES                         AGE   VERSION
scw-friendly-austin   Ready    control-plane,master,worker   32m   v1.27.6
```

## Accéder à MicroShift en remote

Pour cela nous allons devoir utiliser un nouvea kubconfig. 

Ouvrez/Creez le fichier suivant
```shell
vi /etc/microshift/config.yaml
```

Et ajouter la configuration ci-dessous en remplacent l'address IP, par l'address IP public de la machine

```shell
dns:
  baseDomain: neutron-it.fr
node:
  hostnameOverride: "scw-friendly-austin"
  nodeIP: 10.75.52.217
apiServer:
  subjectAltNames:
  - YOURPUBLICIPADDRESS
```

Puis restart

```
systemctl restart microshift
```

Copiez le fichier kubeconfig d'accès remote généré dans le répertoire 

```shell
/var/lib/microshift/resources/kubeadmin/YOURPUBLICIPADDRESS/kubeconfig 
```


