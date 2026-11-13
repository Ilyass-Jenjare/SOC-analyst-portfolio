# 02 - Investigation d'attaque : *Boogeyman 1*

![Plateforme](https://img.shields.io/badge/Plateforme-TryHackMe-red)
![Verdict](https://img.shields.io/badge/Verdict-COMPROMISSION%20CONFIRMÉE-critical)

Reconstitution d'une chaîne de compromission complète, de l'e-mail de phishing jusqu'à l'exfiltration de données, à partir des en-têtes e-mail, des journaux PowerShell et d'une capture réseau (PCAP).

📄 **[Lire le rapport complet (PDF)](rapport.pdf)** · 🕒 **[Chronologie technique détaillée](incident_timeline.md)**

## Contexte

Julianne, du service financier de Quick Logistics LLC, reçoit un e-mail concernant une facture impayée du partenaire B Packaging Inc. La pièce jointe est malveillante et exploitée par le groupe de menace **Boogeyman**. L'investigation couvre l'accès initial, les actions sur le poste et le trafic réseau de commande et contrôle (C2).

## Verdict

**Compromission confirmée** : exécution de code sur le poste, accès à des données sensibles et exfiltration vers une infrastructure C2.

## Chaîne d'attaque

| Étape | Constat |
|---|---|
| Accès initial | Phishing ciblé via l'infrastructure Elastic Email, avec le domaine de typosquatting `bpakcaging.xyz`. Pièce jointe : archive ZIP protégée par mot de passe contenant un fichier LNK malveillant |
| Exécution | Le LNK contient une charge PowerShell obfusquée en Base64, qui récupère des instructions depuis `files.bpakcaging.xyz` |
| Découverte et vol | Outils d'énumération (`Seatbelt.exe`, `sq3.exe`) utilisés pour accéder à `plum.sqlite` (Sticky Notes) et `protected_data.kdbx` (KeePass) |
| Exfiltration | Résultats et fichiers envoyés par requêtes HTTP POST vers le C2 `cdn.bpakcaging.xyz` |

## Indicateurs de compromission (IOC)

| Type | Valeur |
|---|---|
| Domaine (typosquatting) | `bpakcaging[.]xyz` |
| Serveur de payload | `files.bpakcaging[.]xyz` |
| Serveur C2 / exfiltration | `cdn.bpakcaging[.]xyz` |
| Outils | `Seatbelt.exe`, `sq3.exe` |
| Données ciblées | `plum.sqlite`, `protected_data.kdbx` |

## Méthodologie

1. Analyse de l'e-mail : en-têtes (DKIM, SPF, relais tiers), corps, extraction de la pièce jointe
2. Analyse du LNK (`lnkparse`) et décodage de la charge PowerShell (CyberChef)
3. Exploitation des journaux PowerShell JSON en ligne de commande (`jq`, `grep`)
4. Analyse du PCAP dans Wireshark : filtrage HTTP, reconstruction de flux TCP, trafic C2 et exfiltration
5. Corrélation des sources et rédaction de la chronologie

## MITRE ATT&CK

`T1566.001` Spearphishing Attachment · `T1204.002` User Execution · `T1059.001` PowerShell · `T1027` Obfuscated Files · `T1005` Data from Local System · `T1555.005` Password Managers · `T1071.001` Web Protocols (C2) · `T1041` Exfiltration Over C2 Channel

## Outils

- **E-mail :** Thunderbird, vérification des en-têtes
- **Endpoint :** `lnkparse`, `jq`, `grep`
- **Réseau :** Wireshark
- **Décodage :** CyberChef (Base64, Hex)

## Structure du dossier

```text
02_Attack_Investigation/
├── README.md               résumé du projet
├── incident_timeline.md    chronologie technique détaillée
├── rapport.pdf             rapport d'investigation (LaTeX)
└── screenshots/            preuves
    ├── 01_email_headers.jpg    en-têtes (DKIM, relais)
    ├── 02_email_body.png       ingénierie sociale et mot de passe ZIP
    ├── 03_lnk_parse.png        extraction de la charge PowerShell
    ├── 04_powershell_c2.png    boucle de communication C2
    ├── 05_data_access.png      accès aux fichiers .sqlite et .kdbx
    └── 06_wireshark_c2.jpg     trafic réseau vers le serveur de payload
```

## Compétences

Analyse d'e-mails · Forensique endpoint · Analyse PowerShell · Forensique réseau (Wireshark) · Désobfuscation Base64/Hex · Corrélation multi-sources · Chronologie d'incident · MITRE ATT&CK
