# SOC Lab Wazuh — V1 Linux

**Détection d'intrusion et validation par attaques contrôlées (Blue Team / Purple Team)**

![Status](https://img.shields.io/badge/status-en%20cours-yellow)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%204.9-success)

> Projet personnel de cybersécurité mené en parallèle d'un encadrement en entreprise. Ce dépôt documente le déploiement d'un SOC miniature autour de Wazuh, la construction d'une cible Linux volontairement vulnérable, et l'émulation d'attaques réelles pour observer et améliorer la capacité de détection.

---

## Sommaire

1. [Introduction](#01-introduction)
2. [Architecture](#02-architecture)
3. [Infrastructure](#03-infrastructure)
4. [Implémentation](#04-implémentation)
5. [Scénarios de sécurité](#05-scénarios-de-sécurité)
6. [Détection et investigation](#06-détection-et-investigation)
7. [Limites et améliorations](#07-limites-et-améliorations)
8. [Conclusion](#08-conclusion)
9. [Références](#09-références)

---

## 01. Introduction

### Contexte

Ce projet a été réalisé dans le cadre d'un encadrement en entreprise, avec pour objectif de monter en compétences en détection d'intrusion (**Blue Team**) et de valider cette détection par des attaques contrôlées (**Purple Team**).

### Objectifs

- Déployer un SOC miniature fonctionnel autour de **Wazuh** (SIEM/XDR open source).
- Construire une cible Linux volontairement vulnérable (DVWA), exploitable en CLI.
- Émuler des attaques réelles et observer leur détection.
- Écrire des règles de détection personnalisées pour combler les lacunes identifiées.
- Produire une documentation et des livrables exploitables pour un portfolio cybersécurité (candidature SOC, alternance, stage).

### Objectifs pédagogiques

- Maîtriser le déploiement et l'administration de Wazuh (Manager, Indexer, agents).
- Comprendre et exploiter les logs systèmes Linux (`auth.log`, `syslog`, logs applicatifs Apache).
- Savoir cartographier des attaques sur le référentiel **MITRE ATT&CK**.
- Développer un réflexe de detection engineering : attaquer, mesurer, corriger, revalider.

### Périmètre

| Dans le périmètre (V1 Linux) | Hors périmètre (V1) |
|---|---|
| Wazuh Manager/Indexer/Dashboard (all-in-one) | Cluster Wazuh multi-nœuds |
| 1 cible Ubuntu Server (SSH + Apache + DVWA) | Cibles Windows |
| 1 poste offensif Kali (CLI) | Automatisation SOAR avancée |
| Détection réseau via Suricata (NIDS) | EDR commercial tiers |
<<<<<<< HEAD:06_Wazuh_SOC/readme.md
| FIM, SCA, Active Response base, Notifications d'alertes par email | Threat Intel feeds externes payants |
=======
| FIM, SCA, Active Response de base, alertes par email | Threat Intel feeds externes payants |
>>>>>>> ecbb0837ff7109b2d3c12826193cc3f18c5f3338:06_Wazuh_SOC_V1_Linux/readme.md

### Contraintes du projet

| Type | Détail |
|---|---|
| Matérielle | Laptop 16 Go RAM, hyperviseur VMware Workstation Pro, pas d'interface graphique sur les VM pour limiter la RAM |
| Sécurité | Lab totalement isolé du réseau domestique (réseau NAT/host-only interne à l'hyperviseur), aucune attaque ne doit sortir du périmètre, identifiants fictifs documentés comme tels |


---

## 02. Architecture

### Schéma logique

```=====================================================================
  LABORATOIRE SOC : Réseau LAN Unique (Intranet & Mouvement Latéral)
=====================================================================

                            +-------------------+
                            |     INTERNET      |
                            +---------+---------+
                                      | (VMnet8)
+-------------------------------------------------------------------+
|                      FORTIGATE (Edge Firewall)                    |
|                                                                   |
| [Port 1 - WAN / Extérieur]                [Port 2 - LAN / Interne]|
| IP: DHCP / NAT                              IP: 10.0.10.254       |
+-------------------------------------------------------------------+
                                      | 
  +-----------------------------------+-----------------------------------+
  |                                (VMnet2)                               |
  |               |                               |                       |
+-+-------------+ +-------------+ +---------------+ +-------------------+ |
| Kali Linux    | | Serveur Linux | Client Windows| | Serveur Wazuh     | |
| (Attaquant)   | | (Intranet)  | | (Employé)     | | (Manager SOC)     | |
| IP:10.0.10.30 | |10.0.10.20   | | IP:10.0.10.40 | | IP:10.0.10.50     | |
+---------------+ +-------------+ +---------------+ +-------------------+ |
        |                 ▲               ▲                  ▲            |
        |  Attaque directe|               |                  |            |
        +-----------------+               |                  |            |
        |                                 |                  |            |
        |   (Logs de l'agent Wazuh)       |                  |            |
        +----------------------------------------------------+            | 
    
```

Flux de données : chaque attaque lancée depuis Kali touche la machine Victime (service SSH, application web DVWA). Les événements générés (`auth.log`, `access.log`, `eve.json` de Suricata, checks FIM/SCA) sont collectés par l'agent Wazuh installé sur la Victime, transmis au Manager sur le port **1514** (événements) et **1515** (enrôlement), corrélés par le moteur de règles, puis visualisés dans le Dashboard consulté depuis le navigateur de l'hôte Windows.

### Choix « CLI only »

- Les trois VM (Wazuh, Kali, Victime) sont installées en mode **serveur / sans interface graphique**.
- Administration exclusivement en **SSH** depuis le terminal de l'hôte.
- Le dashboard Wazuh (interface web) est consulté depuis le navigateur de l'hôte Windows en pointant vers l'IP de la VM Wazuh — aucune VM n'a besoin d'environnement graphique pour cela.

### Adressage réseau — évolution du lab

> **Note de transparence :** l'adressage IP du lab a évolué au fil des sessions de travail, le réseau NAT de VMware Workstation ayant été réattribué à plusieurs reprises entre les phases de déploiement. Le tableau ci-dessous reflète la plage utilisée. Cette dérive d'adressage est elle-même documentée comme point de vigilance en [section 07](#07-limites-et-améliorations).

| Plage NAT | Wazuh Manager | Victime | Kali |
|---|---|---|---|
| `192.168.111.0/24` | `192.168.111.100` | `192.168.111.150` | `192.168.111.200` |


Pour toute reproduction du lab, fixer la plage NAT **avant** la configuration réseau de chaque VM et la garder stable sur toute la durée du projet évite cette dérive.

---

## 03. Infrastructure

### Dimensionnement des machines

| VM | Rôle | OS | RAM allouée | vCPU | Disque |
|---|---|---|---|---|---|
| **Machine 1 — Wazuh** | Manager + Indexer + Dashboard | Ubuntu Server 22.04 LTS (CLI) | 4 Go | 2 | 40 Go |
| **Machine 2 — Kali** | Poste offensif (Red Team) | Kali Linux 2026.x (CLI/minimal) | 2 Go | 2 | — |
| **Machine 3 — Victime** | Cible (SSH, web, agent Wazuh) | Ubuntu Server 22.04 LTS (CLI) | 1,5 Go | 1 | 20 Go |
| **Total VM actives** | | | **~7,5 Go** | | Marge confortable sur 16 Go |

### Stack logicielle

| Composant | Machine | Rôle |
|---|---|---|
| Wazuh Manager | M1 | Corrélation, moteur de règles, alertes |
| Wazuh Indexer | M1 | Stockage des événements (OpenSearch) |
| Wazuh Dashboard | M1 | Visualisation (consultée depuis l'hôte, port 443) |
| Postfix (relais SMTP) | M1 | Envoi des notifications d'alertes par email |
| Wazuh Agent (v4.9.2) | M3 | Collecte logs, FIM, SCA |
| OpenSSH Server | M3 | Service exposé, cible de brute force |
| Apache2 + PHP + MariaDB | M3 | Serveur web applicatif |
| DVWA | M3 | Application web volontairement vulnérable |
| Suricata | M3 | Détection réseau (NIDS), écrit dans `eve.json` |
| nmap, hydra, sqlmap, curl, netcat, Wireshark | M2 | Arsenal offensif CLI |

### Gestion des ressources

- Hyperviseur : **VMware Workstation Pro**.
- Snapshots de référence pris à chaque jalon stable : `01-Wazuh-Ready`, `01-Kali-Ready`, `01-Victime-Ready`, puis snapshot global `02-Lab-Operational`.
- Bonne pratique appliquée : **Kali reste éteinte** en dehors des phases d'attaque pour limiter la consommation RAM continue sur le laptop hôte.

---

## 04. Implémentation

Cette section résume, machine par machine, ce qui a été déployé et pourquoi. **Le détail pas-à-pas (commandes exactes, fichiers de configuration, captures de validation) est tenu dans les pages Notion du projet**, référencées en [section 09](#09-références), afin d'éviter toute redondance avec ce README.

### 4.1 Machine 1 — Serveur SIEM Wazuh

La VM a été installée en Ubuntu Server minimal avec OpenSSH activé dès l'installeur, puis dotée d'une adresse IP statique pour garantir que les agents et les attaquants pointent toujours vers la même adresse sans dépendre d'un bail DHCP.
Une intégration SMTP a également été configurée au niveau du Manager pour le routage des alertes critiques par email, permettant une notification en temps réel de l'analyste.
La stack Wazuh (Manager, Indexer, Dashboard) a été déployée en mode **all-in-one** via le script d'installation assistée officiel, qui gère automatiquement les certificats TLS entre les trois composants. Les identifiants administrateur générés à l'issue de l'installation sont conservés dans un gestionnaire de mots de passe, jamais commités sur le dépôt.

Un premier niveau de durcissement a ensuite été appliqué via un pare-feu applicatif (UFW), n'autorisant que les ports strictement nécessaires : SSH, HTTPS pour le dashboard, et les ports d'enrôlement/transmission des agents. Un scan de validation depuis Kali confirme qu'aucun autre port n'est exposé.

Une intégration SMTP a ensuite été configurée sur le Manager : Wazuh ne gérant pas nativement l'authentification TLS exigée par les fournisseurs de messagerie, un relais local **Postfix** (TLS, authentification par mot de passe d'application) assure l'envoi, et le Manager lui transmet les alertes dépassant un seuil de sévérité défini (niveau 10) afin de notifier l'analyste par email. Les identifiants du relais restent hors du dépôt, comme les autres secrets.

→ Détail des sous-étapes et commandes : page Notion **« Serveur SIEM (Wazuh) »**.

### 4.2 Machine 2 — Poste offensif Kali Linux

Déployée à partir de l'image officielle VMware, la VM a été volontairement bridée en ressources (2 Go RAM, 2 vCPU) pour rester cohérente avec la contrainte des 16 Go du laptop hôte, et dotée d'une adresse IP statique afin que toutes les attaques simulées proviennent d'une source identifiable dans le SIEM.

La disponibilité de l'arsenal offensif (`nmap`, `hydra`, `sqlmap`, `netcat`, `Wireshark`) a été vérifiée, puis un script de reconnaissance/brute force réutilisable (`recon_and_bruteforce.sh`) a été préparé pour pouvoir rejouer le même scénario d'attaque de façon identique d'une session de test à l'autre.

→ Détail des sous-étapes, script complet et wordlist : page Notion **« Kali Linux »**.

### 4.3 Machine 3 — Cible vulnérable (SSH + Web + agent)

Cette machine concentre les deux surfaces d'attaque du lab : un service SSH exposé (cible de brute force) et une application web volontairement vulnérable.

**Service web :** Apache, PHP et MariaDB ont été installés, puis DVWA a été cloné depuis le dépôt officiel et connecté à une base de données dédiée. Le niveau de sécurité de DVWA a été positionné à `low` pour les premiers tests. Les identifiants par défaut de DVWA sont explicitement documentés comme fictifs.

**Détection réseau :** Suricata a été installé en complément des capacités HIDS de Wazuh, pour combler sa limite native de visibilité réseau (voir section 06). Il écrit ses alertes dans un fichier JSON, ensuite lu par l'agent Wazuh — validé par un scan de test générant une entrée correspondante.

**Agent Wazuh :** installé et enrôlé auprès du Manager, avec ajout de la lecture du log Suricata comme source supplémentaire. Le module **FIM** (surveillance d'intégrité des fichiers) a été activé en temps réel sur le répertoire DVWA et sur `/etc`, et le module **SCA** (évaluation de la configuration de sécurité face au CIS Benchmark) a été activé avec un scan toutes les 12 heures. Un test manuel de dépôt de fichier a permis de valider le déclenchement d'une alerte FIM.

→ Détail des sous-étapes, fichiers `ossec.conf` et scripts : page Notion **« Victime (Ubuntu Server) »**.

### 4.4 Incidents de configuration rencontrés

Trois incidents réels ont été rencontrés pendant l'enrôlement de l'agent, conservés ici à titre de retour d'expérience plutôt que comme étapes de configuration :

- **Erreur d'assignation de groupe** côté base de données interne de l'agent (`wazuh-db`), résolue par une correction des permissions et une réassignation manuelle de l'agent à son groupe.
- **Saturation de l'espace disque** sur la VM Victime, résolue par un nettoyage des paquets, journaux et fichiers temporaires.
- **Crash de l'API Wazuh** (`wazuh-apid` ne démarrait plus), causé par des processus zombies bloquant le port de l'API, résolu par un arrêt forcé puis un redémarrage propre du Manager.

→ Commandes de résolution détaillées : page Notion **« Victime (Ubuntu Server) »**.

---

## 05. Scénarios de sécurité

### Méthodologie Purple Team

Chaque scénario suit le même cycle de detection engineering :

1. **Préparation** — définir la technique ATT&CK visée et le résultat attendu en détection.
2. **Exécution de l'attaque** depuis Kali (CLI).
3. **Vérification** dans le dashboard Wazuh (`Security Events`, `Threat Hunting`).
4. **Si non détecté** — analyse du log brut, écriture d'une règle/décodeur custom.
5. **Re-test** de la même attaque pour valider la nouvelle règle.
6. **Documentation** — rédaction du rapport d'incident.

### Règles d'engagement (Rules of Engagement)

- **Cible autorisée :** uniquement la machine Victime du lab.
- **Hors périmètre :** la machine hôte Windows (non testée).
- **Réseau :** environnement isolé/NAT interne, aucune exposition Internet.
- **Objectif :** détection, pas exploitation destructive (pas de DoS, pas de persistance destructrice).
- **Niveau DVWA Security :** tests idéalement menés à plusieurs niveaux (`low`, `medium`, `high`) pour observer si l'obfuscation de la requête échappe aux règles par défaut.

### Scénarios définis, mappés MITRE ATT&CK

| # | Technique | Module DVWA / cible | MITRE ATT&CK | Approche |
|---|---|---|---|---|
| 1 | Reconnaissance réseau | Hôte Victime | T1046 | Scan de ports et services (nmap) |
| 2 | Brute force SSH | Service OpenSSH | T1110.001 | Dictionnaire de mots de passe courts |
| 3 | Brute force applicatif | DVWA — Brute Force | T1110.001 | Formulaire web, détection d'échec de connexion |
| 4 | Injection SQL | DVWA — SQL Injection | T1190 | Extraction de bases via sqlmap |
| 5 | Command Injection | DVWA — Command Injection | T1059.004 / T1190 | Chaînage de commandes dans un champ IP |
| 6 | Cross-Site Scripting | DVWA — XSS (Reflected/Stored/DOM) | T1059.007 / T1189 | Injection de payload JavaScript |
| 7 | Dépôt de webshell | DVWA — File Upload | T1505.003 / T1105 | Upload d'un fichier PHP minimal |
| 8 | Local File Inclusion | DVWA — File Inclusion | T1083 | Traversée de répertoire vers fichiers système |
| 9 | CSRF | DVWA — CSRF | T1185 | Requête forgée hors session légitime |
| 10 | Session IDs faibles | DVWA — Weak Session IDs | T1539 | Analyse de prévisibilité des identifiants de session |

→ Commandes exactes de chaque scénario : page Notion **« Intégration et Validation »** et rapport de pentesting associé.

---

## 06. Détection et investigation

### Sources de données corrélées par Wazuh

| Source / Module | Donne de la visibilité sur (ou permet de) |
|---|---|
| Logs d'authentification système | Brute force SSH, connexions réussies/échouées |
| Logs d'accès Apache | Requêtes web suspectes (SQLi, XSS, LFI) |
| Alertes Suricata (NIDS) | Scans de ports, signatures réseau |
| Syscheck (FIM) | Dépôt de fichiers (ex. webshell), temps réel sur DVWA et `/etc` |
| SCA | Niveau de durcissement vs. CIS Benchmark |
| auditd (recommandé, non actif en V1) | Exécution de commandes post-exploitation |
<<<<<<< HEAD:06_Wazuh_SOC/readme.md
| Intégration SMTP (Alerting) | Notification en temps réel des incidents critiques par email |
=======
| Intégration SMTP (alerting) | Notification de l'analyste par email pour les incidents de sévérité élevée |
>>>>>>> ecbb0837ff7109b2d3c12826193cc3f18c5f3338:06_Wazuh_SOC_V1_Linux/readme.md

### Détection attendue par scénario

- **Reconnaissance réseau (T1046) :** point de vigilance documenté — Wazuh est un **HIDS**, pas un NIDS. Sans Suricata, un simple scan de ports n'est **pas** détecté nativement. Suricata comble cette lacune en alimentant l'agent Wazuh.
- **Brute force SSH / applicatif (T1110) :** couvert par les décodeurs natifs Wazuh sur les logs d'authentification et d'accès web.
- **Injection SQL (T1190) :** détection native possible si le decoder web reconnaît les patterns d'injection ; sinon nécessite une règle custom.
- **Command Injection (T1059) :** détection fiable uniquement si **auditd** est configuré sur l'agent pour tracer l'exécution de commandes par le processus web — non actif en V1, identifié comme axe d'amélioration.
- **Webshell / File Upload (T1505.003) :** scénario de démonstration du **FIM temps réel** — un fichier déposé dans le répertoire d'upload doit déclencher une alerte en quelques minutes.
- **XSS (T1059.007) :** généralement couvert par les règles web par défaut si le decoder applicatif est actif.
- **LFI (T1083) :** pattern de traversée de répertoire dans les logs web, couvert si le décodeur web est actif.
- **CSRF et Weak Session IDs :** signature réseau faible, documentés comme **angles morts probables** d'une détection HIDS classique sans WAF applicatif dédié.

### Exemple de logique de règle custom

Lorsqu'une technique n'est pas couverte par les règles natives (par exemple certains patterns SQLi), le principe retenu est d'écrire une règle `local_rules.xml` qui s'appuie sur la règle web générique de Wazuh et ajoute une détection par motif (`union select`, `information_schema`, etc.), associée à l'identifiant MITRE correspondant. Chaque règle custom ainsi créée est documentée dans `rules/` avec son objectif, sa logique, le log brut qui l'a motivée et la commande de re-test ayant permis de la valider.

### Active Response

Objectif V1 : blocage automatique d'une IP source après dépassement d'un seuil de tentatives de brute force SSH. Statut : **à finaliser** — voir [section 07](#07-limites-et-améliorations).

### Alerting par email

Objectif V1 : notifier automatiquement l'analyste SOC pour toute alerte de sévérité élevée (niveau ≥ 10 dans la configuration retenue), via le relais Postfix du Manager. Le seuil est volontairement élevé pour limiter le bruit, et un plafond horaire d'envoi (`email_maxperhour`) évite la saturation de la boîte de réception, au prix d'un risque de notifications différées lors d'un pic d'alertes. Cette notification est une **action de sortie** : elle complète la détection sans s'y substituer.

### Processus d'investigation dans le Dashboard

1. Module **Security Events** → filtrer par l'IP de la machine Victime.
2. Pour chaque technique testée, vérifier l'apparition d'une alerte correspondante.
3. Relever l'identifiant de règle, le niveau de sévérité et l'horodatage pour mesurer le temps de détection.
4. Si absente : passer au module **Discover**, isoler le log brut correspondant, et remonter à l'écriture d'une règle custom.

---

## 07. Limites et améliorations

### Limites connues de l'architecture V1

| Limite | Impact | Mitigation / statut |
|---|---|---|
| Wazuh est un **HIDS**, pas un NIDS natif | Les scans réseau simples ne sont pas détectés sans Suricata | Suricata intégré — mitigé |
| Suricata et agent Wazuh colocalisés sur la machine cible | Un attaquant qui compromet la Victime peut désactiver la détection | Documenté comme limite de lab ; exploitable comme scénario d'évasion (T1562, Impair Defenses) pour une V2 |
| auditd non configuré en V1 | Command Injection / exécution post-exploitation moins visible | Axe d'amélioration prioritaire V1.1 |
| CSRF et Weak Session IDs | Signature réseau faible, angles morts probables d'un HIDS classique | Nécessiterait un WAF ou une corrélation applicative avancée — hors périmètre V1 |
| Dérive d'adressage IP (voir section 02) | Documentation et scripts à réajuster entre sessions | Fixer la plage NAT avant toute configuration réseau pour les prochaines itérations |
| Active Response non finalisé | Pas de blocage automatique démontré en V1 | Priorité pour une prochaine itération |
| Aucune mesure chiffrée de détection relevée | Taux de détection, temps de détection et score SCA non documentés | Axe prioritaire pour la suite : mener une campagne de test et consigner les résultats |

### Risques identifiés et mitigation

| Risque | Impact | Mitigation |
|---|---|---|
| Consommation RAM excessive si les 3 VM tournent en continu | Ralentissement du lab | Éteindre Kali hors des phases d'attaque |
| Dérive de périmètre (vouloir tout faire en V1) | Projet jamais terminé | Respecter strictement le hors-périmètre défini en section 01 |
| Oubli de l'isolation réseau | Risque légal/éthique en cas de fuite d'attaque | Vérifier la configuration réseau avant chaque session d'attaque |

### Perspectives V2

Plutôt qu'une simple extension de la V1, la V2 introduit une segmentation réseau réelle autour d'un firewall nouvelle génération (FortiGate), rapprochant le lab d'une architecture d'entreprise type DMZ/LAN :

```
                       [ INTERNET ]
                            |
                     (VMnet8 - NAT)
                      Port 1 (WAN)
                    +-------------------+
                    |   FORTIGATE VM    |  (L'agent de sécurité)
                    +-------------------+
                   /                     \
     DMZ (Port 3) /                       \ LAN (Port 2)
     VMnet3 (Host-Only)                   VMnet2 (Host-Only)
     Passerelle : 172.16.10.254           Passerelle : 10.0.10.254
                 |                                      |
                 |                        +-------------+-------------+
     [ Serveur Web (Ubuntu) ]             |                           |
     Site accessible au public     [ Client Linux ]          [ Client Windows ]
     IP : 172.16.10.20             Poste de l'entreprise     Poste de l'entreprise
                                   IP : 10.0.10.30           IP : 10.0.10.40

```

### Principes de cette évolution

1. **FortiGate comme point de passage obligé** entre Internet, la DMZ et le LAN, apportant une inspection de trafic (filtrage, NAT, politiques inter-zones) en amont de ce que Wazuh/Suricata observent côté hôte — complète la limite HIDS identifiée en V1.
2. **Séparation DMZ / LAN** : le serveur web public (cible exploitable, équivalent du rôle DVWA de la V1) est isolé en DMZ, tandis que les postes clients (Linux et Windows) rejoignent un réseau interne distinct — un attaquant qui compromet le serveur web ne se retrouve plus directement sur le même segment que les postes de l'entreprise, corrigeant la limite de colocalisation identifiée en V1.
3. **Extension à un client Windows**, hors périmètre de la V1, pour élargir la couverture de détection au-delà des seules machines Linux.
4. **Wazuh reste le SIEM central**, mais collecte désormais des logs et événements depuis des zones réseau cloisonnées par le FortiGate, ce qui permet de tester des scénarios de mouvement latéral (DMZ → LAN) et de valider que la segmentation elle-même est détectable et traçable.
5. **Politiques FortiGate comme nouvelle surface de detection engineering** : au même titre que les règles Wazuh custom en V1, les règles de pare-feu inter-zones devront être testées, affinées et documentées suite aux campagnes d'attaque.

---

## 08. Conclusion

Ce projet a permis de déployer de bout en bout un SOC miniature fonctionnel autour de Wazuh, intégrant une détection basée hôte (HIDS, FIM, SCA) et une détection réseau complémentaire (Suricata/NIDS), face à une cible Linux volontairement vulnérable (DVWA) pilotée via un arsenal offensif CLI reproductible (Kali).

La démarche définie (attaquer, mesurer dans le Dashboard, analyser les logs bruts en cas de non-détection, écrire une règle custom, re-tester) constitue le cœur de la pratique de **detection engineering**. Le projet a mis en évidence des limites réelles et documentées (HIDS vs NIDS, absence d'auditd, colocalisation du capteur réseau) plutôt que de masquer une couverture de détection idéalisée.

L'infrastructure, l'architecture et les 10 scénarios d'attaque mappés ATT&CK sont opérationnels et reproductibles. Aucune mesure chiffrée de détection n'a été relevée en V1 ; c'est un axe prioritaire pour la suite.

Les perspectives (auditd, Active Response finalisé, puis la V2 segmentée autour de FortiGate avec extension Windows) donnent une feuille de route claire pour approfondir ce lab au-delà de sa version V1 Linux.

---

## 09. Références

- Documentation officielle Wazuh : [https://documentation.wazuh.com](https://documentation.wazuh.com)
- MITRE ATT&CK : [https://attack.mitre.org](https://attack.mitre.org)
- DVWA (Damn Vulnerable Web Application) : [https://github.com/digininja/DVWA](https://github.com/digininja/DVWA)
- Suricata : [https://suricata.io](https://suricata.io)
- CIS Benchmarks : [https://www.cisecurity.org/cis-benchmarks](https://www.cisecurity.org/cis-benchmarks)

### Documentation détaillée du projet (Notion)

Le détail pas-à-pas de chaque étape (commandes exactes, fichiers de configuration, captures d'écran de validation) est tenu dans les pages Notion suivantes :

- [01 — Wazuh SOC | Cahier des charges et tableau de bord](https://scarce-lead-70d.notion.site/Wazuh-SOC-406baa6620a2809abda7efe0859e6758)
- [02 — Kali Linux | Machine de test](https://scarce-lead-70d.notion.site/Kali-Linux-406baa6620a28093bfd1c979a43e9c8d)
- [03 — Ubuntu Server | Machine cible](https://scarce-lead-70d.notion.site/Victime-Ubuntu-Server-406baa6620a2804f84a1f7e15e288887)
- [04 — Intégration et validation | Tests du laboratoire](https://scarce-lead-70d.notion.site/Int-gration-et-Validation-406baa6620a28077aaefe487b2b08b36)
- [05 — Rapport de Pentesting – Lab Wazuh/DVWA (Purple Team)](https://scarce-lead-70d.notion.site/Rapport-de-Pentesting-Lab-Wazuh-DVWA-409baa6620a281c9bc2ee773c6eeb68a?pvs=143)

---

## Structure du dépôt

```
06_SOC_Lab_Wazuh/
├── README.md                    # Ce document
├── architecture.md              # Schéma détaillé + justification des choix
├── rules/                       # Règles Wazuh custom (XML, commentées)
├── attack-scenarios/            # Scripts/commandes par scénario d'attaque
├── incident-reports/            # Rapports d'incident (Markdown)
└── evidence/                    # Captures d'écran et preuves (si présentes)
```

---

<p align="center"><i>Document évolutif. Environnement de laboratoire fermé et personnel, à but pédagogique (lab isolé, identifiants fictifs).</i></p>
