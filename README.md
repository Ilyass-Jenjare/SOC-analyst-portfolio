# Portfolio SOC Analyst & Cybersécurité

Ce dépôt rassemble mes projets pratiques en cybersécurité, démontrant mes compétences opérationnelles (Blue Team) pour des rôles d'Analyste SOC. Chaque projet simule des tâches réelles d'investigation, de détection et de réponse aux incidents, en s'appuyant sur des standards de l'industrie (MITRE ATT&CK, NIST, SANS).

## Objectifs du Portfolio

- Analyser et qualifier des menaces réelles (Phishing, Malwares, Trafic réseau suspect).
- Concevoir et déployer des architectures de détection (SIEM, NIDS, Firewalls).
- Exploiter les outils standards de l'industrie (Wireshark, Wazuh, FortiGate, OSINT).
- Produire des rapports techniques et exécutifs actionnables.

---

## Projets Réalisés

### 1. Analyse de Phishing (`01_Phishing_Analysis`)
**Objectif :** Décortiquer des campagnes de phishing réelles en toute sécurité.
- **Compétences :** Analyse d'en-têtes SMTP (SPF/DKIM/DMARC), extraction d'URLs/fichiers sans détonation, recherche OSINT.
- **Outils :** MXToolbox, VirusTotal, urlscan.io, ANY.RUN.
- **Livrables :** Rapport d'analyse, liste d'IOCs (CSV), mapping MITRE ATT&CK.

### 2. Investigation Réseau & Analyse de PCAP (`02_Attack_Investigation`)
**Objectif :** Identifier une intrusion et le comportement d'un malware à partir d'une capture réseau (Scénario Boogeyman).
- **Compétences :** Filtrage de paquets, extraction d'objets HTTP, identification de trafic C2 (Beaconing), analyse d'exfiltration.
- **Outils :** Wireshark, NetworkMiner, CyberChef.
- **Livrables :** Timeline de l'attaque, récit d'investigation, extraction de payloads.

### 3. SOC Lab — Détection & Validation avec Wazuh (`03_Wazuh_SOC`)
**Objectif :** Déployer un SOC miniature (Zero Trust, SIEM Wazuh, Pare-feu FortiGate), émuler des attaques réelles et développer des règles de détection personnalisées.
- **Compétences :** Administration Wazuh (Manager/Indexer/Dashboard), analyse de logs (Syslog, auth.log, Apache), détection réseau (NIDS), configuration de pare-feu (DoS Policy, Early Drop), cartographie MITRE ATT&CK.
- **Outils :** Wazuh, FortiGate (FortiOS), Suricata, DVWA, Kali Linux (nmap, Hydra, sqlmap).
- **Livrables :** Schéma d'architecture réseau, documentation de déploiement, rapports d'incident par scénario d'attaque.

---

## Boîte à Outils & Frameworks

- **Analyse & Investigation :** Wireshark, NetworkMiner, CyberChef.
- **Threat Intelligence & OSINT :** VirusTotal, urlscan.io, ANY.RUN, MXToolbox.
- **SIEM & Sécurité Périmétrique :** Wazuh, FortiGate, Suricata.
- **Tests d'intrusion & Émulation :** Kali Linux (nmap, Hydra, sqlmap), DVWA.
- **Cadres de Référence :** MITRE ATT&CK, NIST SP 800-61, SANS Incident Handling.

---

## Auteur

**Ilyass Jenjare**
- **LinkedIn :** [https://www.linkedin.com/in/ilyass-jenjare](https://www.linkedin.com/in/ilyass-jenjare)
- **Contact :** Ilyas.Jenjare@Usherbrooke.ca
