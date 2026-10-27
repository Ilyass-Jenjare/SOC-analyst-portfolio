# SOC Lab Wazuh — V1 Linux (CLI Edition)

Cahier des charges — Projet personnel cybersécurité (Blue Team / Purple Team)

---

## 1. Contexte et objectifs

### 1.1 Contexte
Projet personnel réalisé dans le cadre d'un encadrement en entreprise, visant à monter en compétences en détection d'intrusion (Blue Team) et à valider cette détection par des attaques contrôlées (Purple Team). L'environnement est volontairement limité à des machines Linux en ligne de commande (CLI only, sans interface graphique) afin de réduire la consommation de ressources sur un laptop de 16 Go de RAM.

### 1.2 Objectifs du projet
- Déployer un SOC miniature fonctionnel autour de **Wazuh** (SIEM/XDR open source).
- Construire une cible Linux volontairement vulnérable, exploitable en CLI.
- Émuler des attaques réelles et mesurer leur détection.
- Écrire des règles de détection personnalisées pour combler les lacunes identifiées.
- Produire une documentation et des livrables exploitables pour un CV / portfolio cybersécurité (candidature SOC, alternance, stage).

### 1.3 Objectifs pédagogiques
- Maîtriser le déploiement et l'administration de Wazuh (Manager, Indexer, agents).
- Comprendre et exploiter les logs systèmes Linux (`auth.log`, `syslog`, logs applicatifs).
- Savoir cartographier des attaques sur le référentiel **MITRE ATT&CK**.
- Développer un réflexe de détection engineering : attaquer, mesurer, corriger, revalider.

### 1.4 Hors périmètre (V1)
- Pas de machine Windows (prévue en V2).
- Pas d'interface graphique sur les machines victimes (tout en CLI).
- Pas de haute disponibilité / cluster Wazuh (hors sujet pour un lab).
- Pas d'intégration SOAR complète (TheHive/Shuffle), envisagée uniquement en bonus.

---

## 2. Contraintes

### 2.1 Contraintes matérielles
| Élément | Valeur |
|---|---|
| Laptop | Katana B13.7 |
| RAM totale | 16 Go |
| Hyperviseur | VirtualBox ou VMware Workstation |
| Contrainte | Pas d'interface graphique sur les VM victimes/attaquant pour limiter la RAM |

