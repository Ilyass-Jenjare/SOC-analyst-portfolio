# Portfolio SOC Analyst & Cybersécurité

Ce dépôt documente un sprint intensif de 10 jours visant à démontrer mes compétences opérationnelles (Blue Team) pour des rôles d'Analyste SOC et de spécialiste en cybersécurité. Chaque projet simule des tâches réelles d'investigation, de triage et de réponse aux incidents, en s'appuyant sur des standards de l'industrie (MITRE ATT&CK, NIST, SANS).

## Objectifs du Portfolio
- Analyser et qualifier des menaces réelles (Phishing, Malwares, Trafic réseau suspect).
- Appliquer les méthodologies de réponse à incident et documenter les investigations.
- Exploiter les outils standards de l'industrie (Wireshark, SIEM, Sandbox, OSINT).
- Produire des rapports techniques et exécutifs actionnables.

---

## Projets Réalisés

### 1. Analyse de Phishing
**Objectif :** Décortiquer des campagnes de phishing réelles en toute sécurité.
- **Compétences :** Analyse d'en-têtes SMTP (SPF/DKIM/DMARC), extraction d'URLs/fichiers sans détonation, recherche OSINT.
- **Outils :** MXToolbox, VirusTotal, urlscan.io, ANY.RUN.
- **Livrables :** Rapport d'analyse, liste d'IOCs (CSV), mapping MITRE ATT&CK.

### 2. Investigation Réseau & Analyse de PCAP ( boogeyman1 )
**Objectif :** Identifier une intrusion et le comportement d'un malware à partir d'une capture réseau.
- **Compétences :** Filtrage de paquets, extraction d'objets HTTP, identification de trafic C2 (Beaconing), exfiltration.
- **Outils :** Wireshark, NetworkMiner.
- **Livrables :** Timeline de l'attaque, récit d'investigation, extraction de payloads.

### 3. Réponse à Incident ( plannifié )
**Objectif :** Résoudre un scénario complet d'incident et rédiger un rapport formel.
- **Compétences :** Triage, confinement, éradication, application des frameworks d'intervention.
- **Cadre :** SANS PICERL / NIST SP 800-61.
- **Livrables :** Rapport d'incident complet incluant les leçons apprises et les recommandations de durcissement.

### 4. Rapport de Cyber Threat Intelligence (CTI) ( plannifié )
**Objectif :** Profiler un acteur de la menace (ex. Scattered Spider) et analyser ses TTPs.
- **Compétences :** Recherche OSINT, modélisation des menaces, création de matrices.
- **Outils :** MITRE ATT&CK Navigator.
- **Livrables :** Brief exécutif (BLUF), matrice tactique, recommandations de détection.

### 5. Laboratoire Home SIEM & Règles de Détection ( en cours )
**Objectif :** Déployer une infrastructure de journalisation et créer des alertes personnalisées.
- **Compétences :** Déploiement de serveurs, centralisation de logs, requêtes de détection (SPL/KQL/Sigma).
- **Outils :** Splunk Enterprise / Elastic Stack, Sysmon, Windows/Linux VM, Atomic Red Team.
- **Livrables :** Schéma d'architecture, write-up d'investigation d'une alerte, fichiers de règles Sigma/SPL.

### 6. SOC Lab — Détection & Validation avec Wazuh ( en cours )
**Objectif :** Déployer un SOC miniature basé sur Wazuh (cible Linux), émuler des attaques réelles et développer des règles de détection personnalisées pour mesurer et améliorer la couverture de détection.
- **Compétences :** Administration Wazuh (Manager/Indexer/Dashboard), analyse de logs Linux (auth.log, logs Apache), détection réseau (NIDS), écriture de règles/décodeurs custom, cartographie MITRE ATT&CK, Active Response.
- **Outils :** Wazuh, Suricata, DVWA, OpenSSH, Apache, Kali Linux (nmap, Hydra, sqlmap), MITRE ATT&CK.
- **Livrables :** Schéma d'architecture, règles de détection custom documentées, matrice de couverture MITRE ATT&CK, rapports d'incident par scénario d'attaque.

---

## Boîte à Outils & Frameworks
- **Analyse & Investigation :** Wireshark, NetworkMiner, Sysmon.
- **Threat Intelligence & OSINT :** VirusTotal, urlscan.io, PhishTank, ANY.RUN.
- **SIEM & Log Management :** Splunk / Elastic Stack, Wazuh.
- **Détection réseau & Tests d'intrusion :** Suricata, Kali Linux (nmap, Hydra, sqlmap), DVWA.
- **Cadres de Référence :** MITRE ATT&CK, NIST SP 800-61, SANS Incident Handling.

---

## Auteur
**Ilyass Jenjare**
- **LinkedIn :** https://www.linkedin.com/in/ilyass-jenjare
- **Contact :** Ilyas.Jenjare@Usherbrooke.ca
