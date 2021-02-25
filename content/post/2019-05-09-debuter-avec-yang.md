---
title: Débuter avec YANG
author: Florian Lacommare
date: '2019-05-09'
slug: debuter-avec-yang
categories:
  - Automatisation
tags:
  - automatisation
  - yang
image:
  caption: ''
  focal_point: ''
---

Suite à mon <a href="/post/l-automatisation-reseau-multi-vendeurs-pari-tenu/" target="_blank">précédent article</a>
 sur l'introduction de l'automatisation multi-vendeurs et à l'annonce de Jason Edelman la <a href="https://www.networktocode.com/blog/post/open-source-model-driven-network-programmability/" target="_blank">semaine dernière</a> sur un nouveau projet <a href="https://github.com/networktocode/yangify" target="_blank">Yangify</a>, il est intéressant de faire une introduction au langage YANG. Celui-ci est inspiré de l'article de <a href="https://napalm-automation.net/yang-for-dummies/" target="_blank">David Barroso</a>.

&nbsp;

## YANG, c'est quoi ?

YANG (Yet Another Next Generation) est un langage de modélisation de données qui permet de manipuler des états et configurations réseau. YANG est défini par de multiples RFC, <a href="https://tools.ietf.org/html/rfc6020" target="_blank">RFC 6020</a> et la <a href="https://tools.ietf.org/html/rfc7950" target="_blank">RFC 7950</a> pour la version 1.1.  
Il ne faut pas confondre YANG, un langage, avec JSON ou XML, qui sont des formats de données. Et par ailleurs, comme on le verra dans l'article, YANG utilise nativement XML comme format et le support du JSON est ajouté par la <a href="https://tools.ietf.org/html/rfc7951" target="_blank">RFC 7951</a>.

&nbsp;


## Terminologie

Je vais tenter d'expliquer simplement les termes YANG que je vais utiliser dans cet article, cependant sachez qu'il en existe d'autres, que vous trouverez dans la <a href="https://tools.ietf.org/html/rfc6020#section-3" target="_blank">RFC 6020 section 3</a> :

* **module** : Un module YANG défini une hiérarchie de noeuds, ou autrement dit, un ensemble hiérarchisé de données, qui peuvent être utilisés par NETCONF (protocole de communication avec les équipements réseaux).

* **leaf** : C'est un noeud qui contiendra une valeur lorsque le module sera instancié.

* **list** : C'est un noeud qui comporte un ensemble de noeuds enfants et qui peut être instancié plusieurs fois. Plus simplement, c'est une liste comme dans tout type de langage. Chaque entrée de la liste doit avoir un identifiant unique.

