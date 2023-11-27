# Installation de MicroShift (mode RPM)

## Procédure d'installation

### Activation des dépôts Red Hat de MicroShift
En tant qu'utilisateur root, exécutez la commande suivante pour activer les dépôts Red Hat de MicroShift :

```shell
sudo subscription-manager repos \
--enable rhocp-4.14-for-rhel-9-$(uname -i)-rpms \
--enable fast-datapath-for-rhel-9-$(uname -i)-rpms
```
Puis installez microshift

```shell
sudo dnf install -y microshift
```

###  Copie du pull secret vers /etc/crio

Téléchargez votre pull secret d'installation depuis la Console Cloud Hybride Red Hat (https://console.redhat.com/openshift/install/pull-secret) vers un dossier temporaire, par exemple, $HOME/openshift-pull-secret. 

Pour copier le pull secret dans le dossier /etc/crio de votre machine RHEL, exécutez :

```shell
sudo cp $HOME/openshift-pull-secret /etc/crio/openshift-pull-secret
```

Rendre le fichier /etc/crio/openshift-pull-secret lisible et inscriptible par l'utilisateur root uniquement en
en exécutant la commande suivante :

```shell
sudo chown root:root /etc/crio/openshift-pull-secret
```

## Démarrez Red Hat Device Edge.

En tant qu'utilisateur root, démarrez MicroShift:

```shell
sudo systemctl start microshift
```

Pour configurer votre machine RHEL afin qu'elle démarre MicroShift lorsque votre Machine démarre

```shell
sudo systemctl enable microshift
```