---
layout: default
title: "ARP Suppression et TCAM sous NX-OS"
---

# ARP Suppression et TCAM sous NX-OS : Résolution pas-à-pas

## Contexte
Cet article documente un cas d'usage rencontré lors d'une maquette de préparation pour le déploiement d'une boucle MAN en environnement EVPN-VxLAN sous Cisco NX-OS (Nexus 9000v en version 10.5.3.F). 

L'objectif initial était d'optimiser le trafic BUM (Broadcast, Unknown Unicast, Multicast) en activant la fonctionnalité **ARP Suppression** sur les VNI (VXLAN Network Identifiers). Cependant, l'activation ne s'est pas passée comme prévu.

## Étape 1 : Le Symptôme et le Diagnostic
Lorsque l'on tente d'activer l'ARP Suppression directement sous l'interface NVE, le système rejette la commande avec un message d'erreur explicite :

```text
site2(config-if-nve)# member vni 10010
site2(config-if-nve-vni)# suppress-arp
ERROR: Please configure TCAM region for Ingress ARP-Ether ACL before configuring ARP supression.
```
L'ARP Suppression n'est pas qu'une simple fonctionnalité logicielle. Pour que le switch intercepte les requêtes ARP à la vitesse du silicium (ou de l'ASIC émulé), il a besoin d'espace dans une mémoire matérielle ultra-rapide appelée TCAM (Ternary Content-Addressable Memory). Par défaut, aucune mémoire TCAM n'est allouée pour intercepter les paquets ARP entrants (arp-ether).

## Étape 2: L'Échec de l'Allocation Directe
En suivant la logique du message d'erreur, la première tentative consiste à allouer directement 256 entrées dans la TCAM pour cette fonctionnalité :

```
site2(config)# hardware access-list tcam region arp-ether 256 double-wide
ERROR: Aggregate TCAM region configuration exceeded the available Ingress TCAM slices. Please re-configure.
```

La TCAM est une ressource finie et précieuse, surtout sur des Nexus virtuelles de Lab. Sur ce Nexus, l'intégralité de l'espace disponible est déjà pré-allouée à d'autres fonctionnalités par défaut. Nous sommes face à un problème d'espace matériel.

## Étape 3: Auditer et Libérer de l'Espace TCAM
Pour pouvoir allouer de la mémoire à l'ARP Suppression, il faut en retirer ailleurs. C'est ce qu'on appelle le TCAM Carving (le découpage de la TCAM).
Regardons comment est actuellement répartie la mémoire :

```
site2# show hardware access-list tcam region 
                               IPV4 PACL [ifacl] size =    0 
[...]
                                IPV4 RACL [racl] size = 1536 
[...]
                       Egress IPV4 RACL [e-racl] size =  768 
                  Egress IPV6 RACL [e-ipv6-racl] size =    0 
               Egress IPV4 QoS Lite [e-qos-lite] size =    0 
                             IPV4 L3 QoS [l3qos] size =  256 
                        IPV6 L3 QoS [ipv6-l3qos] size =    0 
                          MAC L3 QoS [mac-l3qos] size =    0 
                                  Ingress System size =  256 
                                   Egress System size =  256 
                                     SPAN [span] size =  256 
                             Ingress COPP [copp] size =  256 
                    Ingress Flow Counters [flow] size =    0 
                   Egress Flow Counters [e-flow] size =    0 
                      Ingress SVI Counters [svi] size =    0 
                             Redirect [redirect] size =  256 
 VPC Convergence/ES-Multi Home [vpc-convergence] size =  512 
                  IPSG SMAC-IP bind table [ipsg] size =    0 
               Ingress ARP-Ether ACL [arp-ether] size =    0 
             ranger+ IPV4 QoS Lite [rp-qos-lite] size =    0 
                       ranger+ IPV4 QoS [rp-qos] size =  256 
                  ranger+ IPV6 QoS [rp-ipv6-qos] size =  256 
                    ranger+ MAC QoS [rp-mac-qos] size =  256 
[...] 
Ingress SVC Redir L3 Pass-1 IPV4 [ing-svc-redir-l3-pass-1-ipv4] size =    0 
Ingress SVC Redir L3 Pass-1 IPV6 [ing-svc-redir-l3-pass-1-ipv6] size =    0 
Ingress SVC Redir L3 Pass-2 IPV4 [ing-svc-redir-l3-pass-2-ipv4] size =    0 
Ingress SVC Redir L3 Pass-2 IPV6 [ing-svc-redir-l3-pass-2-ipv6] size =    0
```

En analysant la sortie, on constate que le plus gros consommateur est IPV4 RACL (Routed Access Control Lists) avec 1536 entrées. Dans notre environnement EVPN-VxLAN, le filtrage IPv4 L3 intensif n'est pas la priorité du VTEP, et nos tables de routage seront réduites.
Nous allons donc réduire drastiquement la taille allouée aux RACL IPv4 (de 1536 à 512) pour libérer des "slices" (tranches) de mémoire, sauvegarder, puis redémarrer :

