# Scénario 2 : Attaque Web Applicative (SQL Injection) et Détection HIDS

![Status](https://img.shields.io/badge/Status-Validé-brightgreen)
![Severity](https://img.shields.io/badge/Wazuh_Level-5-orange)
![MITRE](https://img.shields.io/badge/MITRE_ATT%26CK-T1190-blue)

##  Résumé exécutif

| | |
|---|---|
| **Objectif** | Valider la stratégie de Defense-in-Depth : démontrer que lorsqu'une attaque applicative (L7) traverse le pare-feu faute d'inspection de contenu, l'agent HIDS Wazuh installé sur l'hôte prend le relais |
| **Technique MITRE ATT&CK** | [T1190 – Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/) |
| **Attaquant** | Kali Linux — `10.0.10.30` (Zone LAN) |
| **Cible** | Serveur Web applicatif — `192.16.10.20` (Zone DMZ), agent Wazuh `victim2` |
| **Résultat** | Attaque **non bloquée** par le réseau, mais **détectée** par le HIDS (Règle `31101`) |

---

## 1. Contexte et faille ciblée

Contrairement au Scénario 1 (attaque volumétrique L4), ce scénario cible la couche applicative (L7). Le pare-feu FortiGate autorise le trafic HTTP vers la DMZ via un port légitime (80), mais sans profil de sécurité (IPS/WAF), il ne peut pas analyser le contenu des requêtes. La faille réside donc dans deux absences combinées :
- **Absence d'inspection de contenu** côté pare-feu (Security Profiles vide).
- **Vulnérabilité applicative** côté serveur web (paramètre `id` non assaini, injectable).

---

## 2. Préparation de la posture (FortiGate — sans inspection)

**Navigation :** `Policy & Objects > IPv4 Policy > LAN_to_DMZ`

| Paramètre | Valeur |
|---|---|
| Source / Destination | `LAN` / `DMZ` |
| Service | `HTTP` |
| Action | `ACCEPT` |
| Security Profiles | *(vide — aucun IPS, aucun WAF)* |

Le pare-feu agit ici en simple routeur filtrant L3/L4 : il vérifie l'adresse et le port, mais laisse passer le contenu du paquet sans inspection.


---

## 3. Exécution de l'attaque (Kali Linux)

**Machine attaquante :** `10.0.10.30`

```bash
curl -s "http://192.16.10.20/vulnerabilities/sqli/?id=1'+OR+1=1--&Submit=Submit" > /dev/null
```

**Payload :** `1' OR 1=1--` — injection SQL classique visant à altérer la logique de la requête côté serveur (contournement d'authentification / extraction de données selon le contexte applicatif).

**Résultat réseau :** la requête traverse le FortiGate sans être bloquée ni journalisée comme suspecte, confirmant l'absence d'inspection L7.


---

## 4. Chaîne de détection et triage SIEM (Wazuh — HIDS)

L'attaque ayant atteint le serveur, le payload malformé provoque une erreur applicative. L'agent Wazuh installé localement (`victim2`) lit les journaux Apache (`access.log` / `error.log`) et remonte l'anomalie au Manager.

**Triage dans Wazuh (`Threat Hunting`) :**

| Champ | Valeur observée |
|---|---|
| **agent.name** | `victim2` |
| **Règle déclenchée** | `31101` |
| **Description** | *Web server 400 error code.* |
| **Niveau de sévérité** | `5` |
| **Source de la détection** | Journaux applicatifs Apache (HIDS), **pas** le FortiGate |

**Résultat :** le réseau n'a pas bloqué la menace, mais la Défense en Profondeur a fonctionné — l'analyste SOC est alerté d'une requête web malformée (signature probable d'injection) ciblant le serveur DMZ, permettant d'engager une réponse à incident.

---

##  Checklist de validation

- [x] Règle FortiGate `LAN_to_DMZ` confirmée sans Security Profiles (aucune inspection L7)
- [x] Requête SQLi envoyée depuis Kali (`10.0.10.30`) vers `192.16.10.20`
- [x] Trafic non bloqué au niveau réseau (traversée confirmée)
- [x] Alerte Niveau 5 visible dans Wazuh (règle `31101`, agent `victim2`)
- [x] Capture d'écran de l'alerte HIDS

---

##  Enseignements clés (Lessons Learned)

- Une politique de pare-feu sans **Security Profiles** ne protège que contre les menaces de couche 3/4 : elle est aveugle à toute attaque applicative transitant sur un port autorisé.
- Le **HIDS** (agent Wazuh sur l'hôte) constitue une ligne de défense complémentaire indispensable, capable de détecter ce que le réseau ne voit pas.
- La règle `31101` (erreur 400) est un indicateur **générique** : elle signale une anomalie de requête, pas spécifiquement une injection SQL. Une règle dédiée affinerait la précision (voir recommandations).

---

##  Recommandations

La question « faut-il activer l'IPS du firewall » appelle une réponse en plusieurs couches — l'IPS seul ne suffit pas pour ce type d'attaque, et c'est justement tout l'intérêt pédagogique de ce scénario.

### 1. Activer un profil de sécurité L7 sur le FortiGate — nécessaire mais insuffisant seul
- **IPS (Intrusion Prevention System)** : ajoute un profil IPS à la politique `LAN_to_DMZ`, avec les signatures de la catégorie *Web Application* / *SQL Injection* activées en mode `Block`. Cela peut intercepter des payloads SQLi *connus et signés*, mais un attaquant qui obfusque sa requête (encodage, espaces, commentaires SQL) peut contourner une signature générique.
- **Limite réelle** : un NGFW avec IPS est pensé pour des exploits réseau et des signatures connues, pas pour une analyse syntaxique fine du paramètre d'une application web. C'est pourquoi FortiGate propose un module distinct pour ce cas précis.

### 2. Déployer un WAF (Web Application Firewall) — la vraie réponse au problème
- Sur FortiGate, cela correspond au profil **Web Application Firewall** dans Security Profiles, qu'on peut combiner avec l'IPS sur la même politique.
- Le WAF applique des règles spécifiques aux attaques applicatives (OWASP Top 10 : SQLi, XSS, path traversal...) en analysant la structure des requêtes HTTP (paramètres, en-têtes, body), ce qu'un IPS générique ne fait pas aussi finement.
- Alternative si pas de FortiWeb/WAF dédié : un reverse proxy avec **ModSecurity** + règles **OWASP CRS** devant le serveur applicatif.

### 3. Corriger la vulnérabilité à la source (priorité réelle en environnement de production)
- Le vrai problème est que le paramètre `id` n'est pas assaini côté application (requêtes SQL non préparées). Aucun contrôle périmétrique ne remplace une correction applicative :
  - Utiliser des **requêtes préparées / paramétrées** (prepared statements).
  - Valider et filtrer les entrées utilisateur côté serveur.
  - Appliquer le principe du moindre privilège sur le compte de base de données utilisé par l'application.

### 4. Affiner la détection côté Wazuh
- Créer une règle personnalisée dans `local_rules.xml` qui corrèle la règle générique `31101` avec la présence de motifs SQLi dans l'URL loggée (`OR 1=1`, `UNION SELECT`, `--`, `'`), pour distinguer une vraie tentative d'injection d'une simple erreur applicative bénigne.
- Augmenter la sévérité de cette règle dérivée (niveau 10+) pour qu'elle déclenche une alerte prioritaire plutôt qu'un niveau 5 générique, facilement noyé dans le bruit.

### Synthèse priorisée

| Priorité | Action | Effet |
|---|---|---|
| 1 | Corriger le code applicatif (requêtes préparées) | Élimine la vulnérabilité à la racine |
| 2 | Activer WAF (ou ModSecurity/CRS) sur le chemin HTTP | Bloque la majorité des injections avant l'application |
| 3 | Activer IPS en complément sur la politique DMZ | Couvre les signatures d'exploits réseau connus |
| 4 | Règle Wazuh dédiée SQLi | Réduit le bruit et accélère le triage SOC |

---

*Scénario précédent : [01_DoS_SYN_Flood.md](../scenario_1/01_DoS_SYN_Flood.md)*
*Scénario suivant : [03_IPS_Blocking.md](../scenario_3/03_IPS_Blocking.md)*