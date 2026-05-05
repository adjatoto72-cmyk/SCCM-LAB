# 🖥️ Lab SCCM / MECM — Guide Complet de A à Z

> **Environnement** : Windows Server 2022 + SCCM 2509 + SQL Server 2019  
> **Objectif** : Monter un lab complet de packaging et déploiement d'applications avec Microsoft Configuration Manager  
> **Niveau** : Débutant → Intermédiaire

---

## 📋 Table des matières

- [Architecture du lab](#architecture-du-lab)
- [Prérequis matériels](#prérequis-matériels)
- [Phase 1 — Prérequis & SQL Server 2019](#phase-1--prérequis--sql-server-2019)
- [Phase 2 — Installation de SCCM 2509](#phase-2--installation-de-sccm-2509)
- [Phase 3 — Configuration de la machine cliente](#phase-3--configuration-de-la-machine-cliente)
- [Phase 4 — Packaging & Déploiement d'applications](#phase-4--packaging--déploiement-dapplications)
- [Phase 5 — Fonctionnalités avancées](#phase-5--fonctionnalités-avancées)
- [Liens de téléchargement](#liens-de-téléchargement)
- [Résolution des problèmes courants](#résolution-des-problèmes-courants)

---

## 🏗️ Architecture du lab

```
┌─────────────────────────────────────────┐
│         Windows Server 2022             │
│  ┌─────────────┐  ┌──────────────────┐  │
│  │  AD DS      │  │  SQL Server 2019 │  │
│  │  DNS        │  │  (MSSQLSERVER)   │  │
│  └─────────────┘  └──────────────────┘  │
│  ┌─────────────────────────────────────┐ │
│  │         SCCM 2509 (P05)            │ │
│  │  - Management Point                │ │
│  │  - Distribution Point              │ │
│  │  - Software Update Point           │ │
│  └─────────────────────────────────────┘ │
│  IP : 192.168.223.129                   │
└─────────────────────────────────────────┘
                    │
                    │ Domaine : sccm.lab
                    │
┌─────────────────────────────────────────┐
│         VM Cliente (TEST)               │
│         Windows 10 / 11                │
│         IP : 192.168.223.130           │
│         Client SCCM installé ✅        │
└─────────────────────────────────────────┘
```

---

## 💻 Prérequis matériels

| Ressource | Serveur SCCM | VM Cliente |
|-----------|-------------|------------|
| **RAM** | 16 Go recommandé (8 Go minimum) | 4 Go minimum |
| **CPU** | 4 vCPU | 2 vCPU |
| **Disque OS** | 100 Go | 60 Go |
| **Disque SQL** | 50 Go (séparé si possible) | — |
| **Disque SCCM** | 50 Go (séparé si possible) | — |
| **Réseau** | IP fixe | IP fixe même sous-réseau |

---

## Phase 1 — Prérequis & SQL Server 2019

### 1.1 Configuration réseau du serveur

```powershell
# Vérifier l'IP du serveur
Get-NetIPConfiguration

# Vérifier la jonction au domaine
(Get-WmiObject Win32_ComputerSystem).Domain
```

### 1.2 Installer les rôles Windows requis

```powershell
Install-WindowsFeature -Name NET-Framework-Features, NET-Framework-Core, `
NET-Framework-45-Features, NET-Framework-45-Core, `
NET-WCF-Services45, NET-WCF-TCP-PortSharing45, `
BITS, BITS-IIS-Ext, `
Web-Server, Web-WebServer, Web-Common-Http, Web-Default-Doc, `
Web-Dir-Browsing, Web-Http-Errors, Web-Static-Content, `
Web-Http-Redirect, Web-Health, Web-Http-Logging, `
Web-Log-Libraries, Web-Request-Monitor, Web-Http-Tracing, `
Web-Performance, Web-Stat-Compression, Web-Dyn-Compression, `
Web-Security, Web-Filtering, Web-Windows-Auth, `
Web-App-Dev, Web-Net-Ext, Web-Net-Ext45, `
Web-Asp-Net, Web-Asp-Net45, Web-ISAPI-Ext, Web-ISAPI-Filter, `
Web-Mgmt-Tools, Web-Mgmt-Console, Web-Mgmt-Compat, `
Web-Metabase, Web-WMI, RDC `
-Source D:\Sources\SxS -Restart
```

### 1.3 Installer SQL Server 2019

**Paramètres importants lors de l'installation :**

| Paramètre | Valeur |
|-----------|--------|
| **Instance** | MSSQLSERVER (défaut) |
| **Collation** | `SQL_Latin1_General_CP1_CI_AS` ⚠️ Obligatoire |
| **Authentication** | Mixed Mode |
| **Admin SQL** | Ajouter `DOMAINE\svc_sccm` |

### 1.4 Configuration post-installation SQL

```sql
-- Vérifier la collation
SELECT SERVERPROPERTY('Collation')

-- Activer CLR
sp_configure 'clr enabled', 1
RECONFIGURE

-- Désactiver CLR strict security (SQL 2017+)
sp_configure 'clr strict security', 0
RECONFIGURE WITH OVERRIDE

-- Configurer la mémoire max (exemple pour 16 Go RAM)
EXEC sp_configure 'max server memory (MB)', 12288
RECONFIGURE WITH OVERRIDE
```

```powershell
# Activer TCP/IP dans SQL Server Configuration Manager
# Port : 1433
# Redémarrer le service SQL après modification
Restart-Service -Name MSSQLSERVER
```

### 1.5 Installer WSUS

```powershell
Install-WindowsFeature -Name UpdateServices, UpdateServices-WidDB -IncludeManagementTools

& "C:\Program Files\Update Services\Tools\wsusutil.exe" postinstall CONTENT_DIR=C:\WSUS
```

### 1.6 Installer l'ADK Windows 11 24H2

> ⚠️ Toujours installer l'ADK **avant** SCCM

**Fonctionnalités à cocher dans l'ADK :**
- ✅ Outils de déploiement
- ✅ Concepteur de fonctions d'acquisition d'images
- ✅ Concepteur de configuration
- ✅ Outil de migration utilisateur (USMT)
- ✅ Microsoft Application Virtualization (App-V) Sequencer

**Installer ensuite le WinPE add-on :**
- ✅ Windows PE uniquement

### 1.7 Créer les comptes de service AD

```powershell
# Créer le compte de service SCCM
New-ADUser -Name "svc_sccm" -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true

# Ajouter au groupe Administrateurs locaux du serveur
Add-LocalGroupMember -Group "Administrators" -Member "DOMAINE\svc_sccm"
```

### ✅ Checklist Phase 1

- [ ] IP fixe configurée sur le serveur
- [ ] Serveur membre du domaine AD
- [ ] Rôles Windows installés (.NET, IIS, BITS)
- [ ] SQL Server 2019 installé avec collation `SQL_Latin1_General_CP1_CI_AS`
- [ ] TCP/IP activé sur SQL, port 1433
- [ ] SSMS installé et connexion OK
- [ ] WSUS installé
- [ ] ADK Windows 11 24H2 + WinPE add-on installés
- [ ] Compte `svc_sccm` créé et admin local

---

## Phase 2 — Installation de SCCM 2509

> ℹ️ SCCM 2509 n'est pas disponible en ISO directe. On installe la **baseline 2403** puis on met à jour vers 2509 depuis la console.

### 2.1 Étendre le schéma Active Directory

```
# Depuis le media d'installation SCCM
SCCM_Media\SMSSETUP\BIN\X64\extadsch.exe
```

Vérifier le résultat :
```powershell
Get-Content "C:\ExtADSch.log" | Select-String "Successfully"
# Doit afficher : Successfully extended the Active Directory schema
```

### 2.2 Créer le conteneur System Management dans l'AD

1. Ouvrir **ADSI Edit** (`adsiedit.msc`)
2. Naviguer vers `DC=sccm,DC=lab → CN=System`
3. Clic droit → **Nouveau → Objet → container**
4. Nom : `System Management`
5. Ajouter le compte ordinateur du serveur SCCM (`WIN-XXXXXXX$`) avec **Full Control**

### 2.3 Lancer l'installation SCCM

**Paramètres de l'assistant :**

| Paramètre | Valeur |
|-----------|--------|
| **Type** | Install a Configuration Manager primary site |
| **Site code** | P05 |
| **Site name** | P05-SCCM.DEV |
| **Install path** | C:\Program Files\Microsoft Configuration Manager |
| **Type de site** | Stand-alone primary site |
| **SQL Server** | Nom du serveur local |
| **Database name** | CM_P05 |
| **Rôles** | Management Point + Distribution Point |

```powershell
# Surveiller l'installation
Get-Content "C:\ConfigMgrSetup.log" -Wait -Tail 30
```

### 2.4 Configuration post-installation

#### Créer les Boundaries

1. **Administration → Configuration de la hiérarchie → Limites**
2. Créer une limite de type **Plage d'adresses IP** :
   - Début : `192.168.223.1`
   - Fin : `192.168.223.254`

#### Créer le Boundary Group

1. **Administration → Configuration de la hiérarchie → Groupes de limites**
2. Créer un groupe → ajouter la limite → associer le Distribution Point
3. Cocher **Utiliser ce groupe de limites pour l'attribution de site**

#### Configurer la découverte AD

1. **Administration → Configuration de la hiérarchie → Méthodes de découverte**
2. Activer :
   - ✅ Découverte des systèmes Active Directory
   - ✅ Découverte des utilisateurs Active Directory
   - ✅ Découverte par pulsation

### 2.5 Mettre à jour vers SCCM 2509

1. **Administration → Mises à jour et maintenance**
2. Installer dans l'ordre :
   - 2403 → **2409** (~30-45 min)
   - 2409 → **2503** (~30-45 min)
   - 2503 → **2509** (~30-45 min)

### ✅ Checklist Phase 2

- [ ] Schéma AD étendu
- [ ] Conteneur System Management créé
- [ ] SCCM 2403 installé sans erreur
- [ ] Console SCCM accessible
- [ ] Boundaries et Boundary Groups configurés
- [ ] Découverte AD active
- [ ] Site Status tout en vert
- [ ] Mis à jour vers SCCM 2509

---

## Phase 3 — Configuration de la machine cliente

### 3.1 Configuration IP de la VM cliente

| Paramètre | Valeur |
|-----------|--------|
| **Adresse IP** | 192.168.223.130 |
| **Masque** | 255.255.255.0 |
| **Passerelle** | 192.168.223.2 |
| **DNS** | 192.168.223.129 ← pointe vers le serveur SCCM/AD |

```powershell
# Configurer l'IP fixe
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.223.130 -PrefixLength 24 -DefaultGateway 192.168.223.2
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.223.129
```

### 3.2 Joindre la VM au domaine

```powershell
$password = ConvertTo-SecureString "TonMotDePasse" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("sccm\Administrateur", $password)
Add-Computer -DomainName "sccm.lab" -Credential $cred -Restart
```

### 3.3 Installer le client SCCM

```powershell
\\192.168.223.129\SMS_P05\Client\ccmsetup.exe /mp:WIN-UI39PMJB67B.sccm.lab SMSSITECODE=P05 FSP=WIN-UI39PMJB67B.sccm.lab
```

Surveiller l'installation :
```powershell
Get-Content "C:\Windows\ccmsetup\Logs\ccmsetup.log" -Wait -Tail 20
# Attendre : CcmSetup is exiting with return code 0
```

### 3.4 Vérifier l'installation du client

```powershell
# Vérifier le service
Get-Service -Name 'CcmExec' | Select-Object Status, DisplayName

# Vérifier le site code assigné
Get-WmiObject -Namespace root\ccm -Class SMS_Client | Select-Object AssignedSiteCode, ClientVersion
```

### 3.5 Résolution — Site code vide (problème courant)

```powershell
# Forcer le site code dans le registre
reg add "HKLM\SOFTWARE\Microsoft\SMS\Mobile Client" /v "AssignedSiteCode" /t REG_SZ /d "P05" /f

# Redémarrer le service
Restart-Service -Name CcmExec -Force
```

### 3.6 Forcer la remontée dans la console

```powershell
# Heartbeat
Invoke-WmiMethod -Namespace root\ccm -Class SMS_Client -Name TriggerSchedule -ArgumentList "{00000000-0000-0000-0000-000000000021}"
```

### 3.7 Vérifier le Management Point

Dans un navigateur sur la VM cliente :
```
http://192.168.223.129/sms_mp/.sms_aut?mplist
```
Doit retourner un XML avec les infos du site ✅

### ✅ Checklist Phase 3

- [ ] VM cliente jointe au domaine sccm.lab
- [ ] Connectivité réseau OK (ports 80, 443, 445)
- [ ] Client SCCM installé (return code 0)
- [ ] Service CcmExec = Running
- [ ] AssignedSiteCode = P05
- [ ] VM visible dans Actifs et Conformité → Appareils
- [ ] Client = Oui ✅

---

## Phase 4 — Packaging & Déploiement d'applications

### 4.1 Organiser les sources

```powershell
# Créer l'arborescence des sources sur le serveur
New-Item -Path "C:\Sources\Applications\7-Zip" -ItemType Directory -Force
New-Item -Path "C:\Sources\Applications\VLC" -ItemType Directory -Force
New-Item -Path "C:\Sources\Applications\Chrome" -ItemType Directory -Force
```

### 4.2 Créer une application MSI (exemple : 7-Zip)

1. **Bibliothèque de logiciels → Gestion des applications → Applications**
2. Clic droit → **Créer une application**
3. Type : **Windows Installer (fichier \*.msi)**
4. Emplacement : `\\WIN-UI39PMJB67B\Sources\Applications\7-Zip\7z2407-x64.msi`

**Commandes MSI standards :**
```
Installation    : msiexec /i "7z2407-x64.msi" /qn /norestart
Désinstallation : msiexec /x {PRODUCT_CODE} /qn /norestart
```

### 4.3 Distribuer le contenu

1. Clic droit sur l'application → **Distribuer du contenu**
2. Sélectionner le Distribution Point
3. Vérifier dans **Surveillance → État de distribution → État du contenu**

### 4.4 Déployer l'application

| Mode | Description | Usage |
|------|-------------|-------|
| **Disponible** | L'utilisateur installe depuis le Centre logiciel | Tests, apps optionnelles |
| **Obligatoire** | SCCM installe automatiquement | Apps obligatoires, sécurité |

### 4.5 Forcer la mise à jour des stratégies sur le client

```powershell
# Télécharger la stratégie machine
Invoke-WmiMethod -Namespace root\ccm -Class SMS_Client -Name TriggerSchedule -ArgumentList "{00000000-0000-0000-0000-000000000021}"

# Évaluation des déploiements
Invoke-WmiMethod -Namespace root\ccm -Class SMS_Client -Name TriggerSchedule -ArgumentList "{00000000-0000-0000-0000-000000000108}"
```

### 4.6 Vérifier dans le Centre logiciel

```powershell
Start-Process "C:\Windows\CCM\SCClient.exe"
```

### ✅ Checklist Phase 4

- [ ] Arborescence C:\Sources créée
- [ ] Application 7-Zip créée dans SCCM
- [ ] Contenu distribué sur le DP
- [ ] Déploiement créé (Disponible)
- [ ] Application visible dans le Centre logiciel
- [ ] Installation réussie ✅
- [ ] Tester un déploiement Obligatoire

---

## Phase 5 — Fonctionnalités avancées

### 5.1 OSD — Operating System Deployment

- Créer une Task Sequence de déploiement OS
- Configurer PXE sur le Distribution Point
- Créer des images WIM personnalisées

### 5.2 Compliance Settings

- Créer des éléments de configuration
- Vérifier des paramètres registre
- Remédiation automatique

### 5.3 Software Update Point

- Configurer WSUS avec SCCM
- Créer des groupes de mises à jour
- Déployer les patches Windows

### 5.4 App-V

- Séquencer une application avec l'App-V Sequencer (ADK)
- Déployer via SCCM

### 5.5 CMPivot

```kusto
// Exemple — lister les apps installées sur tous les clients
InstalledSoftware
| where Publisher contains "Microsoft"
| summarize count() by ProductName
```

---

## 🔗 Liens de téléchargement

| Composant | Version | Lien |
|-----------|---------|------|
| **SQL Server 2019 Developer** | Gratuit pour lab | https://www.microsoft.com/fr-fr/sql-server/sql-server-downloads |
| **SSMS** | Dernière version | https://aka.ms/ssmsfullsetup |
| **ADK Windows 11 24H2** | Obligatoire pour SCCM 2509 | https://aka.ms/adk |
| **ADK WinPE Add-on** | Obligatoire pour OSD | https://aka.ms/adkwinpe |
| **SCCM Baseline 2403** | Point de départ | https://www.microsoft.com/en-us/evalcenter/download-microsoft-endpoint-configuration-manager |
| **7-Zip MSI** | Exemple packaging | https://www.7-zip.org/download.html |
| **VLC MSI** | Exemple packaging | https://www.videolan.org/vlc/download-windows.html |
| **CMTrace** | Lecteur de logs SCCM | Inclus dans le media SCCM → `SMSSETUP\TOOLS\CMTrace.exe` |

---

## 🔧 Résolution des problèmes courants

### ❌ Client = Non dans la console après installation

```powershell
# Forcer le site code dans le registre
reg add "HKLM\SOFTWARE\Microsoft\SMS\Mobile Client" /v "AssignedSiteCode" /t REG_SZ /d "P05" /f
Restart-Service -Name CcmExec -Force
```

### ❌ Erreur "Ressources non attribuées à aucun site" lors du push

**Cause** : La VM cliente n'est pas dans les Boundaries  
**Solution** : Créer une limite IP `192.168.223.1 - 192.168.223.254` et l'ajouter au Boundary Group

### ❌ AssignedSiteCode vide

```powershell
# Vérifier que le Management Point répond
# Dans un navigateur sur la VM cliente :
# http://192.168.223.129/sms_mp/.sms_aut?mplist
# Doit retourner un XML ✅

# Forcer via registre
reg add "HKLM\SOFTWARE\Microsoft\SMS\Mobile Client" /v "AssignedSiteCode" /t REG_SZ /d "P05" /f
Restart-Service CcmExec -Force
```

### ❌ Erreur collation SQL lors de l'installation SCCM

**Cause** : La collation SQL n'est pas `SQL_Latin1_General_CP1_CI_AS`  
**Solution** : Réinstaller SQL Server avec la bonne collation (impossible de changer après)

### ❌ CcmSetup return code != 0

```powershell
# Lire les dernières lignes du log
Get-Content "C:\Windows\ccmsetup\Logs\ccmsetup.log" -Tail 30

# Désinstaller et réinstaller proprement
& "C:\Windows\CCM\ccmsetup.exe" /uninstall
Start-Sleep -Seconds 120
\\192.168.223.129\SMS_P05\Client\ccmsetup.exe /mp:WIN-UI39PMJB67B.sccm.lab SMSSITECODE=P05
```

### ❌ Message "Fin du support atteinte pour la version du site"

**Cause** : Tu es encore en version baseline 2403  
**Solution** : **Administration → Mises à jour et maintenance** → installer 2409 → 2503 → 2509

---

## 📁 Logs importants à connaître

| Log | Emplacement | Utilité |
|-----|-------------|---------|
| `ccmsetup.log` | `C:\Windows\ccmsetup\Logs\` | Installation du client |
| `ClientIDManagerStartup.log` | `C:\Windows\CCM\Logs\` | Enregistrement client |
| `LocationServices.log` | `C:\Windows\CCM\Logs\` | Localisation du site |
| `AppDiscovery.log` | `C:\Windows\CCM\Logs\` | Détection des apps |
| `AppEnforce.log` | `C:\Windows\CCM\Logs\` | Installation des apps |
| `mpcontrol.log` | `C:\Program Files\Microsoft Configuration Manager\Logs\` | Santé du Management Point |
| `ccm.log` | `C:\Program Files\Microsoft Configuration Manager\Logs\` | Client push |
| `ConfigMgrSetup.log` | `C:\` | Installation SCCM |

> 💡 **Astuce** : Utilise **CMTrace.exe** pour lire les logs SCCM — il colore les erreurs en rouge et les warnings en jaune, beaucoup plus lisible que Notepad !

---

## 🎓 Ressources pour aller plus loin

- 📚 [Documentation officielle Microsoft ConfigMgr](https://docs.microsoft.com/fr-fr/mem/configmgr/)
- 📚 [PrajwalDesai — Tutoriels SCCM](https://www.prajwaldesai.com/sccm/)
- 📚 [System Center Dudes](https://www.systemcenterdudes.com/)
- 📚 [HTMD Blog](https://www.anoopcnair.com/)
- 🎥 [Microsoft Learn — ConfigMgr](https://learn.microsoft.com/fr-fr/training/browse/?products=configuration-manager)

---

## 📊 Versions utilisées dans ce lab

| Composant | Version |
|-----------|---------|
| Windows Server | 2022 |
| Active Directory | Windows Server 2022 |
| SQL Server | 2019 |
| ADK | Windows 11 24H2 |
| SCCM / ConfigMgr | 2509 |
| VM Cliente OS | Windows 11 |
| Site Code | P05 |
| Domaine | sccm.lab |

---

*Lab réalisé le 05/05/2026 — Guide rédigé pas à pas lors de l'installation*
