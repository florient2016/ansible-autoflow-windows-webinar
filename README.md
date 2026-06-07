# 🚀 Ansible Automation Platform — Webinar Windows Server

> **Repository de démonstration** pour le webinar Ansible Automation Platform  
> Automatisation Windows Server avec 3 projets opérationnels

---

## 🎯 But du Webinar

Ce repository illustre comment **Ansible Automation Platform (AAP)** peut gérer des serveurs Windows à travers 3 projets progressifs :

| # | Projet | Description |
|---|--------|-------------|
| 1 | `windows-ping-test` | Test de connectivité WinRM |
| 2 | `windows-iis-management` | Gestion complète d'IIS |
| 3 | `windows-system-baseline` | Configuration baseline système |

---

## 🏗️ Architecture du Repository

```
ansible-aap-windows-webinar/
├── README.md                          # Ce fichier
├── ansible.cfg                        # Configuration Ansible globale
├── requirements.yml                   # Collections Ansible requises
├── .gitignore
│
├── inventories/
│   └── dev/
│       ├── hosts.yml                  # Inventaire des hôtes Windows
│       └── group_vars/
│           └── windows.yml            # Variables globales WinRM
│
├── projects/
│   ├── windows-ping-test/             # Projet 1 : Test connectivité
│   ├── windows-iis-management/        # Projet 2 : Gestion IIS
│   └── windows-system-baseline/       # Projet 3 : Baseline système
│
├── controller/
│   ├── job-templates/                 # Définitions des Job Templates AAP
│   └── workflow-template.yml          # Workflow orchestrant les 3 projets
│
└── docs/
    ├── webinar-scenario.md            # Scénario pas à pas
    ├── setup-winrm.md                 # Configuration WinRM côté Windows
    ├── aap-configuration.md           # Configuration dans AAP
    └── troubleshooting.md             # Guide de dépannage
```

---

## 📋 Prérequis

### Côté Ansible Automation Platform
- Ansible Automation Platform 2.x ou supérieur
- Automation Controller (anciennement AWX/Tower)
- Accès administrateur à AAP
- Execution Environment avec les collections Windows

### Côté Windows Server
- Windows Server 2016 / 2019 / 2022
- PowerShell 5.1 ou supérieur
- WinRM activé et configuré (voir [docs/setup-winrm.md](docs/setup-winrm.md))
- Compte avec droits administrateur local

### Collections Ansible requises
```yaml
# requirements.yml — installées automatiquement par AAP
collections:
  - ansible.windows
  - community.windows
  - chocolatey.chocolatey
```

---

## 🔧 Configuration WinRM (Windows Server)

> Documentation complète : [docs/setup-winrm.md](docs/setup-winrm.md)

### Configuration rapide (PowerShell — sur le serveur Windows)

```powershell
# Activer WinRM
winrm quickconfig -Force

# Autoriser l'authentification Basic (pour les tests — NTLM/Kerberos en production)
winrm set winrm/config/service/auth '@{Basic="true"}'

# Autoriser les connexions non chiffrées (HTTP — pour lab uniquement)
winrm set winrm/config/service '@{AllowUnencrypted="true"}'

# Ouvrir le pare-feu
netsh advfirewall firewall add rule name="WinRM HTTP" protocol=TCP dir=in localport=5985 action=allow

# Vérifier
winrm enumerate winrm/config/listener
```

> ⚠️ **En production** : utilisez HTTPS (port 5986) avec un certificat valide et l'authentification Kerberos ou NTLM.

---

## ⚙️ Configuration dans Ansible Automation Platform

> Documentation complète : [docs/aap-configuration.md](docs/aap-configuration.md)

### 1. Importer le Repository GitHub

1. Dans AAP → **Projects** → **Add**
2. **Source Control Type** : Git
3. **Source Control URL** : `https://github.com/<votre-org>/ansible-aap-windows-webinar`
4. **Source Control Branch** : `main`
5. Cocher **Update Revision on Launch**
6. Sauvegarder et lancer une synchronisation

