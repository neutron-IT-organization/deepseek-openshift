# Connexion a la console 

## Configure dynamic storage


MicroShift permet un provisionnement de stockage dynamique et prêt à l'emploi avec LVMS. Le plugin LVMS est la version Red Hat de TopoLVM, un plugin CSI permettant de gérer les volumes LVM pour Kubernetes.

## Ajouter du stockage

Dans un premier temps ajoutez un disque de stockage dans scaleway. Pour cela cliquez sur votre instance microshift puis sur Attached volumes et enfin sur Create volume. Selectionnez une taille (en GB) et cliquez sur Add Volume.

![Create Disk](../images/create-disk.png)


vous devrez ensuite identifier les disques que vous souhaitez utiliser pour le VG. 

```shell
sudo pvcreate /dev/sdb
```

Une fois le volume physique préparé, vous pouvez créer un groupe de volumes nommé « microshift » et y ajouter le(s) volume(s) physique(s) initialisé(s) :


```shell
sudo vgcreate microshift /dev/sdb
```

Redemarrez ensuite microshift 

```shell
systemctl restart microshift
```

## Vérification

Vous devriez maintenant avoir un nouveau namespace "openshift-storage" dans la console. Avec un pod topolvm-controller et un pod topolvm-node Running.

![Create Disk](../images/topolvm.png)