### 2.2 Contraintes de sécurité
- Le lab doit être **totalement isolé** du réseau domestique (réseau host-only ou interne à l'hyperviseur).
- Aucune attaque ne doit sortir du périmètre du lab.
- Les identifiants/mots de passe utilisés dans le lab doivent être clairement identifiés comme fictifs dans la documentation.

### 2.3 Contraintes de temps
- Durée estimée : 3 semaines (phase V1 Linux uniquement).
- Projet mené en parallèle d'une activité encadrée en entreprise : les créneaux de travail doivent être compatibles avec un rythme soirs/week-ends.

---

## 3. Architecture technique

### 3.1 Schéma

```
                 Réseau host-only isolé (ex: 192.168.56.0/24)
 ┌──────────────────────────────────────────────────────────────────┐
 │                                                                    │
 │   ┌───────────────┐      attaques CLI      ┌────────────────────┐ │
 │   │ Kali Linux     │ ─────────────────────► │ Ubuntu Server      │ │
 │   │ (CLI only)     │                        │ (CLI only)         │ │
 │   │ nmap, hydra,   │                        │ SSH, Apache+DVWA,  │ │
 │   │ sqlmap, curl   │                        │ agent Wazuh,       │ │
 │   │                │                        │ Suricata           │ │
 │   └───────────────┘                        └──────────┬──────────┘ │
 │                                                         │ logs      │
 │                                                         ▼           │
 │                              ┌───────────────────────────────────┐ │
 │                              │ Wazuh Server (all-in-one)         │ │
 │                              │ Manager + Indexer + Dashboard     │ │
 │                              │ (dashboard consulté via navigateur│ │
 │                              │  de l'hôte, pas de GUI sur la VM) │ │
 │                              └───────────────────────────────────┘ │
 └──────────────────────────────────────────────────────────────────┘
```

### 3.2 Choix "CLI only"
- Toutes les VM (Kali, Ubuntu Server, Wazuh) sont installées en mode **serveur / sans interface graphique**.
- Administration exclusivement en SSH depuis le terminal de l'hôte.
- Le dashboard Wazuh (interface web) est consulté depuis le navigateur de l'hôte Windows, en pointant vers l'IP de la VM Wazuh — aucune VM n'a besoin d'environnement graphique pour cela.

### 3.3 Dimensionnement des ressources

| VM | OS | RAM allouée | vCPU | Rôle |
|---|---|---|---|---|
| Wazuh Server | Ubuntu Server 22.04 LTS (CLI) | 4 Go | 2 | Manager + Indexer + Dashboard |
| Ubuntu Victime | Ubuntu Server 22.04 LTS (CLI) | 1,5 Go | 1 | SSH, Apache, DVWA, agent Wazuh, Suricata |
| Kali Attaquant | Kali Linux (CLI/headless) | 1,5 Go | 2 | Outils d'attaque en ligne de commande |
| **Total VM** | | **~7 Go** | | Marge confortable sur 16 Go |

### 3.4 Composants logiciels

| Composant | Machine | Rôle |
|---|---|---|
| Wazuh Manager | Wazuh Server | Corrélation, règles, alertes |
| Wazuh Indexer | Wazuh Server | Stockage des événements (OpenSearch) |
| Wazuh Dashboard | Wazuh Server | Visualisation (consultée depuis l'hôte) |
| Wazuh Agent | Ubuntu Victime | Collecte logs, FIM, SCA |
| OpenSSH Server | Ubuntu Victime | Service exposé, cible de brute force |
| Apache2 | Ubuntu Victime | Serveur web |
| DVWA | Ubuntu Victime | Application web volontairement vulnérable |
| Suricata | Ubuntu Victime (ou Wazuh Server) | Détection réseau (NIDS) |
| nmap, hydra, sqlmap, curl, nikto | Kali | Outils d'attaque CLI |

---

## 4. Fonctionnalités attendues

### 4.1 Détection (Blue Team)
- FO-01 : Détecter une tentative de brute force SSH (volumétrie d'échecs d'authentification).
- FO-02 : Détecter une connexion SSH réussie après plusieurs échecs (compte potentiellement compromis).
- FO-03 : Détecter une tentative d'injection SQL sur DVWA via les logs Apache et/ou Suricata.
- FO-04 : Détecter le dépôt d'un webshell via le module FIM (surveillance d'intégrité des fichiers).
- FO-05 : Détecter un scan de ports (nmap) via Suricata.
- FO-06 : Évaluer le niveau de durcissement du serveur via le module SCA (CIS Benchmark).
- FO-07 : Bloquer automatiquement une IP après brute force détectée (Active Response).

### 4.2 Attaque (Red Team / validation)
- FO-08 : Scanner la cible (découverte de services, versions).
- FO-09 : Réaliser un brute force SSH avec Hydra.
- FO-10 : Exploiter une injection SQL sur DVWA avec sqlmap.
- FO-11 : Déposer un webshell via une faille d'upload de DVWA.
- FO-12 : Exécuter des commandes à distance via le webshell.

### 4.3 Documentation et mesure
- FO-13 : Tenir une matrice de couverture MITRE ATT&CK (technique testée / détectée / règle associée).
- FO-14 : Rédiger un rapport d'incident par scénario d'attaque majeur.
- FO-15 : Documenter chaque règle Wazuh custom créée (objectif, logique, test de validation).

---

## 5. Spécifications non fonctionnelles

| Exigence | Détail |
|---|---|
| Performance | Le lab doit tourner avec les 3 VM actives simultanément sans swap excessif sur 16 Go |
| Reproductibilité | Scripts d'installation documentés ou automatisés (bash) pour pouvoir recréer le lab |
| Isolation | Aucune communication entre le lab et le réseau domestique pendant les phases d'attaque |
| Traçabilité | Chaque test doit être horodaté et retrouvable dans les logs Wazuh |
| Portabilité de la documentation | README et rapports rédigés en Markdown, versionnés sur GitHub |

---

## 6. Planning

| Semaine | Phase | Livrable associé |
|---|---|---|
| S1 | Déploiement Wazuh + VM Ubuntu victime + agent, réseau isolé | Architecture fonctionnelle, agent connecté |
| S1-S2 | Installation Apache, DVWA, SSH, Suricata, intégration des logs | Suricata et logs web remontés dans Wazuh |
| S2 | Phase d'attaque (scan, brute force, injection SQL, webshell) | Captures d'alertes + logs bruts |
| S2-S3 | Analyse des résultats, écriture de règles custom, re-test | Règles Wazuh versionnées + preuve de détection |
| S3 | Rédaction des rapports d'incident et de la matrice ATT&CK | Rapports + matrice de couverture |
| S3 | Finalisation documentation GitHub | README, schéma, vidéo démo (optionnel) |

---

## 7. Livrables

1. **Dépôt GitHub** structuré avec :
   - `README.md` (ce document, mis à jour avec les résultats finaux)
   - `architecture.md` (schéma détaillé + justification des choix)
   - `rules/` (règles Wazuh custom au format XML, commentées)
   - `attack-scenarios/` (scripts ou commandes utilisées pour chaque attaque)
   - `incident-reports/` (rapports au format Markdown)
   - `attack-coverage.md` (matrice MITRE ATT&CK)
2. **Matrice de couverture ATT&CK** : tableau technique / statut détection / règle.
3. **2 à 3 rapports d'incident** structurés (résumé, timeline, IOC, analyse, remédiation).
4. **Démonstration vidéo courte (optionnelle)** : attaque → alerte → réponse.

---

## 8. Critères de réussite (KPI)

| Critère | Objectif cible |
|---|---|
| Techniques ATT&CK testées | Au moins 8 techniques distinctes |
| Taux de détection initial | Mesuré et documenté (même faible) |
| Taux de détection après règles custom | Amélioration démontrable (ex. +40 points) |
| Règles custom créées | Au moins 5, documentées et testées |
| Active Response fonctionnel | Blocage automatique validé sur au moins 1 scénario |
| Documentation | README et rapports complets, compréhensibles par un tiers |

---

## 9. Risques et points de vigilance

| Risque | Impact | Mitigation |
|---|---|---|
| Consommation RAM excessive si les 3 VM tournent en continu | Ralentissement du lab | Éteindre Kali hors des phases d'attaque |
| Suricata et agent Wazuh sur la même VM que la cible | Un attaquant compromettant la cible peut désactiver la détection | Documenté comme limite de lab ; testable comme scénario d'évasion (T1562) |
| Dérive de périmètre (vouloir tout faire en V1) | Projet jamais terminé | Respecter strictement le hors périmètre défini en 1.4 |
| Oubli de l'isolation réseau | Risque légal/éthique en cas de fuite d'attaque | Vérifier la configuration réseau avant chaque session d'attaque |

---

## 10. Méthodologie de travail

Chaque scénario d'attaque suit le même cycle :

1. **Préparation** : définir la technique ATT&CK visée et le résultat attendu en détection.
2. **Exécution de l'attaque** depuis Kali (CLI).
3. **Vérification** dans le dashboard Wazuh : alerte générée ou non.
4. **Si non détecté** : analyse du log brut, écriture d'une règle/décodeur custom.
5. **Re-test** de la même attaque pour valider la nouvelle règle.
6. **Documentation** : mise à jour de la matrice ATT&CK et rédaction du rapport si pertinent.

---

## 11. Références

- Documentation officielle Wazuh : https://documentation.wazuh.com
- MITRE ATT&CK : https://attack.mitre.org
- DVWA : https://github.com/digininja/DVWA
- Suricata : https://suricata.io
- CIS Benchmarks : https://www.cisecurity.org/cis-benchmarks

---

*Document évolutif : à compléter au fil du projet avec les résultats réels, captures d'écran de preuve et retours d'expérience.*
documentation : https://app.notion.com/p/Wazuh-SOC-406baa6620a2809abda7efe0859e6758?source=copy_link
