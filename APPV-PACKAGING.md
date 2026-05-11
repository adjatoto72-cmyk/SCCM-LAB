# 📦 Packaging App-V avec SCCM — Notepad++

> **Objectif** : Séquencer une application avec l'App-V Sequencer et la déployer via SCCM  
> **Application exemple** : Notepad++ 8.6.2  
> **Durée estimée** : ~1h

---

## 📋 Table des matières

- [Concept App-V](#concept-app-v)
- [Prérequis](#prérequis)
- [Étape 1 — Préparer la VM de séquençage](#étape-1--préparer-la-vm-de-séquençage)
- [Étape 2 — Installer l'App-V Sequencer](#étape-2--installer-lapp-v-sequencer)
- [Étape 3 — Séquencer Notepad++](#étape-3--séquencer-notepad)
- [Étape 4 — Importer dans SCCM](#étape-4--importer-dans-sccm)
- [Étape 5 — Déployer sur les clients](#étape-5--déployer-sur-les-clients)
- [Résolution des problèmes](#résolution-des-problèmes)

---

## 💡 Concept App-V

App-V (Application Virtualization) permet de **virtualiser des applications** — elles tournent dans une bulle isolée sans s'installer vraiment sur le système.

```
Application réelle          Package App-V
┌─────────────┐            ┌─────────────────────┐
│ Setup.exe   │  Sequencer │  .appv               │
│ Registry    │ ─────────► │  VFS (Virtual FS)   │
│ DLL         │            │  Registry virtuel   │
│ Config      │            │  Config isolée      │
└─────────────┘            └─────────────────────┘
                                     │
                                     ▼ SCCM déploie
                            VM Cliente (sans install)
```

**Avantages App-V :**
- ✅ Pas de conflits entre applications
- ✅ Désinstallation propre et complète
- ✅ Isolation totale du système
- ✅ Déploiement rapide via SCCM

---

## 🔧 Prérequis

| Composant | Détail |
|-----------|--------|
| **VM de séquençage** | Windows 10/11 propre et dédiée |
| **App-V Sequencer** | Inclus dans l'ADK Windows 11 24H2 |
| **Application à séquencer** | Notepad++ x64 .exe |
| **SCCM** | 2509 avec Distribution Point configuré |

> ⚠️ **Règle d'or** : toujours séquencer sur une **VM dédiée propre** — jamais sur ta VM de production ou ta VM TEST.

---

## Étape 1 — Préparer la VM de séquençage

### 1.1 Cloner une VM Windows 11 propre

Dans VMware :
1. Clic droit sur ta VM Windows 11 → **Gérer → Cloner**
2. Nomme le clone `SEQUENCER`
3. Lance la VM clonée

### 1.2 Générer une nouvelle adresse MAC

> ⚠️ Le clone a la même MAC que la VM source — cela cause des conflits réseau

1. **Éteins la VM clonée**
2. VMware → Paramètres → **Carte réseau → Avancé**
3. Clique **Générer** pour créer une nouvelle MAC unique
4. Redémarre la VM

### 1.3 Configurer l'IP fixe

| Paramètre | Valeur |
|-----------|--------|
| **Adresse IP** | `192.168.223.131` |
| **Masque** | `255.255.255.0` |
| **Passerelle** | `192.168.223.2` |
| **DNS** | `192.168.223.129` |

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.223.131 -PrefixLength 24 -DefaultGateway 192.168.223.2
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.223.129

# Vérifier la connectivité
ping 192.168.223.129
```

### 1.4 Renommer et joindre au domaine

```powershell
# Renommer la machine
Rename-Computer -NewName "SEQUENCER" -DomainCredential sccm\Administrateur -Restart

# Après redémarrage — joindre au domaine si nécessaire
$password = ConvertTo-SecureString "TonMotDePasse" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("sccm\Administrateur", $password)
Add-Computer -DomainName "sccm.lab" -Credential $cred -Restart
```

### 1.5 ⚠️ Prendre un snapshot AVANT le séquençage

1. **Éteins la VM SEQUENCER**
2. VMware → clic droit → **Snapshot → Prendre un snapshot**
3. Nomme-le : `CLEAN - Avant sequençage`
4. Redémarre la VM

> 💡 Ce snapshot te permet de revenir à un état propre en 30 secondes pour chaque nouvelle application.

---

## Étape 2 — Installer l'App-V Sequencer

L'App-V Sequencer est inclus dans l'**ADK Windows 11 24H2**.

### Option 1 — Installer l'ADK sur la VM SEQUENCER

Télécharge l'ADK : 👉 https://aka.ms/adk

Lance `adksetup.exe` et coche **uniquement** :
- ✅ Microsoft Application Virtualization (App-V) Sequencer

### Option 2 — Copier depuis le serveur SCCM

```powershell
Copy-Item "\\192.168.223.129\C$\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Microsoft Application Virtualization" -Destination "C:\AppV" -Recurse
```

### Lancer le Sequencer

```
Démarrer → Microsoft Application Virtualization Sequencer
```

---

## Étape 3 — Séquencer Notepad++

### 3.1 Télécharger Notepad++

👉 https://notepad-plus-plus.org/downloads/

Sauvegarde dans `C:\Sources\NotepadPlusPlus\` sur la VM SEQUENCER.

### 3.2 Lancer le séquençage

Dans l'App-V Sequencer :

**1 — Packaging Method**
- Clique **Create a New Virtual Application Package**
- Sélectionne **Create Package (default)**
- Clique **Next**

**2 — Prepare Computer**
- Résous les warnings éventuels → clique **Resolve**
- Une fois tout en vert → **Next**

**3 — Type of Application**
- Sélectionne **Standard Application** ✅
- Clique **Next**

**4 — Select Installer**
- Clique **Browse**
- Sélectionne `npp.x.x.x.Installer.x64.exe`
- Clique **Next**

**5 — Package Name**
```
Package Name : Notepad++
```
> Le Primary Virtual Application Directory se remplit automatiquement ✅

**6 — Installation**
- Le setup Notepad++ se lance automatiquement
- Installe normalement
- ⚠️ **Ne lance PAS l'application après installation**
- ⚠️ Décoche toute option de lancement automatique
- Ferme l'installeur → reviens dans le Sequencer → **Next**

**7 — Configure Software**
- Clique **Run** (pas Run All !) sur Notepad++ dans la liste
- L'application s'ouvre dans l'environnement virtuel
- Ferme la fenêtre de bienvenue
- Teste rapidement (ouvrir un fichier, menus)
- **Ferme complètement Notepad++**
- Clique **Next** dans le Sequencer

**8 — Installation Report**
- Vérifie qu'il n'y a pas d'erreurs critiques
- Clique **Next**

**9 — Customize**
- Laisse par défaut
- Clique **Next**

**10 — Create Package**
- Sélectionne **Save the package now** ✅
- Save Location : `C:\Users\Administrateur.SCCM\Desktop\Notepad++\`
- Ajoute une description optionnelle : `Notepad++ 8.6.2 - Package App-V`
- Clique **Create**

**11 — Completion**
- ✅ **Package completed**
- Warning "Files excluded from package" → **normal**, pas grave
- Clique **Close**

### 3.3 Fichiers générés

```
C:\Users\Administrateur.SCCM\Desktop\Notepad++\
├── Notepad++.appv      ← Package principal ✅
├── Notepad++.msi       ← Optionnel
└── [Content_Types].xml ← Métadonnées
```

---

## Étape 4 — Importer dans SCCM

### 4.1 Copier le .appv sur le serveur SCCM

```powershell
# Créer le dossier de destination
New-Item -Path "\\192.168.223.129\C$\Sources\AppV\NotepadPlusPlus" -ItemType Directory -Force

# Copier le package
Copy-Item "C:\Users\Administrateur.SCCM\Desktop\Notepad++\Notepad++.appv" `
          -Destination "\\192.168.223.129\C$\Sources\AppV\NotepadPlusPlus\"
```

Ou via **Explorateur Windows** → glisse-dépose dans `\\192.168.223.129\C$\Sources\AppV\NotepadPlusPlus\`

### 4.2 Créer l'application dans la console SCCM

1. **Bibliothèque de logiciels → Gestion des applications → Applications**
2. Clic droit → **Créer une application**
3. Type : **Microsoft Application Virtualization**
4. Emplacement :
```
\\WIN-UI39PMJB67B\C$\Sources\AppV\NotepadPlusPlus\Notepad++.appv
```
5. SCCM remplit automatiquement les métadonnées ✅
6. Clique **Next → Next → Fermer**

### 4.3 Distribuer le contenu

1. Clic droit sur **Notepad++** → **Distribuer du contenu**
2. Ajoute le **Distribution Point** → **Next → Fermer**
3. Vérifie dans **Surveillance → État de distribution → État du contenu**
4. Attends le statut **Succès** ✅

---

## Étape 5 — Déployer sur les clients

### 5.1 Créer le déploiement

1. Clic droit sur **Notepad++** → **Déployer**
2. Regroupement : **Lab - Machines Test**
3. Mode : **Disponible** (Centre logiciel) ou **Obligatoire** (automatique)
4. Clique **Next → Fermer**

### 5.2 Tester sur la VM TEST

```powershell
# Forcer la mise à jour des stratégies
Invoke-WmiMethod -Namespace root\ccm -Class SMS_Client -Name TriggerSchedule `
    -ArgumentList "{00000000-0000-0000-0000-000000000021}"

# Ouvrir le Centre logiciel
Start-Process "C:\Windows\CCM\SCClient.exe"
```

Notepad++ doit apparaître dans le Centre logiciel → clique **Installer** ✅

---

## 🔧 Résolution des problèmes

### ❌ Impossible de renommer la VM clonée

**Cause** : La VM ne peut pas contacter le contrôleur de domaine (conflit IP/MAC)  
**Solution** :
1. Changer la MAC d'abord (VMware → Avancé → Générer)
2. Configurer la bonne IP
3. Vérifier le ping vers le serveur
4. Réessayer le renommage

### ❌ Pas de connectivité réseau sur le clone

**Cause** : Conflit d'adresse MAC avec la VM source  
**Solution** : Générer une nouvelle MAC dans VMware → Paramètres → Carte réseau → Avancé → Générer

### ❌ Feature "Microsoft-AppV-Sequencer" introuvable

**Cause** : Sur Windows 11, le Sequencer vient de l'ADK et non de Windows  
**Solution** : Installer l'ADK et cocher uniquement "App-V Sequencer"

### ❌ Warning "Files excluded from package"

**C'est normal** — le Sequencer exclut les fichiers temporaires et système inutiles. Le package fonctionne parfaitement. ✅

### ❌ Notepad++ ne s'installe pas depuis le Centre logiciel

```powershell
# Vérifier les logs de déploiement App-V
Get-Content "C:\Windows\CCM\Logs\AppDiscovery.log" -Tail 20
Get-Content "C:\Windows\CCM\Logs\AppEnforce.log" -Tail 20
```

---

## 📊 Arborescence des sources recommandée

```
C:\Sources\
├── Applications\
│   ├── 7-Zip\
│   │   └── 7z2407-x64.msi
│   ├── VLC\
│   └── Chrome\
└── AppV\
    ├── NotepadPlusPlus\
    │   └── Notepad++.appv
    └── [autres apps App-V]
```

---

## 🔗 Liens utiles

| Ressource | Lien |
|-----------|------|
| **Notepad++ Downloads** | https://notepad-plus-plus.org/downloads/ |
| **ADK Windows 11 24H2** | https://aka.ms/adk |
| **ADK WinPE Add-on** | https://aka.ms/adkwinpe |
| **Doc App-V Microsoft** | https://docs.microsoft.com/fr-fr/windows/application-management/app-v/appv-getting-started |

---

## ✅ Checklist complète

- [ ] VM de séquençage clonée et propre
- [ ] Nouvelle MAC générée sur le clone
- [ ] IP fixe configurée (192.168.223.131)
- [ ] Ping vers le serveur SCCM OK
- [ ] VM jointe au domaine sccm.lab
- [ ] Snapshot `CLEAN - Avant séquençage` pris
- [ ] App-V Sequencer installé
- [ ] Notepad++ séquencé (return code 0)
- [ ] Fichier .appv généré
- [ ] .appv copié sur le serveur SCCM
- [ ] Application créée dans SCCM
- [ ] Contenu distribué sur le DP
- [ ] Déploiement créé
- [ ] Notepad++ visible dans le Centre logiciel ✅
- [ ] Installation réussie sur la VM TEST ✅

---

*Guide réalisé le 09/05/2026 — Packaging App-V Notepad++ via SCCM 2509*