* **grouping** : Cela défini un ensemble de noeuds qui peut être utilisé dans un module (qu'il soit local, ou dans un des modules qui l'inclu ou l'importe).

* **identity** : Permet de définir un type abstrait et unique.

* **container** : Ne contient pas de valeur mais un ensemble de noeuds.

* **augment** : Ajoute un nouveau schéma, une hiérachie, à un module déjà existant.

&nbsp;

## Cas d'usage

### Création d'un modèle

Afin de comprendre le fonctionnement de YANG, prenons l'exemple d'un inventaire de voiture à vendre.  
Pour commencer, nous allons créer notre structure permettant d'ajouter des voitures sous forme de liste.

Les différentes informations dont nous avons besoins sont : 

* **name**, qui sera le nom du vendeur. Son type est une chaîne de caractères

* **inService**, l'année de mise en service, que l'on souhaite démarrer à 2009, année de mise en place du nouveau système d'immatriculation, ainsi, voici le type de valeur attendu : 

```plaintext
typedef yearInService {
  type uint16 {
    range 2009..2999;
  }
}
```

* **numberplate**, la plaque d'immatriculation utilisant le système SIV (ex: AB-120-CD) :

```plaintext
typedef numberplate {
  type string {
    pattern '[a-zA-Z]{2}-[0-9]{3}-[a-zA-Z]{2}';
  }
}
```

* **brand**, qui répresente la marque du véhicule. Pour la représenter, nous allons créer une `identity` pour identifier chaque possibilité :

```plaintext
identity BRAND {
  description "Describe the brand of the vehicule";
}

identity RENAULT {
  base BRAND;
  description "Vehicule manufactured by Renault";
}

identity PEUGEOT {
  base BRAND;
  description "Vehicule manufactured by Peugeot";
}
```

Maintenant que le principal est fait, il ne reste plus qu'à créer le module :
```plaintext
// module name
module sell-vehicule {

    yang-version "1";
    namespace "http://flolaco.ovh/post/debuter-avec-yang/";

    prefix "sell-vehicule";

    // identity to identify the brand of the vehicule
    identity BRAND {
        description "Describe the brand of the vehicule";
    }

    identity RENAULT {
        base BRAND;
        description "Vehicule manufactured by Renault";
    }

    identity PEUGEOT {
        base BRAND;
        description "Vehicule manufactured by Peugeot";
    }

    // new type to enforce the date
    typedef yearInService {
      type uint16 {
        range 2009..2999;
      }
    }
    // new type to enforce the numberplate
    typedef numberplate {
      type string {
        pattern '[a-zA-Z]{2}-[0-9]{3}-[a-zA-Z]{2}';
      }
    }

    // this grouping gather all data assigned to vehicules
    grouping vehicule-data {
        leaf name {
            type string;
        }
        leaf inService {
            type yearInService;
        }
        leaf numberplate {
            type numberplate;
        }
        leaf brand {
            type identityref {
                base sell-vehicule:BRAND;
            }
        }
    }

    // this is the root object defined by the model
    container stock {
        list vehicule {
            // identify each vehicule by using it's numberplate
            key "numberplate";

            // each vehicule have the elements defined in the grouping
            uses vehicule-data;
        }
    }
}
```

Dans le module, nous avons créé un `grouping` qui permet de regrouper toutes les données de chaque véhicule puis en enfin un `container` afin de contenir notre `list` de véhicules. Notez que la `key` de chaque élément de la liste (les véhicules) est la plaque d'immatriculation.

&nbsp;

### Utilisation du modèle

Pour la suite de l'article, nous allons utiliser l'outil `pyang` qui nous permet de représenter les modèles YANG sous une forme d'arbre. Nous allons également utiliser `pyangbind` qui est un plugin de `pyang` permettant de générer un code Python capable de fonctionner avec un modèle YANG particulier.

L'installation de cet outil est très simple : 
`pip install pyang pyangbind`

Pour représenter notre modèle sous la forme d'un arbre :
```shell
$ pyang -f tree sell-vehicule.yang
module: sell-vehicule
  +--rw stock
     +--rw vehicule* [numberplate]
        +--rw name?          string
        +--rw inService?     yearInService
        +--rw numberplate    numberplate
        +--rw brand?         identityref
```

Notez que `numberplate` ne contient pas de `?` car il représente la clé de la liste.  
Il est temps de générer notre code Python pour tester notre modèle YANG grâce au plugin `pyangbind` : 
```shell
$ export PYBINDPLUGIN=`/usr/bin/env python -c \
        'import pyangbind; \
        import os; print "%s/plugin" % os.path.dirname(pyangbind.__file__)'`
$ pyang --plugindir $PYBINDPLUGIN -f pybind sell-vehicule.yang > sell_vehicule.py
```

Maintenant que notre code Python a été généré on peut le tester :

```python
>>> import sell_vehicule
>>>
>>> v = sell_vehicule.sell_vehicule()
>>>
>>> flo = v.stock.vehicule.add("AA-101-ZZ")
>>> flo.inService = 2009
>>> flo.name = "Flo"
>>> flo.brand = "RENAULT"
>>>
>>> laco = v.stock.vehicule.add("AB-987-CD")
>>> laco.inService = 2011
>>> laco.name = "Lacommare"
>>> laco.brand = "PEUGEOT"
>>>
>>> jeff = v.stock.vehicule.add("FL-999-LF")
>>> jeff.inService = 2031
>>> jeff.name = "Jeff"
>>> jeff.brand = "RENAULT"
>>>
>>>
>>> import json
>>> print(json.dumps(v.get(), indent=4))
{
    "stock": {
        "vehicule": {
            "AA-101-ZZ": {
                "brand": "RENAULT",
                "name": "Flo",
                "numberplate": "AA-101-ZZ",
                "inService": 2009
            },
            "AB-987-CD": {
                "brand": "PEUGEOT",
                "name": "Lacommare",
                "numberplate": "AB-987-CD",
                "inService": 2011
            },
            "FL-999-LF": {
                "brand": "RENAULT",
                "name": "Jeff",
                "numberplate": "FL-999-LF",
                "inService": 2031
            }
        }
    }
}
```

Testons maintenant la création d'une entrée avec le format d'une plaque d'immatriculation incorrect :
```python
>>> willfail = v.stock.vehicule.add("KAFE-9-KAFE")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python2.7/site-packages/pyangbind/lib/yangtypes.py", line 760, in add
    self.__set(_k=k)
  File "/usr/local/lib/python2.7/site-packages/pyangbind/lib/yangtypes.py", line 679, in __set
    raise KeyError("key value must be valid, %s" % m)
KeyError: u'key value must be valid, {\'error-string\': \'numberplate must be of a type compatible with numberplate\', \'generated-type\': \'YANGDynClass(base=RestrictedClassType(base_type=six.text_type, restriction_dict={u\\\'pattern\\\': u\\\'[a-zA-Z]{2}-[0-9]{3}-[a-zA-Z]{2}\\\'}), is_leaf=True, yang_name="numberplate", parent=self, path_helper=self._path_helper, extmethods=self._extmethods, register_paths=True, is_keyval=True, namespace=\\\'http://flolaco.ovh/post/debuter-avec-yang/\\\', defining_module=\\\'sell-vehicule\\\', yang_type=\\\'numberplate\\\', is_config=True)\', \'defined-type\': \'sell-vehicule:numberplate\'}'
```

Nous avons une erreur, normal car le format est incorrect.  
Et maintenant imaginons qu'on souhaite ajouter une voiture Dacia :
```python
>>> dacia = v.stock.vehicule.add("DA-030-IA")
>>> dacia.inService = 2019
>>> dacia.name = "John Doe"
>>> dacia.brand = "DACIA"
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "sell_vehicule.py", line 199, in _set_brand
    'generated-type': """YANGDynClass(base=RestrictedClassType(base_type=six.text_type, restriction_type="dict_key", restriction_arg={u'sell-vehicule:RENAULT': {u'@namespace': u'http://flolaco.ovh/post/debuter-avec-yang/', u'@module': u'sell-vehicule'}, u'sell-vehicule:PEUGEOT': {u'@namespace': u'http://flolaco.ovh/post/debuter-avec-yang/', u'@module': u'sell-vehicule'}, u'PEUGEOT': {u'@namespace': u'http://flolaco.ovh/post/debuter-avec-yang/', u'@module': u'sell-vehicule'}, u'RENAULT': {u'@namespace': u'http://flolaco.ovh/post/debuter-avec-yang/', u'@module': u'sell-vehicule'}},), is_leaf=True, yang_name="brand", parent=self, path_helper=self._path_helper, extmethods=self._extmethods, register_paths=True, namespace='http://flolaco.ovh/post/debuter-avec-yang/', defining_module='sell-vehicule', yang_type='identityref', is_config=True)""",
ValueError: {'error-string': 'brand must be of a type compatible with identityref', 'generated-type': 'YANGDynClass(base=RestrictedClassType(base_type=six.text_type, restriction_type="dict_key", restriction_arg={u\'sell-vehicule:RENAULT\': {u\'@namespace\': u\'http://flolaco.ovh/post/debuter-avec-yang/\', u\'@module\': u\'sell-vehicule\'}, u\'sell-vehicule:PEUGEOT\': {u\'@namespace\': u\'http://flolaco.ovh/post/debuter-avec-yang/\', u\'@module\': u\'sell-vehicule\'}, u\'PEUGEOT\': {u\'@namespace\': u\'http://flolaco.ovh/post/debuter-avec-yang/\', u\'@module\': u\'sell-vehicule\'}, u\'RENAULT\': {u\'@namespace\': u\'http://flolaco.ovh/post/debuter-avec-yang/\', u\'@module\': u\'sell-vehicule\'}},), is_leaf=True, yang_name="brand", parent=self, path_helper=self._path_helper, extmethods=self._extmethods, register_paths=True, namespace=\'http://flolaco.ovh/post/debuter-avec-yang/\', defining_module=\'sell-vehicule\', yang_type=\'identityref\', is_config=True)', 'defined-type': 'sell-vehicule:identityref'}
```

&nbsp;

## Étendre le modèle

Comme nous venons de le voir, il est impossible d'ajouter une voiture d'une nouvelle marque si notre modèle YANG ne le supporte pas. Et si on voulait ajouter une information supplémentaire : Neuve ou d'occassion ? 

C'est tout à fait possible, YANG est puissant pour rajouter des compléments dans un modèle existant. En effet, nous ne sommes pas obligé de modifier le modèle original, ni de faire un fork de celui-ci.  
Nous pouvons simplement importer l'ancien et ajouter nos nouveautés comme suit :

```plaintext
module sell-vehicule-ext {

    yang-version "1";
    namespace "http://flolaco.ovh/post/debuter-avec-yang/ext";

    prefix "sell-vehicule-ext";

    // We import the old model
    import sell-vehicule { prefix sv; }

    // New manufacture based on the old BRAND
    identity DACIA {
        base sv:BRAND;
        description "Vehicule manufactured by Renault-Dacia";
    }

    // We are adding more information to the vehicule
    // data of the old model
    grouping ext-vehicule-data {
        leaf category {
            type enumeration {
                enum NEW {
                    description "New vehicule, never used";
                }
                enum USED {
                    description "Vehicule already used";
                }
            }
            default NEW;
        }
    }

    // We are actually adding new stuff in the old model
    augment "/sv:stock/sv:vehicule" {
        uses ext-vehicule-data;
    }
}
```

La force d'utiliser ce système d'extension est que si le modèle original subit des changements, notre modèle va également en bénéficier.  
Maintenant, affichons notre modèle sous forme d'arbre et essayons de créer nos données :

```shell
$ pyang -f tree sell-vehicule-ext.yang sell-vehicule.yang
module: sell-vehicule
  +--rw stock
     +--rw vehicule* [numberplate]
        +--rw name?                         string
        +--rw inService?                    yearInService
        +--rw numberplate                   numberplate
        +--rw brand?                        identityref
        +--rw sell-vehicule-ext:category?   enumeration
        
$ pyang --plugindir $PYBINDPLUGIN -f pybind sell-vehicule-ext.yang sell-vehicule.yang > sell_vehicule_ext.py
```

```python
>>> import sell_vehicule_ext
>>>
>>> v = sell_vehicule_ext.sell_vehicule()
>>>
>>> flo = v.stock.vehicule.add("AA-101-ZZ")
>>> flo.inService = 2009
>>> flo.name = "Flo"
>>> flo.brand = "RENAULT"
>>>
>>> laco = v.stock.vehicule.add("AB-987-CD")
>>> laco.inService = 2011
>>> laco.name = "Lacommare"
>>> laco.brand = "PEUGEOT"
>>>
>>> jeff = v.stock.vehicule.add("FL-999-LF")
>>> jeff.inService = 2031
>>> jeff.name = "Jeff"
>>> jeff.brand = "RENAULT"
>>>
>>> dacia = v.stock.vehicule.add("DA-030-IA")
>>> dacia.inService = 2019
>>> dacia.name = "John Doe"
>>> dacia.brand = "DACIA"
>>> dacia.category = "USED"
>>>
>>> import json
>>> print(json.dumps(v.get(), indent=4))
{
    "stock": {
        "vehicule": {
            "AA-101-ZZ": {
                "category": "NEW",
                "brand": "RENAULT",
                "name": "Flo",
                "numberplate": "AA-101-ZZ",
                "inService": 2009
            },
            "AB-987-CD": {
                "category": "NEW",
                "brand": "PEUGEOT",
                "name": "Lacommare",
                "numberplate": "AB-987-CD",
                "inService": 2011
            },
            "FL-999-LF": {
                "category": "NEW",
                "brand": "RENAULT",
                "name": "Jeff",
                "numberplate": "FL-999-LF",
                "inService": 2031
            },
            "DA-030-IA": {
                "category": "USED",
                "brand": "DACIA",
                "name": "John Doe",
                "numberplate": "DA-030-IA",
                "inService": 2019
            }
        }
    }
}
```

&nbsp;

## Exemple appliqué au réseau

Maintenant que l'on a fait une brève introduction du langage YANG, il est intéressant de voir l'usage qui en est fait par `napalm-yang`. OpenConfig, qui est un groupe de travail pour la création de modèle YANG multi-vendeur, a créé (entre autre) un modèle pour définir les interfaces réseaux des équipements.  

Cependant, ils n'ont pas pris en compte le support des IP `secondary` de certains constructeurs. `napalm-yang` a donc créé un modèle qui étend celui d'<a href="https://github.com/openconfig/public/blob/master/release/models/interfaces/openconfig-if-ip.yang" target="_blank">openconfig-if-ip</a> :

```plaintext
module napalm-if-ip {
    yang-version "1";

    namespace "https://github.com/napalm-automation/napalm-yang/yang_napalm/interfaces";

    prefix "napalm-ip";

    import openconfig-interfaces { prefix oc-if; }
    import openconfig-vlan { prefix oc-vlan; }
    import openconfig-if-ip { prefix oc-ip; }

    organization "NAPALM Automation";

    contact "napalm-automation@googlegroups.com";

    description "This module defines some augmentations to the interface's IP model of OC";

    revision "2017-03-17" {
      description
        "First release";
      reference "1.0.0";
    }

    grouping secondary-top {
        description "Add secondary statement";

        leaf secondary {
            type boolean;
            default "false";

            description
            "Most platforms need a secondary statement on when configuring multiple IPv4
            addresses on the same interfaces";

            reference "https://www.cisco.com/c/en/us/td/docs/ios/12_2/ip/configuration/guide/fipr_c/1cfipadr.html#wp1001012";
        }
    }

    augment "/oc-if:interfaces/oc-if:interface/" +
        "oc-if:subinterfaces/oc-if:subinterface/" +
        "oc-ip:ipv4/oc-ip:addresses/oc-ip:address/" +
        "oc-ip:config" {
        description "Add secondary statement to subinterfaces' IPs";

        uses secondary-top;
    }

    augment "/oc-if:interfaces/oc-if:interface/" +
        "oc-vlan:routed-vlan/" +
        "oc-ip:ipv4/oc-ip:addresses/oc-ip:address/" +
        "oc-ip:config" {
        description "Add secondary statement to routed VLANs' IPs";

        uses secondary-top;
    }
}
```
&nbsp;

Maintenant que YANG n'a plus de secret pour vous, je vous conseil d'aller jeter un oeil au nouveau projet <a href="https://github.com/networktocode/yangify" target="_blank">Yangify</a> qui s'appuie justement sur des modèles YANG pour générer des configurations et vice-versa et qui fera sans doute l'objet d'un prochain article.
