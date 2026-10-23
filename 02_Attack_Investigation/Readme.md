#  Projet 2 : Investigation d'Attaque (Boogeyman 1)

**Objectif :** Analyser une chaîne de compromission complète, de la réception d'un e-mail de phishing jusqu'à l'exfiltration de données sensibles, en s'appuyant sur l'analyse d'en-têtes, de journaux d'événements (PowerShell) et de captures réseau (PCAP).

##  Contexte de l'Incident
L'employée Julianne (département financier de Quick Logistics LLC) a reçu un e-mail concernant une facture impayée du partenaire B Packaging Inc. Cet e-mail contenait une pièce jointe malveillante exploitée par le groupe de menace **Boogeyman**. L'investigation couvre l'analyse du vecteur d'accès initial, les actions sur le point de terminaison et le trafic réseau de commande et contrôle (C2).

##  Compétences et Outils
- **Analyse d'e-mails :** Thunderbird, extraction de charges utiles, vérification des en-têtes (DKIM, SPF, relais tiers).
- **Forensique Endpoint :** Analyse d'artefacts LNK (`lnkparse`), traitement de journaux JSON PowerShell via la ligne de commande (`jq`, `grep`).
- **Analyse Réseau :** Wireshark (Filtrage HTTP, reconstruction de flux TCP, analyse de trafic C2 et d'exfiltration).
- **Rétro-ingénierie légère :** Décodage Base64 et Hex via CyberChef.

##  Découvertes Clés
1. **Accès Initial :** Phishing ciblé via l'infrastructure "Elastic Email" utilisant le domaine de typosquatting `bpakcaging.xyz`. La pièce jointe était une archive ZIP protégée par mot de passe contenant un fichier LNK malveillant.
2. **Exécution :** Le fichier LNK contenait une charge utile PowerShell obfusquée en Base64 téléchargeant des instructions depuis `files.bpakcaging.xyz`.
3. **Découverte et Vol :** Déploiement d'outils d'énumération (`Seatbelt.exe`, `sq3.exe`) pour accéder à des données sensibles (`plum.sqlite` pour Sticky Notes et `protected_data.kdbx` pour KeePass).
4. **Exfiltration :** Envoi des résultats d'exécution et des fichiers via des requêtes HTTP POST vers le domaine C2 `cdn.bpakcaging.xyz`.

##  Structure du Répertoire
```text
02_Attack_Investigation/
├── README.md                  # Résumé du projet (ce fichier)
├── incident_timeline.md       # Chronologie technique détaillée de l'attaque
├── report.pdf                 # Rapport d'investigation officiel (généré via LaTeX)
└── screenshots/               # Preuves et artefacts visuels
    ├── 01_email_headers.jpg   # En-têtes (DKIM, Relais)
    ├── 02_email_body.png      # Ingénierie sociale & mot de passe ZIP
    ├── 03_lnk_parse.png       # Extraction de la charge utile PowerShell
    ├── 04_powershell_c2.png   # Boucle de communication C2
    ├── 05_data_access.png     # Accès aux fichiers .sqlite et .kdbx
    └── 06_wireshark_c2.jpg    # Trafic réseau vers le Payload Server