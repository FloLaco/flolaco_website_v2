---
highlight_languages: ["yaml"]
title: L'automatisation réseau multi-vendeurs, pari tenu ?
author: Florian Lacommare
date: '2018-10-02'
slug: l-automatisation-reseau-multi-vendeurs-pari-tenu
categories:
  - Automatisation
tags: [automatisation]
image:
  caption: ''
  focal_point: ''
summary: 'L’automatisation au service des entreprises.

Cela fait maintenant plusieurs années que les entreprises cherchent à gagner en agilité, en rapidité de déploiement pour un « Time To Market » de plus en plus court ou pour minimiser les risques opérationnels. Une des solutions pour répondre à ces enjeux est l’automatisation de son SI.'
links:
  - icon_pack: fab
    icon: linkedin
    name: Join my network
    url: 'https://www.linkedin.com/in/florian-lacommare/'
share: true  # Show social sharing links?

---

### L’automatisation au service des entreprises

Cela fait maintenant plusieurs années que les entreprises cherchent à gagner en agilité, en rapidité de déploiement pour un « Time To Market » de plus en plus court ou pour minimiser les risques opérationnels. Une des solutions pour répondre à ces enjeux est l’automatisation de son SI.

L’automatisation a démarré il y a maintenant plusieurs années sur le domaine du système et de la virtualisation. Là où une équipe mettait plusieurs semaines pour provisionner un serveur, elle arrive à déployer une VM en quelques secondes. Peut-on en dire autant pour le réseau ?

&nbsp;

### L’automatisation système est mature. Et maintenant, place à l’automatisation réseau
Peut-on automatiser son réseau ? La réponse est bien évidemment oui, mais la vraie question à se poser est comment le faire ?

La première réponse possible est d’intégrer un système d’automatisation adapté au fournisseur déjà en place dans la production. Une des principales difficultés avec cette approche apparaît lorsqu’on souhaite faire rentrer un nouveau fournisseur dans le parc déjà existant ou tout simplement quand on ajoute des équipements de gammes différentes. Dans ces cas-là, il faut soit multiplier les solutions d’automatisation (une par fournisseur, et rarement compatible), soit tout reconstruire pour l’adapter aux nouveaux équipements.

La deuxième possibilité est d’intégrer dès le début une solution d’automatisation multi-vendeurs permettant de minimiser les adaptations / développements à réaliser en fonction des nouveaux fournisseurs. Pour répondre à cette problématique, il est possible soit de retenir une solution SDN multi-vendeurs, soit de bâtir une solution sur un framework comme NAPALM.

&nbsp;

### NAPALM, framework d’automatisation multi-vendeurs
NAPALM est un framework open source agnostique, écrit en Python, qui utilise de façon unifiée les API des différents fournisseurs. Actuellement, ce framework supporte IOS, IOS-XR, NX-OS de Cisco, EOS d’Arista et JunOS de Juniper.


{{< figure src="/post/2019-05-06-l-automatisation-réseau-multi-vendeurs-pari-tenu_files/compatibility-matrix.png" caption="Matrice de compatibilité NAPALM (source : https://napalm.readthedocs.io/en/latest/support/index.html)" >}}

&nbsp;

### Un framework aux multiples interactions
Ce framework regroupe plusieurs projets, dont un qui nous intéresse plus particulièrement : la bibliothèque NAPALM, du même nom que le framework. Cette bibliothèque permet de se connecter aux équipements réseaux et d’interagir avec eux par le biais de « getters » et de fonctions permettant la configuration, le tout via le développement de scripts en Python.



```python
# NAPALM code in Python

from napalm import get_network_driver

driver = get_network_driver("nxos")
device = driver("IP", "my_login", "my_passwd")
device.open()
mac_table = device.get_mac_address_table()
device.close()
```


