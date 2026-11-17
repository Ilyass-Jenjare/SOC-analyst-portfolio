# Scénario 2 : Attaque Web Applicative (SQL Injection) et Détection HIDS

![Status](https://img.shields.io/badge/Status-Validé-brightgreen)
![Severity](https://img.shields.io/badge/Wazuh_Level-12-red)
![MITRE](https://img.shields.io/badge/MITRE_ATT%26CK-T1190-blue)

##  Résumé exécutif

| | |
|---|---|
| **Objectif** | Valider la stratégie de Defense-in-Depth : démontrer que lorsqu'une attaque applicative (L7) traverse le pare-feu faute d'inspection de contenu, l'agent HIDS Wazuh prend le relais — puis affiner la détection générique en règle SQLi dédiée à haute sévérité |
| **Technique MITRE ATT&CK** | [T1190 – Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/) |
| **Attaquant** | Kali Linux — `10.0.10.30` (Zone LAN) |
| **Cible** | Serveur Web applicatif — `192.16.10.20` (Zone DMZ), agent Wazuh `victim2` |
| **Résultat** | Attaque **non bloquée** par le réseau, **détectée** par le HIDS, puis **qualifiée et priorisée** par une règle de corrélation custom (`100050`, niveau 12) |

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

## 4. Détection initiale (HIDS générique)

L'attaque ayant atteint le serveur, le payload malformé provoque une erreur applicative. L'agent Wazuh installé localement (`victim2`) lit les journaux Apache (`access.log` / `error.log`) et remonte une première alerte générique au Manager.

| Champ | Valeur observée |
|---|---|
| **agent.name** | `victim2` |
| **Règle déclenchée** | `31101` |
| **Description** | *Web server 400 error code.* |
| **Niveau de sévérité** | `5` |

**Limite identifiée :** cette règle native ne distingue pas une tentative d'injection d'une simple erreur applicative bénigne (requête malformée, typo utilisateur, bug front-end...). Elle sert uniquement de signal faible.


---

## 5. Affinage de la détection — Règle de corrélation SQLi personnalisée

Pour éliminer ce bruit et qualifier précisément la menace, une règle locale a été écrite sur le Wazuh Manager afin de corréler l'erreur HTTP 400 (`31101`) avec la présence de motifs d'injection SQL dans la requête loggée.

**Fichier :** `/var/ossec/etc/rules/local_rules.xml`

```xml
<group name="apache,sqli,">
  <!-- Règle de détection SQLi spécifique -->
  <rule id="100050" level="12">
    <if_sid>31101</if_sid>
    <match>SELECT |UNION |INSERT |UPDATE |DELETE |DROP |' OR|'+OR</match>
    <description>SQL Injection attempt detected in URL (HTTP 400 Error).</description>
    <mitre>
      <id>T1190</id>
    </mitre>
    <group>attack,sqli,</group>
  </rule>
</group>
```

**Logique de la règle :**
- `<if_sid>31101</if_sid>` : la règle ne s'évalue que sur les événements déjà classés comme erreur HTTP 400 (règle parente), évitant tout faux positif sur du trafic normal.
- `<match>` : recherche de motifs SQLi courants dans l'URL loggée (mots-clés SQL, opérateurs d'évasion `OR`, commentaires `--`).
- **Niveau 12** (contre 5 pour la règle générique) : classe l'alerte en sévérité haute, visible immédiatement dans le dashboard principal et non noyée dans le bruit des erreurs 400 ordinaires.
- Tag **MITRE ATT&CK T1190** : enrichit l'alerte pour la cartographie et le reporting.

**Déploiement :**
```bash
# Validation de la syntaxe avant redémarrage
/var/ossec/bin/wazuh-logtest

# Application de la règle
systemctl restart wazuh-manager
```

---

## 6. Validation finale (Triage SIEM)

Rejeu de l'attaque après activation de la règle `100050` :

| Champ | Valeur observée |
|---|---|
| **Règle déclenchée** | `100050` (dérivée de `31101`) |
| **Description** | *SQL Injection attempt detected in URL (HTTP 400 Error).* |
| **Niveau de sévérité** | `12` |
| **Groupe** | `attack, sqli` |
| **Tag MITRE** | `T1190` |

**Résultat :** l'alerte générique de niveau 5, auparavant noyée dans le volume d'erreurs HTTP courantes, est désormais remontée comme un événement de sécurité critique et explicitement qualifié d'injection SQL, avec mapping MITRE ATT&CK natif — directement exploitable par l'analyste SOC pour une réponse à incident.


---

##  Checklist de validation

- [x] Règle FortiGate `LAN_to_DMZ` confirmée sans Security Profiles (aucune inspection L7)
- [x] Requête SQLi envoyée depuis Kali (`10.0.10.30`) vers `192.16.10.20`
- [x] Trafic non bloqué au niveau réseau (traversée confirmée)
- [x] Alerte générique Niveau 5 visible dans Wazuh (règle `31101`, agent `victim2`)
- [x] Règle locale `100050` créée et déployée (`local_rules.xml`)
- [x] Syntaxe validée via `wazuh-logtest` avant redémarrage
- [x] Alerte Niveau 12 confirmée après rejeu de l'attaque
- [x] Capture d'écran de l'alerte qualifiée (règle `100050`)

---

##  Enseignements clés (Lessons Learned)

- Une politique de pare-feu sans **Security Profiles** ne protège que contre les menaces de couche 3/4 : elle est aveugle à toute attaque applicative transitant sur un port autorisé.
- Le **HIDS** (agent Wazuh sur l'hôte) constitue une ligne de défense complémentaire indispensable, capable de détecter ce que le réseau ne voit pas — mais une règle générique (`31101`) reste un signal faible, insuffisant pour prioriser une réponse.
- L'écriture d'une **règle de corrélation custom** (`if_sid` + `match`) transforme un bruit de fond en alerte de sécurité qualifiée et actionnable, sans dépendre d'un équipement réseau supplémentaire.
- Cette approche illustre une compétence SOC clé : **adapter la détection aux logs réellement disponibles**, plutôt que d'attendre une solution périmétrique idéale.

---

##  Recommandations restantes

La détection applicative est désormais opérationnelle côté SIEM. Les axes suivants restent pertinents pour renforcer la défense en profondeur :

### 1. Corriger la vulnérabilité à la source (priorité n°1)
- Utiliser des **requêtes préparées / paramétrées** (prepared statements) côté application.
- Valider et filtrer les entrées utilisateur côté serveur.
- Appliquer le principe du moindre privilège sur le compte de base de données utilisé par l'application.

### 2. Ajouter une couche de blocage réseau (défense en profondeur)
- **WAF (Web Application Firewall)** sur FortiGate (profil dédié dans Security Profiles) ou via un reverse proxy **ModSecurity + OWASP CRS** devant le serveur applicatif — pour bloquer la requête *avant* qu'elle n'atteigne l'application.
- **IPS** en complément sur la politique `LAN_to_DMZ` : utile contre des signatures d'exploits réseau connus, mais à ne pas considérer comme suffisant seul face à une injection applicative obfusquée (voir limite documentée au Scénario 3).


---

*Scénario précédent : [01_DoS_SYN_Flood.md](../scenario_1/01_DoS_SYN_Flood.md)*
*Scénario suivant : [03_IPS_Blocking.md](../scenario_3/03_IPS_Blocking.md)*