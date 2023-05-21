# Projet routage inter-domaine

## sujet 

L’objectif principal de ce TP / mini-projet est d’inter-connecter les différents sous-sites, au moyen de VPN-BGP MPLS, avec un
espace d’adressage privé (attention de ne pas rentrer en conflit avec les autres domaines si vous voulez atteindre la dernière
question facultative) et re-utilisé pour les deux VPN. Votre domaine, son routage interne, et ses liens avec les CE sont déjà préconfigurés; en revanche les liens CE avec leur hôte sont laissés à vos soins ainsi que toute la configuration plus avancée (MPLS,
session iBGP et routage des adresses privées sur le CE – avec leur hôte ou avec des loopback locales).
Pour atteindre les configurations souhaitées vous devrez expérimenter deux pistes : en full-mesh et en hub & spoke. En option
finale, vous pourrez essayer de connecter vos hubs et leurs clients à l’Internet...

## Configuration

Avant toute configuration directe des VPN, notre premiere action était de metre en place les differentes adresse ip et de loopback manquantes sur les différentes instances. Celle-ci pourront être directement consulté sur nos config.
#
### Mise en place d'un VPN en full-mesh:
Dans cette premiere partie, nous cherchons à mettre en place le VPN X en full-mesh tout en faisant varié le type de de protocol de routage au sein des sous-réseaux composant le VPN : static, ospf et rip. L'interêt est double: montrer nos comprehension des differents protocole mais aussi assurer une certaine sécurité à notre mini internet en evitant la dépendance à un seul protocol de communication

Avant de commencer la configuration des routes static en soit, certaines étapes préliminaire doivent être faite. 

On commence dans un premier temps par activé le protocol BGP sur notre routeur PE1 tout en spécifiant notre numéro d'AS. Ensuite, la commande “address-family ipv4 vpn” est utilisée pour activer la famille d’adresses IPv4 VPN. Les commandes “neighbor X.X.X.X activate” sont utilisées pour activer les voisins BGP avec les adresses lo des routeurs d'entrée (PE3 et PE6) liées aux autre sous réseaux du VPN X. On réalise la même chose sur les routeurs PE3 et PE6 en activant les voisins BGP avec les adresses lo des routeurs liées aux autre sous réseaux du VPN X. 

```
conf t
router bgp 10
address-family ipv4 vpn
neighbor 10.155.0.1 activate
neighbor 10.158.0.1 activate
```

Dans un 2ème temps, sachant que les nœuds dans différents VPN ne peuvent pas communiquer entre eux, donc les informations de routage de différents VPN doivent être stockées dans des structures différentes. Cela est possible grâce à VRF (Virtual Routing and Forwarding), qui nous permet de stocker les informations de routage dans différentes tables (une par VPN). Pour utiliser VRF, les interfaces du routeur (dans votre AS) connectées à un bord client doivent être assignées à une VRF particulière. FRRouting s’appuie sur Linux VRF et les interfaces VRF doivent être créées dans Linux avant que nous puissions les configurer dans FRRouting. Pour les créer, nous allons sur le conteneur des routeurs et utilisons la commande ip. Notez que vous pouvez utiliser la commande ip link pour afficher vos paramètres VRF. Une fois dans le conteneur, nous créons la table VRF, la mettons en marche et la lier à l’interface. Nous rentrons les commandes suivantes sur les différents containers des routeurs PE du vpn X (resp vpn Y):
```
link add VRF_X type vrf table 10
link set VRF_X up
link set port_CE1 master VRF_X  
```

#
### Mise en place d'un routage par route static
Les éléments concernant par cette de communication se trouve dans le sous réseaux du routeur CE1 dans le VPN X.

Les routes statiques sont utilisées pour configurer le routage au sein du VPN. Pour le CE (Customer Edge), une route par défaut est configurée vers son PE (Provider Edge) pour toutes les destinations non locales. Par exemple, sur le CE de l'entreprise Y, une route par défaut est ajoutée dirigée vers son PE.

Pour le PE, une route statique est configurée pour rejoindre son côté du VPN. Cette route statique est partagée avec les autres PEs du VPN via MP-BGP (Multiprotocol Border Gateway Protocol). Cela permet à chaque PE de connaître les routes statiques des autres PEs du VPN.