[//]: <> ({{< highlight python "linenos=inline" >}})
[//]: <> (# NAPALM code)
[//]: <> ()
[//]: <> (from napalm import get_network_driver)
[//]: <> ()
[//]: <> (driver = get_network_driver(driver))
[//]: <> (device = driver("IP", "my_login", "my_passwd"))
[//]: <> (device.open())
[//]: <> (mac_table = device.get_mac_address_table())
[//]: <> (device.close())
[//]: <> ({{< / highlight >}})


[//]: <> ({{< figure src="/post/2019-05-06-l-automatisation-réseau-multi-vendeurs-pari-tenu_files/napalm-code.png" caption="Code Python avec NAPALM" >}})


Le projet « napalm-ansible » permet d’utiliser ce framework d’une autre façon, directement au sein de l’outil Ansible, très répandu dans le domaine de l’automatisation système. Ce modèle, malgré l’utilisation d’Ansible, utilisera toujours le back-end de NAPALM, ainsi, l’installation de la bibliothèque Python reste nécessaire.


```yaml
---
# Utilisation de NAPALM dans un playbook Ansible
- hosts: cisco
  connection: no
  gather_facts: no
  tasks:
    - name: Gather Facts
      napalm_get_facts:
        hostname: "{{ ansible_host }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        dev_os: "nxos"
      register: result

    - name: Print Facts
      debug: msg="{{ result }}"

```

[//]: <> ({{< figure src="/post/2019-05-06-l-automatisation-réseau-multi-vendeurs-pari-tenu_files/napalm-with-ansible.png" caption="Utilisation de NAPALM dans un playbook Ansible" >}})
 
&nbsp;
 
### Des uses cases qui fonctionnent ….
Maintenant que l’utilisation du framework NAPALM n’a plus de secrets pour automatiser nos équipements, nous allons nous intéresser à différents use case. Si par exemple nous souhaitons connaitre les serveurs NTP de nos équipements, il est très facile de trouver cette information grâce à la méthode « get_ntp_servers() ».

Si maintenant nous désirons en savoir plus sur les voisins BGP qui sont en peering avec nos équipements ainsi que leur statut, nous pouvons récupérer ces éléments grâce à la méthode « get_bgp_neighbors ».

&nbsp;

### … et d’autres qui ne fonctionnent pas

En revanche, si nous souhaitons disposer de ces mêmes informations pour le protocole OSPF, alors nous constatons qu’il n’est pas possible de le faire car aujourd’hui il n’existe aucune implémentation d’une telle méthode dans le framework. De même, si nos équipements font partie d’une fabrique VXLAN, il n’existe aujourd’hui aucun support pour récupérer les informations des VNI, les informations MP-BGP EVPN ou autres.

Pour répondre à ces problématiques, il existe différentes alternatives. La première est de développer par soi-même ces méthodes. Par exemple, vous pouvez trouver sur mon propre repository GitHub (https://github.com/FloLaco/napalm) l’ajout de plusieurs fonctionnalités (OSPF et VXLAN). Une autre possibilité, si nous utilisons Ansible et NAPALM, est de tirer profit des modules Ansible des différents équipementiers. Malheureusement, il y a plusieurs inconvénients à cela. Le premier est la perte du bénéfice initial de l’approche multi-vendeurs, et le deuxième est que, même avec ces modules, tout n’est pas disponible. Une autre alternative serait alors de changer d’approche et d’utiliser des modèles YANG comme par exemple OpenConfig.


&nbsp;

### NAPALM, oui ! mais pas que …
Au final, l’automatisation multi-vendeurs par le biais du framework NAPALM est possible et recommandée sous certaines conditions. Il est donc impératif de bien étudier les besoins d’automatisation initiaux pour s’assurer que tous les besoins sont bien couverts, et évaluer le cas échéant les développements complémentaires à réaliser. Cependant, il existe d’autres alternatives comme OpenConfig, qui fera certainement l’objet d’un autre article au regard de sa popularité grandissante.

