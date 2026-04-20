---
layout: default
title: "ARP Suppression et TCAM sous NX-OS"
---

# Contexte
Maquette de préparation sur le déploiement d'une boucle MAN en EVPN-VxLAN sous NX-OS.
Erreur lors de l'activation de la fonctionnalité ARP Suppression sur les VNI.

# Symptôme
Impossible de configurer la fonctionnalité d'ARP Suppression sur un Nexus 9000v en version 10.5.3.F.

```
site2(config-if-nve)# member vni 10010
site2(config-if-nve-vni)# suppress-arp
ERROR: Please configure TCAM region for Ingress ARP-Ether ACL before configuring ARP supression.

site2(config-if-nve-vni)#
```

En creusant, on trouve qu'il est nécessaire d'attribuer de la mémoire TCAM pour la table de suppression ARP.
Ajoutons 256 entrées pour cela:
```
site2(config)# hardware access-list tcam region arp-ether 256
ERROR: Aggregate TCAM region configuration exceeded the available Ingress TCAM slices. Please re-configure.
```

Regardons comment libérer de l'espace sur la TCAM de notre équipement:

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

Le plus gros consommateur de mémoire TCAM sur notre Nexus 9000v est IPV4 RACL avec 1536.
Dans cet environnement, les tables de routage IPv4 seront de tailles réduites, on peut donc se permettre de réduire sa taille à 512:
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

Après redémarrage on valide la modification:
```
site2(config)# show hardware access-list tcam region | include racl
                                IPV4 RACL [racl] size =  512 
                           IPV6 RACL [ipv6-racl] size =    0 
                       Egress IPV4 RACL [e-racl] size =  768 
                  Egress IPV6 RACL [e-ipv6-racl] size =    0 
                 Ingress RACL v4 & v6 [racl-all] size =    0 
site2(config)#
```

Puis on attribue de la mémoire qui sera pour l'ARP suppression:
```
site2(config)# hardware access-list tcam region arp-ether 256 double-wide 
Warning: Please save config and reload the system for the configuration to take  effect
site2(config)# 2026 Apr 20 09:35:57 site2 %$ VDC-1 %$ pltfm_config[9616]: WARNING: Configuring  the arp-ether region without "double-wide" is deprecated and can result in silent non-vxlan packet drops. Use the "double-wide" keyword when carving TCAM space for the arp-ether region.

site2(config)# 
```

On sauvegarde la configuration puis redémarrage de l'équipement. Après redémarrage on revalide une dernière fois:
```
site2# show hardware access-list tcam region | inclu arp
               Ingress ARP-Ether ACL [arp-ether] size =  256 (double-wide)
                       N9K ARP ACL [n9k-arp-acl] size =    0 
site2# 
```

# Validation du fonctionnement de l'ARP Suppression
- Validation de l'activation sur le VNI correspondant au VLAN 10
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

- Consulter le cache de suppression ARP
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

- Prouver que l'interception fonctionne
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