MP-BGP est utilisé pour transporter les informations liées au VPN, notamment les routes vers les CEs. Les PEs redistribuent les routes statiques dans MP-BGP afin que les autres PEs puissent les connaître. Des étiquettes MPLS sont également ajoutées aux routes pour faciliter le transfert des paquets dans le réseau MPLS.

En résumé, les routes statiques sont utilisées pour configurer le routage entre les CEs et les PEs du VPN, tandis que MP-BGP est utilisé pour partager ces informations de routage entre les PEs du VPN. Cela permet d'établir une connectivité efficace et sécurisée entre les sites du VPN.
Les différentes commandes à mettre en place sur CE1 et PE1 sont les suivantes:  

Pour CE1_host :
```
CE1_host:~# ip route add default via 172.16.10.1 dev CE1router
```
Pour CE1_router:
```
conf t
CE1_router# ip route 172.16.0.0/12 10.0.13.2
```

Pour PE1_router:
```
PE1_router# ip route 172.16.10.0/24 10.0.13.1 vrf VRF_X
PE1_router# conf t
PE1_router(config)# router bgp 10 vrf VRF_X
PE1_router(config-router)# address-family ipv4
PE1_router(config-router-af)# redistribute static
PE1_router(config-router-af)# label vpn export auto
PE1_router(config-router-af)# rd vpn export 1:100
PE1_router(config-router-af)# rt vpn both 10:100
PE1_router(config-router-af)# export vpn
PE1_router(config-router-af)# import vpn
```

sur PE1_router on a donc les config suivantes :

```
...
vrf VRF_X
 ip route 172.16.10.0/24 10.0.13.1
 exit-vrf
!
interface port_CE1 vrf VRF_X
 ip address 10.0.13.2/24
 ip ospf cost 1
!
...
 address-family ipv4 vpn
  neighbor 10.155.0.1 activate
  neighbor 10.158.0.1 activate
 exit-address-family
!
router bgp 10 vrf VRF_X
 !
 address-family ipv4 unicast
  redistribute static
  label vpn export auto
  rd vpn export 1:100
  rt vpn both 10:100
  export vpn
  import vpn
 exit-address-family
!
```
On voit bien que la route vers le vpn X est bien présente. On pourra donc faire un ping depuis CE1_host vers CE3_host et vice versa lorsque les configurations seront faites sur les autres routeurs du vpn X.



#
### Mise en place d'une communication par OSPF

Nous utilisons le protocole OSPF pour établir une communication entre le routeur CE3 et le routeur PE3. Cette configuration permet aux deux équipements de partager leurs routes et d'échanger des informations de routage. Le routeur PE3 redistribue ensuite les routes OSPF apprises dans le protocole de routage BGP (Border Gateway Protocol) utilisé dans le réseau VPN. Cela permet aux autres routeurs PE du réseau de connaître les routes OSPF associées à ce VRF. En utilisant les commandes d'exportation et d'importation VPN, les routes sont partagées entre le VRF et la table de routage globale, assurant ainsi une connectivité et une communication efficaces entre les différents équipements du réseau VPN.



```
PE3_router# conf t
PE3_router(config)# router ospf vrf VRF_X
PE3_router(config-router)# network 10.0.15.0/24 area 0
```
```
CE3_router# conf t
CE3_router(config)# router ospf
CE3_router(config-router)# network 10.0.15.0/24 area 0
CE3_router(config-router)# network 172.18.10.0/24 area 0
CE3_router(config-router)# network 192.168.10.3/32 area 0
```
```
PE3_router# conf t
PE3_router(config)# router bgp 10 vrf VRF_X
PE3_router(config-router)# address-family ipv4
PE3_router(config-router-af)# redistribute ospf
PE3_router(config-router-af)# label vpn export auto
PE3_router(config-router-af)# rd vpn export 1:100
PE3_router(config-router-af)# rt vpn both 10:100
PE3_router(config-router-af)# export vpn
PE3_router(config-router-af)# import vpn
```