```
site2(config)# hardware access-list tcam region racl 512
Warning: Please save config and reload the system for the configuration to take  effect
site2(config)# copy run start
[########################################] 100%
Copy complete, now saving to disk (please wait)...
Copy complete.
site2(config)# reload
This command will reboot the system. (y/n)?  [n] y
```

**Note Importante**: Les modifications de la table TCAM nécessitent toujours une sauvegarde de la configuration et un redémarrage complet (reload) de l'équipement pour que le matériel réalloue la mémoire.

Après redémarrage, on valide que l'espace a bien été réduit :
```
site2(config)# show hardware access-list tcam region | include racl
                                IPV4 RACL [racl] size =  512 
                           IPV6 RACL [ipv6-racl] size =    0 
                       Egress IPV4 RACL [e-racl] size =  768 
                  Egress IPV6 RACL [e-ipv6-racl] size =    0 
                 Ingress RACL v4 & v6 [racl-all] size =    0 
site2(config)#
```

## Étape 4: Configuration Finale de l'ARP-Ether
Maintenant que nous avons libéré de l'espace, nous pouvons allouer la mémoire pour l'interception ARP.
**Attention au piège de l'architecture VXLAN** : Il est impératif d'utiliser l'argument double-wide. Les paquets VXLAN étant encapsulés, les clés de recherche dans la TCAM sont plus larges que la normale.

```
site2(config)# hardware access-list tcam region arp-ether 256 double-wide 
Warning: Please save config and reload the system for the configuration to take  effect
site2(config)# 2026 Apr 20 09:35:57 site2 %$ VDC-1 %$ pltfm_config[9616]: WARNING: Configuring  the arp-ether region without "double-wide" is deprecated and can result in silent non-vxlan packet drops. Use the "double-wide" keyword when carving TCAM space for the arp-ether region.

site2(config)# 
```

On sauvegarde à nouveau et on redémarre l'équipement. Une dernière validation confirme le succès de l'opération :

```
site2# show hardware access-list tcam region | inclu arp
               Ingress ARP-Ether ACL [arp-ether] size =  256 (double-wide)
                       N9K ARP ACL [n9k-arp-acl] size =    0 
site2# 
```

*Note: On peut maintenant retourner sur l'interface NVE et activer suppress-arp sans générer d'erreur).*

# Étape 5: Validation du fonctionnement
Une fois l'infrastructure prête et la configuration appliquée, voici les trois commandes essentielles pour s'assurer que l'ARP Suppression intercepte et répond correctement au trafic BUM.

**- Vérifier l'activation sur le VNI (Ex: VLAN 10 ici)**
Le flag **SA** (Suppress ARP) doit être présent sur la ligne de votre VNI.
	- `show nve vni 10010`

```
site1# show nve vni 10010
Codes: CP - Control Plane        DP - Data Plane
       UC - Unconfigured         SA - Suppress ARP
       S-ND - Suppress ND
       SU - Suppress Unknown Unicast
       Xconn - Crossconnect
       MS-IR - Multisite Ingress Replication
       HYB - Hybrid IRB mode

Interface VNI      Multicast-group   State Mode Type [BD/VRF]      Flags
--------- -------- ----------------- ----- ---- ------------------ -----
nve1      10010    UnicastBGP        Up    CP   L2 [10]            SA

site1#
```

**- Consulter le cache de suppression ARP**
Cette table affiche les correspondances IP/MAC que le switch a apprises (soit localement, soit via BGP EVPN) et pour lesquelles il est capable de répondre directement.
	- `show ip arp suppression-cache vlan 10`

```
site1# show ip arp suppression-cache vlan 10

Flags: + - Adjacencies synced via CFSoE
       L - Local Adjacency
       R - Remote Adjacency
       L2 - Learnt over L2 interface
       PS - Added via L2RIB, Peer Sync
       RO - Dervied from L2RIB Peer Sync Entry

Ip Address      Age      Mac Address    Vlan Physical-ifindex    Flags    Remote Vtep Addrs

192.168.10.101  00:02:26 5254.00be.9338   10 Ethernet1/3         L
site1#

```

**- Prouver que l'interception fonctionne**
Cette commande est cruciale pour le troubleshooting. Elle permet de voir concrètement si le switch intercepte des requêtes et y répond localement (Local Responses).
	- `show ip arp suppression-cache statistics vlan 10`

```
site1# show ip arp suppression-cache statistics

ARP packet statistics for suppression-cache

Suppressed:
Total 0, Requests 0, Requests on L2 0, Gratuitous 0, Gratuitous on L2 0

Forwarded :
Total: 14
 L3 mode :      Requests 11, Replies 3
                Request on core port 11, Reply on core port 3,
                Flood ARP Probe 0,
                Dropped 0
 L2 mode :      Requests 0, Replies 0
                Request on core port 0, Reply on core port 0,
                Flood ARP Probe 0,
                Dropped 0

Received:
Total: 14
 L3 mode:       Requests 11, Replies 3
                Local Request 0, Local Responses 0
                Gratuitous 0, Dropped 0
 L2 mode :      Requests 0, Replies 0
                Gratuitous 0, Dropped 0
ARP suppression-cache Local entry statistics
Adds 26, Deletes 0

site1#
```