### 2. Créer les Credentials Windows

1. **Credentials** → **Add**
2. **Credential Type** : Machine
3. Renseigner :
   - **Username** : `Administrator` (ou compte de service)
   - **Password** : mot de passe du compte Windows
4. **NE PAS** stocker les credentials dans les fichiers YAML du repository

### 3. Créer l'Inventaire

1. **Inventories** → **Add**
2. Ajouter le groupe `windows`
3. Ajouter l'hôte cible avec les variables :
```yaml
ansible_connection: winrm
ansible_winrm_transport: ntlm    # ou basic pour les tests
ansible_winrm_port: 5985
ansible_winrm_scheme: http       # ou https en production
ansible_winrm_server_cert_validation: ignore  # lab uniquement
```

### 4. Créer les Job Templates

| Job Template | Playbook | Credential |
|---|---|---|
| 01 - Windows Ping Test | `projects/windows-ping-test/playbook.yml` | Windows Credential |
| 02 - Windows IIS Management | `projects/windows-iis-management/playbook.yml` | Windows Credential |
| 03 - Windows System Baseline | `projects/windows-system-baseline/playbook.yml` | Windows Credential |

### 5. Créer le Workflow Template

1. **Templates** → **Add** → **Add workflow template**
2. Nommer : `Windows Server — Full Deployment Workflow`
3. Dans le Workflow Visualizer, chaîner les 3 Job Templates :
   ```
   [01 Ping Test] → (On Success) → [02 IIS Management] → (On Success) → [03 System Baseline]
   ```

---

## ▶️ Exécution des Projets

### Projet 1 — Test de Connectivité

```bash
# En ligne de commande (pour les tests locaux)
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-ping-test/playbook.yml \
  -e "ansible_user=Administrator" \
  -e "ansible_password=YourPassword"
```

**Dans AAP** : Lancer le Job Template `01 - Windows Ping Test`

---

### Projet 2 — Gestion IIS

```bash
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-iis-management/playbook.yml
```

**Dans AAP** : Lancer le Job Template `02 - Windows IIS Management`

Variables personnalisables via **Extra Variables** dans AAP :
```yaml
iis_site_name: "MonSite"
iis_site_port: 80
iis_page_title: "Mon Application"
iis_page_message: "Déployé par Ansible !"
```

---

### Projet 3 — System Baseline

```bash
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-system-baseline/playbook.yml
```

**Dans AAP** : Lancer le Job Template `03 - Windows System Baseline`

Variables personnalisables :
```yaml
baseline_admin_username: "automation_admin"
baseline_admin_password: "ChangeMe123!"
baseline_automation_folder: "C:\\Automation"
baseline_install_chocolatey: true
baseline_choco_packages:
  - git
  - notepadplusplus
  - 7zip
```

---

### Workflow Complet

Lancer le **Workflow Template** `Windows Server — Full Deployment Workflow` pour exécuter les 3 projets dans l'ordre avec gestion des succès/échecs.

---

## 🔐 Sécurité

- ❌ Ne jamais stocker de mots de passe en clair dans le repository
- ✅ Utiliser les **Credentials** d'Automation Controller
- ✅ Utiliser **Ansible Vault** pour les variables sensibles en dehors d'AAP
- ✅ Préférer **HTTPS + Kerberos** en environnement de production
- ✅ Appliquer le **principe du moindre privilège** sur les comptes de service

---

## 📚 Documentation

| Document | Description |
|---|---|
| [docs/webinar-scenario.md](docs/webinar-scenario.md) | Scénario complet du webinar |
| [docs/setup-winrm.md](docs/setup-winrm.md) | Configuration WinRM détaillée |
| [docs/aap-configuration.md](docs/aap-configuration.md) | Guide de configuration AAP |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Résolution des problèmes courants |

---

## 🤝 Contribution

Ce repository est à but pédagogique. N'hésitez pas à l'adapter à votre environnement.

---

*Webinar Ansible Automation Platform — Windows Server Automation*
