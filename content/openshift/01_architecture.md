# Architecture

L'univers de la technologie évolue à un rythme effréné, et les amateurs d'informatique sont de plus en plus nombreux à chercher des moyens d'acquérir une expérience pratique en matière de gestion de conteneurs et d'orchestration. C'est là qu'intervient OpenShift, une plateforme puissante basée sur Kubernetes qui facilite le déploiement, la gestion et la mise à l'échelle d'applications conteneurisées. Dans ce guide, nous allons explorer comment créer votre propre laboratoire domestique pour OpenShift en utilisant un Intel NUC10 comme nœud unique et un routeur GL.iNet GL-AR750-Ext (Slate) pour la connectivité réseau.

## Domain name

| Name | Description |
| ---- | ----------- |
| *.florian-lab.homelab.com | **florian-lab** is a sub-domain of **homelab.com** domain dedicated to this lab. We will use it for internal (with our DNS server) **&** extrernal (Public DNS servers) resolutions. So, depending from where the client requesting a domain resolution for *.florian-lab.homelab.com, the Private DNS servers or Public DNS servers will be use. |

### DNS Configuration :

* Inside Homelab, instances have DNS configuration with router as DNS servers where *.florian-lab.homelab.com requests forwards to private homelab IPs
* Outside Homelab, Openshift access public URL using public DNS



### DNS records :


| URL| Description | Internal Resolution |
| ---------- | ------ | ----------- |
| router.florian-lab.homelab.com | DNS record for router and DNS | 192.168.8.1 |
| api.florian-lab.homelab.com | DNS record for Management cluster API | 192.168.8.11 |
| api-int.florian-lab.homelab.com | DNS record for Management cluster API-INT | 192.168.8.1 |
| *.apps.florian-lab.homelab.com | DNS record for Management cluster Ingress | 192.168.8.11 |
| router.florian-lab.homelab.com | DNS record for Managed cluster 01 Ingress & API | 192.168.8.1 |

## Instances

For the Homelab we will use physical server or virtualized machine. 

| Instance Name | IP | Serveur | OS | Role |
| :------------- | :-: | :-: | :-: | :-: |
| sno-management |  192.168.8.11  | TBD | RHCOS | SNO of Management Cluster |
