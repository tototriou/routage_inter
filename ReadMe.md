# Projet routage inter-domaine

## sujet 

L’objectif principal de ce TP / mini-projet est d’inter-connecter les différents sous-sites, au moyen de VPN-BGP MPLS, avec un
espace d’adressage privé (attention de ne pas rentrer en conflit avec les autres domaines si vous voulez atteindre la dernière
question facultative) et re-utilisé pour les deux VPN. Votre domaine, son routage interne, et ses liens avec les CE sont déjà préconfigurés; en revanche les liens CE avec leur hôte sont laissés à vos soins ainsi que toute la configuration plus avancée (MPLS,
session iBGP et routage des adresses privées sur le CE – avec leur hôte ou avec des loopback locales).
Pour atteindre les configurations souhaitées vous devrez expérimenter deux pistes : en full-mesh et en hub & spoke. En option
finale, vous pourrez essayer de connecter vos hubs et leurs clients à l’Internet...

## Question 1

Commencez par vous occuper du VPN X en full-mesh : déployez des VRF avec un RD arbitraire dans les PE concernés et activez
MPLS dans l’ensemble du réseau. Définissez une RT commune entre les PE.
1. Que constatez vous? Expliquez le rôle de BGP et de MPLS dans le fonctionnement global;
2. Testez et expérimentez le plan de contrôle comme le plan de données (prenez soin de prendre garde à ECMP si besoin);
3. Montrez les effets de l’ajout d‘un nouveau préfixe IP privé dans ce VPN depuis le CE de votre choix (pour cela ajoutez lui
une loopback).

## Question 2

Faites en de même pour le VPN Y mais en hub & spoke, avec PE5 en hub. Contrairement à la Q1, votre RT sera adaptée à chaque
rôle et le hub devra relayer la signalisation iBGP.
1. Trouvez plusieurs moyens pour configurer cette solution (au moins deux). Que constatez vous dans chaque cas?
2. Testez et expérimentez le plan de contrôle comme le plan de données.
3. Appliquez des filtres/politiques sur le hub.

## Question 3

Faculctatif : Connectez votre hub à l’IXP 81 pour offrir une connexion Internet au VPN Y. Démontrez la connectivité de ses sites
malgré l’utilisation d’un espace d’adressage privée.