# Scénario 1 : Attaque par Déni de Service (SYN Flood) et Défense Active

![Status](https://img.shields.io/badge/Status-Validé-brightgreen)
![Severity](https://img.shields.io/badge/Wazuh_Level-3-yellow)
![MITRE](https://img.shields.io/badge/MITRE_ATT%26CK-T1498.001-blue)

##  Résumé exécutif

| | |
|---|---|
| **Objectif** | Valider la capacité de Défense Active du FortiGate à intercepter une attaque volumétrique (L4) et confirmer la transmission de l'alerte au SIEM (Wazuh) |
| **Technique MITRE ATT&CK** | [T1498.001 – Network Denial of Service: Direct Network Flood](https://attack.mitre.org/techniques/T1498/001/) |
| **Attaquant** | Kali Linux — `10.0.10.30` (Zone LAN) |
| **Cible** | Serveur Web — `192.16.10.20` (Zone DMZ) |
| **Résultat** | Attaque bloquée au périmètre, alerte Wazuh générée (Règle `81619`) |

---

## 1. Contexte et faille ciblée

Le SYN Flood exploite une vulnérabilité architecturale inhérente au protocole TCP : l'attaquant initie massivement des connexions semi-ouvertes (SYN) sans jamais compléter le *three-way handshake*, dans le but de saturer la table d'état du pare-feu ou d'épuiser les ressources du serveur cible. Cette attaque réussit systématiquement en l'absence de protection volumétrique en amont.

---

## 2. Préparation de la posture de défense (FortiGate)

**Navigation :** `Policy & Objects > IPv4 DoS Policy`

**Configuration de la règle :**

| Paramètre | Valeur |
|---|---|
| Incoming Interface | `port2` (LAN) |
| Source / Destination / Service | `all` / `all` / `ALL` |
| Anomaly | `tcp_syn_flood` |
| Action | `Block` |
| Logging | `Enable` |
| Threshold | `50` paquets/seconde *(seuil abaissé pour déclenchement immédiat en laboratoire)* |
| Policy Status | `Enable` |

**Principe de l'Early Drop :** le capteur d'anomalie, positionné sur l'interface entrante, détruit les paquets malveillants dès réception, avant qu'ils n'atteignent la DMZ — préservant ainsi les ressources du pare-feu et du serveur cible.

 *Preuve : [`evidence/01_fortigate_dos_policy.png`](./evidence/01_fortigate_dos_policy.png)*

---

## 3. Exécution de l'attaque (Kali Linux)

**Machine attaquante :** `10.0.10.30`

```bash
sudo hping3 -S -p 80 --flood 192.16.10.20
```

| Argument | Rôle |
|---|---|
| `-S` | Active le flag TCP SYN |
| `-p 80` | Cible le port HTTP de la victime |
| `--flood` | Envoi des paquets à vitesse maximale |

> **Note technique :** l'option `--rand-source` n'est volontairement pas utilisée, afin de confirmer que Wazuh identifie correctement l'adresse IP réelle de la machine attaquante.

**Déroulement du test :**
1. Lancement de l'attaque pendant 5 à 10 secondes.
2. Interruption via `Ctrl + C`.
3. Résultat observé : `100% packet loss` — confirmant l'interception des paquets par le pare-feu.

 *Preuve : [`evidence/02_kali_hping3_attack.png`](./evidence/02_kali_hping3_attack.png)*

---

## 4. Chaîne de détection et triage SIEM (Wazuh)

À la suite du blocage, la chaîne de détection suivante s'exécute de bout en bout :

1. Le FortiGate bloque l'attaque (`Action: Block`) et transmet un log Syslog au Wazuh Manager.
2. Le décodeur `fortigate-firewall-v6` normalise le log.
3. Le moteur de corrélation déclenche l'alerte de sécurité.

**Triage dans Wazuh (`Security Events`) :**

| Champ | Valeur observée |
|---|---|
| **Règle déclenchée** | `81619` |
| **Description** | *Fortigate: Multiple high traffic events from same source.* |
| **Niveau de sévérité** | `3` |
| **data.srcip** | `10.0.10.30` |
| **data.dstip** | `192.16.10.20` |
| **data.action** | `Block` |

**Résultat :** l'attaque est bloquée en amont et l'analyste SOC identifie positivement l'IP interne responsable du trafic anormal.

 *Preuve : [`evidence/03_wazuh_alert_81619.png`](./evidence/03_wazuh_alert_81619.png)*

---

##  Checklist de validation

- [x] Règle DoS Policy FortiGate active (seuil 50, action Block, logging Enable)
- [x] Attaque `hping3` lancée depuis Kali (`10.0.10.30`) vers `192.16.10.20`
- [x] Alerte Niveau 3 visible dans Wazuh (règle `81619`)
- [x] Capture d'écran de l'alerte (`data.action`, `data.srcip`, `data.dstip`)
- [x] Confirmation du blocage au niveau du pare-feu

---

##  Enseignements clés (Lessons Learned)

- La **Défense Active L4** du FortiGate neutralise efficacement les attaques volumétriques basiques sans intervention humaine.
- L'intégration **FortiGate → Syslog → Wazuh** permet une visibilité SOC complète, du blocage périmétrique jusqu'à l'alerte exploitable par l'analyste.
- Un seuil de détection réaliste en production devrait être recalibré (le seuil `50` pps utilisé ici est volontairement abaissé pour le laboratoire).

---

*Scénario suivant : [02_SQL_Injection.md](../scenario_2/02_SQL_Injection.md)*