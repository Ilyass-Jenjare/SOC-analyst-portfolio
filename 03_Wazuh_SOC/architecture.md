# Architecture et Segmentation du Laboratoire SOC (V2)

> Document d'architecture du laboratoire SOC Wazuh, version V2 : segmentation réseau, politiques de pare-feu FortiGate et défense active, selon les principes de **Zero Trust Architecture (ZTA)** et de **défense en profondeur**.

| | |
|---|---|
| **Projet** | SOC Lab Wazuh (Blue Team / Purple Team) |
| **Version** | V2 — intégration d'un pare-feu nouvelle génération |
| **Pare-feu** | Fortinet FortiGate (VM) |
| **SIEM** | Wazuh (Manager / Indexer / Dashboard) |
| **Hyperviseur** | VMware Workstation Pro |
| **Document lié** | [README.md](README.md) |

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Schéma logique](#2-schéma-logique)
3. [Plan d'adressage et rôle des machines](#3-plan-dadressage-et-rôle-des-machines)
4. [Zones de sécurité](#4-zones-de-sécurité)
5. [Politiques de filtrage](#5-politiques-de-filtrage)
6. [Défense active : DoS Policy](#6-défense-active--dos-policy)
7. [Flux de télémétrie vers Wazuh](#7-flux-de-télémétrie-vers-wazuh)
8. [Limites et points de vigilance](#8-limites-et-points-de-vigilance)
9. [Compétences mobilisées](#9-compétences-mobilisées)

---

## 1. Vue d'ensemble

La V2 du laboratoire place un **FortiGate** comme passerelle centrale. Il segmente l'environnement en trois zones distinctes :

| Zone | Interface | Réseau VMware | Rôle |
|---|---|---|---|
| **WAN** | Port 1 | VMnet8 (NAT) | Accès Internet |
| **LAN** | Port 2 | VMnet2 (Host-Only) | Réseau interne : supervision, poste offensif, poste client |
| **DMZ** | Port 3 | Host-Only | Services exposés (cible vulnérable) |

**Objectifs de conception**

- Appliquer le **moindre privilège** sur tous les flux inter-zones.
- Considérer chaque zone, y compris le LAN, comme **potentiellement hostile**.
- Simuler une **menace interne** (poste compromis) en plaçant l'attaquant dans le LAN.
- Produire une **télémétrie réseau** exploitable par le SIEM pour la détection et le Threat Hunting.

---

## 2. Schéma logique

```text
                                 [ INTERNET ]
                                       |
                                (VMnet8 - NAT)
                                 Port 1 (WAN)
                               +--------------+
                               | FORTIGATE VM |
                               +--------------+
                                 |          |
                   Port 2 (LAN) -+          +- Port 3 (DMZ)
               (VMnet2 - Host-Only)         (Host-Only)
                                 |          |
         +-----------------+-----+          +----------------------+
         |                 |     |                                 |
  [ Kali attaquant ] [ Wazuh ] [ Client Windows ]        [ Serveur Victim Web ]
                               Poste de l'entreprise
```

---

## 3. Plan d'adressage et rôle des machines

| Machine | Zone | Adresse IP | Rôle |
|---|---|---|---|
| FortiGate (passerelle LAN) | LAN | `10.0.10.254` | Passerelle, filtrage, NAT, DoS Policy |
| Wazuh Server | LAN | `10.0.10.50` | SIEM central (Manager, Indexer, Dashboard) |
| Kali Linux | LAN | `10.0.10.30` | Poste offensif (menace interne simulée) |
| Client Windows | LAN | `10.0.10.40` | Poste de travail, extension de la couverture au-delà de Linux |
| Serveur Victim Web | DMZ | `192.16.10.20` | DVWA + SSH, cible principale des scénarios d'attaque |

---

## 4. Zones de sécurité

### 4.1 Zone LAN (Port 2) — Réseau interne

Héberge les postes de travail et l'infrastructure de supervision. Conformément au modèle Zero Trust, cette zone n'est **pas** considérée comme de confiance.

- **Wazuh Server** : placé dans le LAN pour collecter la télémétrie interne et centraliser l'analyse.
- **Kali Linux** : placé volontairement dans le LAN pour simuler un poste employé compromis et des scénarios de mouvement latéral.
- **Client Windows** : étend la couverture de détection aux environnements Windows.

### 4.2 Zone DMZ (Port 3) — Zone démilitarisée

Héberge les services exposés, isolés du reste du réseau pour **contenir toute compromission**.

- **Serveur Victim Web** : DVWA et service SSH. Isolé du LAN, il constitue la cible des scénarios d'attaque.

---

## 5. Politiques de filtrage

Les règles inter-zones suivent le principe du **moindre privilège**. Tout flux non explicitement autorisé est rejeté (*Implicit Deny*).

| ID | Nom | Source | Destination | Service | Action | NAT | Justification |
|---|---|---|---|---|---|---|---|
| 1 | `LAN_to_Internet` | port2 (all) | port1 (all) | ALL | ACCEPT | Oui | Accès Internet (mises à jour, paquets) pour le LAN. |
| 2 | `LAN_to_DMZ` | port2 (all) | port3 (all) | ALL | ACCEPT | Non | Voie d'accès de l'attaquant (Kali) vers la cible ; génère du trafic inter-zone analysable. |
| 3 | `DMZ_Wazuh` | port3 (all) | port2 (Wazuh Server) | `Wazuh_Ports` (TCP 1514, 1515) | ACCEPT | Non | **Micro-segmentation** : seul l'agent de la DMZ peut remonter ses événements au Manager, sans exposer le reste du LAN. |
| 0 | Implicit Deny | all | all | ALL | DENY | Non | Règle par défaut : tout ce qui n'est pas autorisé est détruit. |

**Détail du service `Wazuh_Ports`** : objet de service personnalisé défini sur le FortiGate.

| Port | Protocole | Usage |
|---|---|---|
| 1515 | TCP | Enrôlement des agents |
| 1514 | TCP | Collecte des événements |

---

## 6. Défense active : DoS Policy

En complément du filtrage L3/L4, une **DoS Policy** comportementale est déployée sur l'interface LAN (`port2`).

### 6.1 Positionnement

Dans une architecture Zero Trust, la menace peut venir de l'intérieur. Placer le capteur sur le LAN protège la DMZ contre une machine interne compromise (ici Kali) tentant d'épuiser les ressources du serveur web, par exemple par *TCP SYN Flood* ou *ICMP Sweep*.

### 6.2 Paramètres

| Paramètre | Valeur |
|---|---|
| Interface | `port2` (entrante) |
| Menaces ciblées | TCP SYN Flood, ICMP Flood / Sweep |
| Seuil (phase de test) | 50 paquets/seconde |
| Action | `Block` |
| Journalisation | `Logging: Enable` |

### 6.3 Mécanisme d'Early Drop

Les capteurs d'anomalies utilisent l'**Early Drop** : le FortiGate détruit le trafic malveillant **avant** de le soumettre aux règles de filtrage standard, ce qui réduit la charge et la surface d'exposition.

---

## 7. Flux de télémétrie vers Wazuh

Chaque blocage par la DoS Policy génère un événement Syslog critique. Le circuit de collecte est le suivant :

```text
 FortiGate (DoS Policy : Block + Logging)
        │
        │  Syslog (UDP 514)
        ▼
 Serveur Wazuh (Agent 000, écoute locale)
        │
        ▼
 Archivage dans wazuh-archives-*  →  Threat Hunting / détection
```

| Source | Transport | Destination | Usage |
|---|---|---|---|
| FortiGate (DoS Policy) | Syslog UDP 514 | Wazuh Server (Agent 000) | Télémétrie réseau, Threat Hunting |
| Agent Wazuh de la DMZ | TCP 1514 / 1515 | Wazuh Manager (LAN) | Événements hôte (logs, FIM, SCA) |

Cette double source (réseau via FortiGate, hôte via agent) permet de **corréler** les événements du pare-feu avec ceux de la machine ciblée.

---

## 8. Limites et points de vigilance

| Limite | Impact | Piste d'amélioration |
|---|---|---|
| Machines d'un même segment LAN : trafic latéral **non inspecté** par le FortiGate | Le trafic Kali ↔ Client Windows échappe au pare-feu ; sa détection repose uniquement sur Wazuh (agents et règles) | Segmenter davantage le LAN ou activer l'inspection intra-zone |
| Règles 1 et 2 en `ALL` (service et adresses) | Moins strictes que le moindre privilège idéal | Restreindre aux services strictement nécessaires |
| Seuil DoS abaissé à 50 pps | Adapté aux tests, risque de faux positifs en usage réel | Calibrer le seuil sur une baseline de trafic |
| Syslog en UDP | Pas de garantie de livraison | Passer à Syslog TCP/TLS |

---

## 9. Compétences mobilisées

- **Architecture réseau sécurisée** : segmentation WAN / LAN / DMZ, Zero Trust, micro-segmentation.
- **Administration de pare-feu Fortinet** : politiques de filtrage, objets de service, NAT, DoS Policy.
- **Détection et supervision** : collecte Syslog, corrélation hôte / réseau dans Wazuh, Threat Hunting.
- **Approche Purple Team** : attaquant interne simulé, validation de la détection par attaques contrôlées.
- **Documentation technique** : justification de chaque choix de conception et limites identifiées.
