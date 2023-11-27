# Connexion sur MicroShift 

## Accéder localement à MicroShift

Copiez le fichier kubeconfig d'accès local généré dans le répertoire ~/.kube/ en exécutant la commande
commande suivante :

```shell
mkdir -p ~/.kube/
sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
```

## Vérification

vérifiez que vous avez accès à l'API.

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


