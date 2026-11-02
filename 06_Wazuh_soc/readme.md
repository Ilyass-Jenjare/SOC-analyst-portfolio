# SOC Lab Wazuh — V1 Linux

**Détection d'intrusion et validation par attaques contrôlées (Blue Team / Purple Team)**

![Status](https://img.shields.io/badge/status-en%20cours-yellow)
![OS](https://img.shields.io/badge/OS-Linux%20CLI%20only-blue)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%204.9-success)

> Projet personnel de cybersécurité mené en parallèle d'un encadrement en entreprise. Ce dépôt documente le déploiement d'un SOC miniature autour de Wazuh, la construction d'une cible Linux volontairement vulnérable, et l'émulation d'attaques réelles pour mesurer et améliorer la capacité de détection.

---

## Sommaire

1. [Introduction](#01-introduction)
2. [Architecture](#02-architecture)
3. [Infrastructure](#03-infrastructure)
4. [Implémentation](#04-implémentation)
5. [Scénarios de sécurité](#05-scénarios-de-sécurité)
6. [Détection et investigation](#06-détection-et-investigation)
7. [Résultats et évaluation](#07-résultats-et-évaluation)
8. [Limites et améliorations](#08-limites-et-améliorations)
9. [Conclusion](#09-conclusion)
10. [Références](#10-références)

---

## 01. Introduction

### Contexte

Ce projet a été réalisé dans le cadre d'un encadrement en entreprise, avec pour objectif de monter en compétences en détection d'intrusion (**Blue Team**) et de valider cette détection par des attaques contrôlées (**Purple Team**). L'environnement est volontairement limité à des machines Linux en ligne de commande (**CLI only**, sans interface graphique) afin de réduire la consommation de ressources sur un laptop de 16 Go de RAM (Katana B13.7).

### Objectifs

- Déployer un SOC miniature fonctionnel autour de **Wazuh** (SIEM/XDR open source).
- Construire une cible Linux volontairement vulnérable (DVWA), exploitable en CLI.
- Émuler des attaques réelles et mesurer leur détection.
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
| FIM, SCA, Active Response de base | Threat Intel feeds externes payants |

---

## 02. Architecture

### Schéma logique

```
=====================================================================
    RÉSEAU LOCAL UNIQUE : VMware NAT (isolé du réseau domestique)
=====================================================================

    +---------------+   +---------------+   +---------------+
    | Windows Hôte  |   | Serveur Linux |   |  Kali Linux   |
    | (PC Physique) |   |   (Victime)   |   |  (Attaquant)  |
    +-------+-------+   +-------+-------+   +-------+-------+
            │   ▲               │   ▲               │
 (Logs 1514)│   │    (Logs 1514)│   │   (Attaques / Scans / Exploits)
            │   │               │   │               │
            │   └───────────────────┴───────────────┘
            │                   │
            └─────────┐ ┌───────┘
                      │ │
                      ▼ ▼
        +-----------------------------------------+
        | Serveur Wazuh (Manager / Indexer /      |
        | Dashboard — stack all-in-one)           |
        +-----------------------------------------+
```

Flux de données : chaque attaque lancée depuis Kali touche la machine Victime (service SSH, application web DVWA). Les événements générés (`auth.log`, `access.log`, `eve.json` de Suricata, checks FIM/SCA) sont collectés par l'agent Wazuh installé sur la Victime, transmis au Manager sur le port **1514** (événements) et **1515** (enrôlement), corrélés par le moteur de règles, puis visualisés dans le Dashboard consulté depuis le navigateur de l'hôte Windows.

### Choix « CLI only »

- Les trois VM (Wazuh, Kali, Victime) sont installées en mode **serveur / sans interface graphique**.
- Administration exclusivement en **SSH** depuis le terminal de l'hôte.
- Le dashboard Wazuh (interface web) est consulté depuis le navigateur de l'hôte Windows en pointant vers l'IP de la VM Wazuh — aucune VM n'a besoin d'environnement graphique pour cela.

### Adressage réseau — évolution du lab

> **Note de transparence :** l'adressage IP du lab a évolué au fil des sessions de travail, le réseau NAT de VMware Workstation ayant été réattribué à plusieurs reprises entre les phases de déploiement. Le tableau ci-dessous reflète les trois plages successivement utilisées. Cette dérive d'adressage est elle-même documentée comme point de vigilance en [section 08](#08-limites-et-améliorations).

| Phase | Plage NAT | Wazuh Manager | Victime | Kali |
|---|---|---|---|---|
| Cahier des charges initial | `192.168.56.0/24` | `192.168.56.100` | `192.168.56.150` | `192.168.56.200` |
| Déploiement M2/M3 (Kali + Victime) | `192.168.48.0/24` | `192.168.48.133` | `192.168.48.150` | `192.168.48.200` |
| Campagne de pentest (rapport final) | `192.168.111.0/24` | `192.168.111.x` | `192.168.111.150` | `192.168.111.x` |

Pour toute reproduction du lab, fixer la plage NAT **avant** la configuration Netplan de chaque VM et la garder stable sur toute la durée du projet évite cette dérive.

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
| Wazuh Agent (v4.9.2) | M3 | Collecte logs, FIM, SCA |
| OpenSSH Server | M3 | Service exposé, cible de brute force |
| Apache2 + PHP + MariaDB | M3 | Serveur web applicatif |
| DVWA | M3 | Application web volontairement vulnérable |
| Suricata | M3 | Détection réseau (NIDS), écrit dans `/var/log/suricata/eve.json` |
| nmap, hydra, sqlmap, curl, netcat, Wireshark | M2 | Arsenal offensif CLI |

### Gestion des ressources

- Hyperviseur : **VMware Workstation Pro**.
- Snapshots de référence pris à chaque jalon stable : `01-Wazuh-Ready`, `01-Kali-Ready`, `01-Victime-Ready`, puis snapshot global `02-Lab-Operational`.
- Bonne pratique appliquée : **Kali reste éteinte** en dehors des phases d'attaque pour limiter la consommation RAM continue sur le laptop hôte.

---

## 04. Implémentation

Cette section documente les étapes réellement exécutées pour déployer chaque brique du lab, avec les commandes utilisées.

### 4.1 Machine 1 — Serveur SIEM Wazuh

**Étape 1 — Déploiement Ubuntu Server et réseau statique**

VM créée avec 4 Go RAM, 2 vCPU, 40 Go disque, réseau NAT ; installation Ubuntu Server minimale avec l'option *Install OpenSSH server* activée dès l'installeur pour permettre l'administration distante immédiate.

Configuration Netplan (`/etc/netplan/...`) :

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.56.100/24]
      gateway4: 192.168.56.2
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
ip -br a show ens33
```

Vérifications :

```bash
ping -c 3 192.168.56.1      # passerelle
ping -c 3 google.com        # accès Internet
```

**Critère de validation :** SSH accessible depuis l'hôte (`ssh wazuh@<ip>`), IP statique persistante après redémarrage.

**Étape 2 — Installation de Wazuh (Manager + Indexer + Dashboard)**

Déploiement en mode **all-in-one** via le script d'installation assistée officiel :

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Téléchargement du script d'installation assistée
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh

# Installation complète (Indexer + Manager + Dashboard + certificats TLS)
sudo bash wazuh-install.sh -a

# Récupération du mot de passe admin généré
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt | grep -A1 "'admin'"
```

Vérification des trois services :

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo ss -tulnp | grep 443
```

**Résultat obtenu** (extrait du journal d'installation) :

```
INFO: --- Summary ---
INFO: You can access the web interface https://<wazuh-dashboard-ip>:443
User: admin
Password: <généré automatiquement, stocké en gestionnaire de mots de passe>
INFO: Installation finished.
```

> Le mot de passe admin généré n'est jamais commité sur GitHub ; il est conservé dans un gestionnaire de mots de passe.

**Étape 3 — Durcissement de base**

```bash
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp
sudo ufw enable
sudo ufw status verbose
```

Validation croisée depuis Kali : un scan `nmap` sur l'IP du Manager doit confirmer que seuls ces quatre ports apparaissent ouverts.

### 4.2 Machine 2 — Poste offensif Kali Linux

Image officielle VMware importée, RAM bridée à 2 Go / 2 vCPU, réseau NAT sur le même sous-réseau que le reste du lab.

Adressage statique :

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.48.200/24 \
  ipv4.gateway 192.168.48.2 \
  ipv4.dns "8.8.8.8,1.1.1.1"

sudo nmcli connection down "Wired connection 1" && sudo nmcli connection up "Wired connection 1"
```

Vérification :

```bash
ip -br a show eth0
ping -c 3 192.168.48.133   # Serveur SIEM
```

**Préparation de l'arsenal offensif** — vérification de la disponibilité des outils, et script de reconnaissance/brute force réutilisable :

```bash
which nmap hydra sqlmap nc wireshark curl
mkdir -p ~/lab-attacks/pcaps ~/lab-attacks/scripts
```

`~/lab-attacks/scripts/recon_and_bruteforce.sh` :

```bash
#!/bin/bash
TARGET="192.168.56.150"

echo "[*] Phase 1 : Scan de ports Nmap (déclenche détection Suricata)..."
nmap -sS -T4 -A -p 1-1000 $TARGET

echo "[*] Phase 2 : Brute force SSH avec Hydra (déclenche détection Wazuh/auth.log)..."
hydra -l victimuser -P ~/lab-attacks/short-pass.txt ssh://$TARGET -t 4
```

```bash
chmod +x ~/lab-attacks/scripts/recon_and_bruteforce.sh
```

### 4.3 Machine 3 — Cible vulnérable (SSH + Web + agent)

**Réseau statique `.150`** (même méthode Netplan qu'en 4.1), puis vérification de la joignabilité bidirectionnelle avec M1 et M2.

**Installation d'Apache, PHP, MariaDB et déploiement de DVWA :**

```bash
sudo apt install -y apache2 php php-mysqli php-gd libapache2-mod-php mariadb-server git

cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git dvwa
sudo chown -R www-data:www-data /var/www/html/dvwa
sudo chmod -R 755 /var/www/html/dvwa
```

Base de données dédiée :

```bash
sudo mysql -u root -e "
CREATE DATABASE dvwa;
CREATE USER 'dvwauser'@'localhost' IDENTIFIED BY 'P@ssw0rdDVWA';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwauser'@'localhost';
FLUSH PRIVILEGES;"
```

Configuration DVWA :

```bash
cd /var/www/html/dvwa/config
sudo cp config.inc.php.dist config.inc.php
sudo sed -i "s/'db_user'     ] = '.*';/'db_user'     ] = 'dvwauser';/" config.inc.php
sudo sed -i "s/'db_password' ] = '.*';/'db_password' ] = 'P@ssw0rdDVWA';/" config.inc.php
sudo sed -i "s/'db_database' ] = '.*';/'db_database' ] = 'dvwa';/" config.inc.php
sudo systemctl restart apache2
```

Vérification : `curl -I http://<ip-victime>/dvwa/login.php` doit retourner `HTTP 200`.

> Identifiants DVWA par défaut (`admin` / `password`) explicitement fictifs — à changer si le lab est exposé au-delà d'une session de test locale.

**Installation de Suricata (NIDS) :**

```bash
sudo apt install -y suricata
sudo nano /etc/suricata/suricata.yaml
```

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.48.0/24]"
af-packet:
  - interface: ens33
```

```bash
sudo suricata-update
sudo systemctl enable suricata
sudo systemctl restart suricata
```

Suricata écrit ses alertes dans `/var/log/suricata/eve.json`, lu ensuite par l'agent Wazuh pour corréler les détections réseau avec les détections système. Validé par un scan `nmap -sS` de test depuis Kali générant une entrée dans `eve.json`.

**Installation et enrôlement de l'agent Wazuh :**

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo WAZUH_MANAGER='192.168.48.133' apt install -y wazuh-agent
```

Ajout de la lecture du log Suricata dans `/var/ossec/etc/ossec.conf` :

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

**Activation FIM et SCA** (bloc `<syscheck>` dans `ossec.conf`) :

```xml
<syscheck>
  <directories check_all="yes" realtime="yes">/var/www/html/dvwa</directories>
  <directories check_all="yes">/etc</directories>
</syscheck>

<sca>
  <enabled>yes</enabled>
  <scan_on_start>yes</scan_on_start>
  <interval>12h</interval>
</sca>
```

```bash
sudo systemctl restart wazuh-agent
```

Test manuel de déclenchement FIM :

```bash
sudo touch /var/www/html/dvwa/hackable/uploads/test.php
```

### 4.4 Incidents de configuration rencontrés et résolutions

Trois incidents réels rencontrés pendant l'enrôlement de l'agent, conservés ici à titre de retour d'expérience technique :

**Erreur d'assignation de groupe (`wazuh-db`) :**

```bash
sudo chown -R wazuh:wazuh /var/ossec/queue/db/
sudo chown -R wazuh:wazuh /var/ossec/var/multigroups/
sudo /var/ossec/bin/agent_groups -a -i 002 -g default
```

**Espace disque saturé (100 % sur `/`) :**

```bash
sudo apt clean
sudo apt autoremove -y
sudo journalctl --vacuum-time=1d
sudo rm -rf /var/ossec/tmp/*
df -h /
```

**Crash de l'API Wazuh / processus zombies (`wazuh-apid did not start`) :**

```bash
sudo fuser -k 55000/tcp
sudo pkill -f wazuh
sudo rm -f /var/ossec/var/run/*.pid
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
sudo /var/ossec/bin/agent_control -i 002
```

---

## 05. Scénarios de sécurité

### Méthodologie Purple Team

Chaque scénario suit le même cycle de detection engineering :

1. **Préparation** — définir la technique ATT&CK visée et le résultat attendu en détection.
2. **Exécution de l'attaque** depuis Kali (CLI).
3. **Vérification** dans le dashboard Wazuh (`Security Events`, `Threat Hunting`).
4. **Si non détecté** — analyse du log brut, écriture d'une règle/décodeur custom.
5. **Re-test** de la même attaque pour valider la nouvelle règle.
6. **Documentation** — mise à jour de la matrice ATT&CK et rédaction du rapport d'incident.

### Règles d'engagement (Rules of Engagement)

- **Cible autorisée :** uniquement la machine Victime du lab.
- **Hors périmètre :** la machine hôte Windows (non testée).
- **Réseau :** environnement isolé/NAT interne, aucune exposition Internet.
- **Objectif :** détection, pas exploitation destructive (pas de DoS, pas de persistance destructrice).
- **Niveau DVWA Security :** tests idéalement menés à plusieurs niveaux (`low`, `medium`, `high`) pour observer si l'obfuscation de la requête échappe aux règles par défaut.

### Scénarios testés, mappés MITRE ATT&CK

| # | Technique | Module DVWA / cible | MITRE ATT&CK | Outil / commande |
|---|---|---|---|---|
| 1 | Reconnaissance réseau | Hôte Victime | T1046 | `nmap -sS -sV -O -p- <ip>` |
| 2 | Brute force SSH | Service OpenSSH | T1110.001 | `hydra -l victimuser -P wordlist.txt ssh://<ip>` |
| 3 | Brute force applicatif | DVWA — Brute Force | T1110.001 | `hydra ... http-get-form "/dvwa/vulnerabilities/brute/..."` |
| 4 | Injection SQL | DVWA — SQL Injection | T1190 | `sqlmap -u "http://<ip>/dvwa/vulnerabilities/sqli/?id=1" --batch --dbs` |
| 5 | Command Injection | DVWA — Command Injection | T1059.004 / T1190 | Payloads `127.0.0.1 && id`, `; whoami`, `\| cat /etc/passwd` |
| 6 | Cross-Site Scripting | DVWA — XSS (Reflected/Stored/DOM) | T1059.007 / T1189 | `<script>alert(document.cookie)</script>` |
| 7 | Dépôt de webshell | DVWA — File Upload | T1505.003 / T1105 | Upload PHP minimal `system($_GET["cmd"])` |
| 8 | Local File Inclusion | DVWA — File Inclusion | T1083 | `curl ".../fi/?page=../../../../etc/passwd"` |
| 9 | CSRF | DVWA — CSRF | T1185 | Requête forgée hors session légitime |
| 10 | Session IDs faibles | DVWA — Weak Session IDs | T1539 | Analyse de prévisibilité des session IDs |

### Exemple de script d'attaque automatisé

Le script `recon_and_bruteforce.sh` (section 4.2) enchaîne un scan de reconnaissance et un brute force SSH en une seule exécution, pour reproduire de façon identique le même scénario d'une session de test à l'autre — condition nécessaire pour comparer objectivement le taux de détection avant/après l'écriture de règles custom.

---

## 06. Détection et investigation

### Sources de données corrélées par Wazuh

| Source | Collectée via | Donne de la visibilité sur |
|---|---|---|
| `/var/log/auth.log` | Agent Wazuh (décodeur natif) | Brute force SSH, connexions réussies/échouées |
| `/var/log/apache2/access.log` | Agent Wazuh (decoder `apache-access`) | Requêtes web suspectes (SQLi, XSS, LFI) |
| `/var/log/suricata/eve.json` | Agent Wazuh (`<localfile log_format="json">`) | Scans de ports, signatures réseau (NIDS) |
| Syscheck (FIM) | Agent Wazuh, temps réel sur `dvwa/` et `/etc` | Dépôt de fichiers (ex. webshell) |
| SCA | Agent Wazuh, scan toutes les 12h | Niveau de durcissement vs. CIS Benchmark |
| auditd (recommandé, non actif en V1) | Règle `execve` sur `www-data` | Exécution de commandes post-exploitation |

### Détection attendue par scénario

- **Reconnaissance réseau (T1046) :** point de vigilance documenté — Wazuh est un **HIDS**, pas un NIDS. Sans Suricata, un simple `nmap` n'est **pas** détecté nativement. Suricata comble cette lacune en écrivant dans `eve.json`, lu par l'agent.
- **Brute force SSH / applicatif (T1110) :** couvert par les décodeurs natifs Wazuh sur `auth.log` et `access.log` (groupe `web`, `attack`, règles ~31100+).
- **Injection SQL (T1190) :** détection native possible si le decoder web détecte des patterns (`UNION SELECT`, `OR 1=1`) ; sinon nécessite une règle custom (voir ci-dessous).
- **Command Injection (T1059) :** détection fiable uniquement si **auditd** est configuré sur l'agent pour tracer les `execve` du processus `www-data` — non actif en V1, identifié comme axe d'amélioration.
- **Webshell / File Upload (T1505.003) :** scénario de démonstration du **FIM temps réel** — un fichier `.php` déposé dans `hackable/uploads/` doit déclencher une alerte `syscheck` en quelques minutes.
- **XSS (T1059.007) :** généralement couvert par les règles web par défaut (groupe `xss`) si le decoder `apache-access` est actif.
- **LFI (T1083) :** pattern `../` / `etc/passwd` dans les logs web, couvert si le décodeur web est actif.
- **CSRF et Weak Session IDs :** signature réseau faible, documentés comme **angles morts probables** d'une détection HIDS classique sans WAF applicatif dédié.

### Exemple de règle custom (si détection native absente)

Règle proposée pour couvrir les patterns SQLi non détectés nativement, à placer dans `local_rules.xml` :

```xml
<rule id="100100" level="10">
  <if_sid>31100</if_sid>
  <url>union select|information_schema|sleep\(|benchmark\(</url>
  <description>Possible SQL Injection attempt detected in web logs</description>
  <mitre>
    <id>T1190</id>
  </mitre>
</rule>
```

> D'autres règles custom (brute force applicatif avec seuil, détection de dépôt de webshell par extension, détection de payloads LFI) sont à documenter au fur et à mesure dans `rules/`, chacune avec : objectif, logique, log brut déclencheur, et commande de re-test.

### Active Response

Objectif V1 : blocage automatique d'une IP source après dépassement d'un seuil de tentatives de brute force SSH, via `firewall-drop` (intégration UFW). Statut : **à finaliser** — voir [section 08](#08-limites-et-améliorations).

### Processus d'investigation dans le Dashboard

1. Module **Security Events** → filtrer par `agent.ip: <ip-victime>`.
2. Pour chaque technique testée, vérifier l'apparition d'une alerte correspondante.
3. Relever le `rule.id`, le niveau de sévérité, et le timestamp (`@timestamp`) pour mesurer le temps de détection.
4. Si absente : passer au module **Discover**, isoler le log brut correspondant, et remonter à l'écriture d'une règle custom.

---

## 07. Résultats et évaluation

> Cette section est **structurée pour être complétée au fil des campagnes de test**. Le tableau et les indicateurs ci-dessous constituent le gabarit de mesure du projet ; les valeurs réelles seront renseignées après chaque session d'attaque contre le Dashboard Wazuh.

### Tableau de résultats (gabarit de mesure)

| # | Technique | Niveau DVWA | Détecté nativement ? | Rule ID Wazuh | Sévérité | Temps de détection | Règle custom nécessaire ? |
|---|---|---|---|---|---|---|---|
| 1 | Scan réseau (nmap) | — | | | | | |
| 2 | Brute force SSH | — | | | | | |
| 3 | Brute force applicatif | low | | | | | |
| 4 | Injection SQL | low | | | | | |
| 5 | Command Injection | low | | | | | |
| 6 | XSS Reflected | low | | | | | |
| 7 | File Upload (webshell) | low | | | | | |
| 8 | LFI | low | | | | | |
| 9 | CSRF | low | | | | | |
| 10 | Weak Session IDs | low | | | | | |

### Indicateurs de performance (KPI)

| Critère | Objectif cible | Résultat actuel |
|---|---|---|
| Techniques ATT&CK testées | Au moins 8 techniques distinctes | 10 scénarios définis (section 05) |
| Taux de détection initial | Mesuré et documenté (même faible) | *à compléter après campagne de test* |
| Taux de détection après règles custom | Amélioration démontrable (ex. +40 points) | *à compléter* |
| Règles custom créées | Au moins 5, documentées et testées | 1 exemple fourni (SQLi) — *à compléter* |
| Active Response fonctionnel | Blocage automatique validé sur au moins 1 scénario | *non finalisé* |
| Documentation | README et rapports complets, compréhensibles par un tiers | Ce document |

### Matrice de couverture ATT&CK

À tenir à jour dans `attack-coverage.md` au format :

| Tactique | Technique | ID | Testée | Détectée | Règle associée |
|---|---|---|---|---|---|
| Discovery | Network Service Discovery | T1046 | ✅ | | |
| Credential Access | Brute Force | T1110.001 | ✅ | | |
| Initial Access | Exploit Public-Facing App | T1190 | ✅ | | |
| Execution | Unix Shell | T1059.004 | ✅ | | |
| Persistence | Web Shell | T1505.003 | ✅ | | |
| Discovery | File and Directory Discovery | T1083 | ✅ | | |

### Score SCA (Security Configuration Assessment)

Module Wazuh → **Security Configuration Assessment** : pourcentage de conformité au CIS Benchmark Linux à relever ici après premier scan complet de la Victime.

---

## 08. Limites et améliorations

### Limites connues de l'architecture V1

| Limite | Impact | Mitigation / statut |
|---|---|---|
| Wazuh est un **HIDS**, pas un NIDS natif | Les scans réseau simples ne sont pas détectés sans Suricata | Suricata intégré et lu via `eve.json` — mitigé |
| Suricata et agent Wazuh colocalisés sur la machine cible | Un attaquant qui compromet la Victime peut désactiver la détection | Documenté comme limite de lab ; exploitable comme scénario d'évasion (T1562, Impair Defenses) pour une V2 |
| auditd non configuré en V1 | Command Injection / exécution post-exploitation moins visible | Axe d'amélioration prioritaire V1.1 |
| CSRF et Weak Session IDs | Signature réseau faible, angles morts probables d'un HIDS classique | Nécessiterait un WAF ou une corrélation applicative avancée — hors périmètre V1 |
| Dérive d'adressage IP (voir section 02) | Documentation et scripts à réajuster entre sessions | Fixer la plage NAT avant toute configuration Netplan pour les prochaines itérations |
| Active Response non finalisé | Pas de blocage automatique démontré en V1 | Priorité pour clôturer les KPI de la section 07 |

### Risques identifiés et mitigation

| Risque | Impact | Mitigation |
|---|---|---|
| Consommation RAM excessive si les 3 VM tournent en continu | Ralentissement du lab | Éteindre Kali hors des phases d'attaque |
| Dérive de périmètre (vouloir tout faire en V1) | Projet jamais terminé | Respecter strictement le hors-périmètre défini en section 01 |
| Oubli de l'isolation réseau | Risque légal/éthique en cas de fuite d'attaque | Vérifier la configuration réseau avant chaque session d'attaque |

### Pistes d'amélioration pour une V2

1. Activer **auditd** sur la Victime pour tracer l'exécution de commandes par `www-data` (amélioration directe de la détection Command Injection).
2. Finaliser l'**Active Response** (blocage automatique via `firewall-drop`/UFW après seuil de brute force).
3. Tester les scénarios DVWA aux niveaux `medium` et `high` pour mesurer la résistance des règles face à des payloads obfusqués.
4. Étendre la cible Windows (hors périmètre V1, déjà évoquée comme agent potentiel 003).
5. Ajouter un second capteur réseau indépendant de la machine cible pour corriger la limite de colocalisation Suricata/agent.
6. Automatiser le déploiement du lab via scripts bash versionnés pour en garantir la reproductibilité totale.

---

## 09. Conclusion

Ce projet a permis de déployer de bout en bout un SOC miniature fonctionnel autour de Wazuh, intégrant une détection basée hôte (HIDS, FIM, SCA) et une détection réseau complémentaire (Suricata/NIDS), face à une cible Linux volontairement vulnérable (DVWA) pilotée via un arsenal offensif CLI reproductible (Kali, scripts bash).

La démarche suivie — attaquer, mesurer dans le Dashboard, analyser les logs bruts en cas de non-détection, écrire une règle custom, re-tester — constitue le cœur de la pratique de **detection engineering**, et a mis en évidence des limites réelles et documentées (HIDS vs NIDS, absence d'auditd, colocalisation du capteur réseau) plutôt que de masquer une couverture de détection idéalisée.

Les résultats chiffrés définitifs (taux de détection initial/après règles, score SCA, couverture MITRE ATT&CK complète) restent à consolider lors de la campagne de test finale (section 07), mais la méthodologie, l'architecture et les 10 scénarios d'attaque mappés ATT&CK sont d'ores et déjà opérationnels et reproductibles.

Les perspectives V2 (auditd, Active Response finalisé, capteur réseau indépendant, extension Windows) donnent une feuille de route claire pour approfondir ce lab au-delà de sa version V1 Linux.

---

## 10. Références

- Documentation officielle Wazuh : [https://documentation.wazuh.com](https://documentation.wazuh.com)
- MITRE ATT&CK : [https://attack.mitre.org](https://attack.mitre.org)
- DVWA (Damn Vulnerable Web Application) : [https://github.com/digininja/DVWA](https://github.com/digininja/DVWA)
- Suricata : [https://suricata.io](https://suricata.io)
- CIS Benchmarks : [https://www.cisecurity.org/cis-benchmarks](https://www.cisecurity.org/cis-benchmarks)

### Documentation détaillée du projet (Notion)

- [01 — Wazuh SOC | Cahier des charges et tableau de bord](https://scarce-lead-70d.notion.site/Wazuh-SOC-406baa6620a2809abda7efe0859e6758)
- [02 — Kali Linux | Machine de test](https://scarce-lead-70d.notion.site/Kali-Linux-406baa6620a28093bfd1c979a43e9c8d)
- [03 — Ubuntu Server | Machine cible](https://scarce-lead-70d.notion.site/Victime-Ubuntu-Server-406baa6620a2804f84a1f7e15e288887)
- [04 — Intégration et validation | Tests du laboratoire](https://scarce-lead-70d.notion.site/Int-gration-et-Validation-406baa6620a28077aaefe487b2b08b36)
- Rapport de Pentesting – Lab Wazuh/DVWA (Purple Team)

---

## Structure du dépôt

```
06_SOC_Lab_Wazuh/
├── README.md                    # Ce document
├── architecture.md              # Schéma détaillé + justification des choix
├── rules/                       # Règles Wazuh custom (XML, commentées)
├── attack-scenarios/            # Scripts/commandes par scénario d'attaque
├── incident-reports/            # Rapports d'incident (Markdown)
├── attack-coverage.md           # Matrice de couverture MITRE ATT&CK
└── evidence/                    # Captures d'écran et preuves (alertes, scores SCA, etc.)
```

---

<p align="center"><i>Document évolutif — à compléter au fil du projet avec les résultats réels, les captures d'écran de preuve et les retours d'expérience. Environnement de laboratoire fermé et personnel, à but pédagogique (lab isolé, identifiants fictifs).</i></p>