Les configurations sur le routeur PE3 sont donc : 
```
interface port_CE3 vrf VRF_X
 ip address 10.0.15.2/24
 ip ospf cost 1
!
...
 address-family ipv4 vpn
  neighbor 10.156.0.1 activate
  neighbor 10.158.0.1 activate
 exit-address-family
!
router bgp 10 vrf VRF_X
 !
 address-family ipv4 unicast
  redistribute ospf
  label vpn export auto
  rd vpn export 1:100
  rt vpn both 10:100
  export vpn
  import vpn
 exit-address-family
!
...
router ospf vrf VRF_X
 redistribute bgp
 network 10.0.15.0/24 area 0
!
...
```
A partir de maintenant on peut ping les host de CE1 et CE3 entre eux.
#
### Mise en place de RIP

Nous utilisons le protocole RIP pour établir une communication entre le routeur PE6 et le routeur CE6A. Cette configuration permet aux deux équipements de partager leurs routes et d'échanger des informations de routage. Le routeur PE6 redistribue ensuite les routes RIP apprises dans le protocole de routage BGP (Border Gateway Protocol) utilisé dans le réseau VPN. Cela permet aux autres routeurs PE du réseau de connaître les routes RIP associées à ce VRF. En utilisant les commandes d'exportation et d'importation VPN, les routes sont partagées entre le VRF et la table de routage globale, assurant ainsi une connectivité et une communication efficaces entre le routeur PE6 et le routeur CE6A dans le réseau VPN.
Idem pour le routeur CE6B.

```
PE6_router# conf t
PE6_router(config)# router rip vrf VRF_X
PE6_router(config-router)# network 10.0.17.0/24 
```
```
CE6A_router# conf t
CE6A_router(config)# router rip
CE6A_router(config-router)# network 10.0.17.0/24 
CE6A_router(config-router)# network 172.20.10.0/24 
CE6A_router(config-router)# network 192.168.10.5/32 
```
```
PE6_router# conf t
PE6_router(config)# router bgp 10 vrf VRF_X
PE6_router(config-router)# address-family ipv4
PE6_router(config-router-af)# redistribute ospf
PE6_router(config-router-af)# label vpn export auto
PE6_router(config-router-af)# rd vpn export 1:100
PE6_router(config-router-af)# rt vpn both 10:100
PE6_router(config-router-af)# export vpn
PE6_router(config-router-af)# import vpn
```



#
## Réponse aux questions :
### Question 1

1. Que constatez vous? Expliquez le rôle de BGP et de MPLS dans le fonctionnement global:
BGP (Border Gateway Protocol) est utilisé pour l'échange des informations de routage entre les routeurs des différents sites VPN, tandis que MPLS (Multiprotocol Label Switching) est utilisé pour acheminer les paquets à travers le réseau en utilisant des chemins prédéfinis basés sur des étiquettes. BGP permet de déterminer les meilleures routes pour atteindre les destinations spécifiques, tandis que MPLS offre une commutation rapide des paquets et une connectivité sécurisée entre les VPN. En résumé, BGP facilite le routage des données, tandis que MPLS assure un acheminement efficace des paquets.

2. Testez et expérimentez le plan de contrôle comme le plan de données (prenez soin de prendre garde à ECMP si besoin);
Pour le plan de controle:
    - capture d'ecran de show bgp summary et voir que les routes sont correctement établie
    - sho bgp vpn  unicast et voir que les routes vpn sont correctement declaré
    - voir les tables de routages bgp et les filtres

Pour le plan de données:
    - test de ping
    - test de cheminement via traceroute (pareille screen)  
3. Montrez les effets de l’ajout d‘un nouveau préfixe IP privé dans ce VPN depuis le CE de votre choix (pour cela ajoutez lui
une loopback).
il faudra probablement ajusté les filtres

#
## Question 2

Faites en de même pour le VPN Y mais en hub & spoke, avec PE5 en hub. Contrairement à la Q1, votre RT sera adaptée à chaque
rôle et le hub devra relayer la signalisation iBGP.
1. Trouvez plusieurs moyens pour configurer cette solution (au moins deux). Que constatez vous dans chaque cas?
2. Testez et expérimentez le plan de contrôle comme le plan de données.
3. Appliquez des filtres/politiques sur le hub.

#
## Question 3

Faculctatif : Connectez votre hub à l’IXP 81 pour offrir une connexion Internet au VPN Y. Démontrez la connectivité de ses sites
malgré l’utilisation d’un espace d’adressage privée.