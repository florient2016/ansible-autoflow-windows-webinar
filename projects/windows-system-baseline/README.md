# Projet 3 — Windows System Baseline 🖥️

## Description

Ce projet configure une **base système minimale et sécurisée** sur un serveur Windows. Il est conçu comme référence de configuration initiale pour tout nouveau serveur intégré à votre parc géré par Ansible.

## Ce que fait ce playbook

1. **Utilisateur local** — Création de `automation_admin` (compte de service Ansible)
2. **Répertoire de travail** — `C:\Automation` avec sous-dossiers structurés
3. **Permissions** — ACL sur le répertoire d'automatisation
4. **Chocolatey** — Installation/vérification du gestionnaire de paquets Windows
5. **Outils de base** — Installation de git, notepad++, 7zip via Chocolatey
6. **Firewall** — Règles pour ICMP, WinRM HTTP/HTTPS, RDP
7. **Rapport système** — Fichier `C:\Automation\reports\system-report.txt`

## Structure du Rôle

```
roles/system_baseline/
├── tasks/
│   └── main.yml                    # Orchestration des étapes
├── templates/
│   └── system-report.txt.j2        # Template du rapport système
├── defaults/
│   └── main.yml                    # Variables configurables
└── vars/
    └── main.yml                    # Variables techniques fixes
```

## Variables Configurables

| Variable | Défaut | Description |
|---|---|---|
| `baseline_admin_username` | `automation_admin` | Nom du compte de service |
| `baseline_admin_password` | `Ansible@Webinar2024!` | ⚠️ À changer ! Utiliser Vault |
| `baseline_automation_folder` | `C:\Automation` | Répertoire de travail |
| `baseline_install_chocolatey` | `true` | Installer Chocolatey |
| `baseline_update_chocolatey` | `false` | Mettre à jour Chocolatey |
| `baseline_choco_packages` | `[git, notepadplusplus, 7zip]` | Paquets à installer |
| `baseline_enable_rdp_firewall` | `true` | Règle firewall RDP |

## Sécurité — Gestion du mot de passe

> ⚠️ **IMPORTANT** : Ne jamais stocker de mot de passe en clair dans git !

### Option 1 : Ansible Vault (usage local)
```bash
# Chiffrer le mot de passe
ansible-vault encrypt_string 'MonMotDePasse123!' --name 'baseline_admin_password'

# Résultat à coller dans votre fichier de variables
baseline_admin_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
```

### Option 2 : AAP Credentials (recommandé en production)
Définir `baseline_admin_password` comme variable dans les **Custom Credentials** ou via les **Extra Variables** du Job Template dans AAP.

## Exécution locale (test)

```bash
# Installation standard
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-system-baseline/playbook.yml

# Avec mot de passe chiffré
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-system-baseline/playbook.yml \
  --ask-vault-pass

# Personnalisation des paquets
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-system-baseline/playbook.yml \
  -e "baseline_choco_packages=['git','vscode','python']"
```

## Configuration dans AAP

### Job Template

| Paramètre | Valeur |
|---|---|
| **Name** | `03 - Windows System Baseline` |
| **Job Type** | Run |
| **Inventory** | `Windows Dev Inventory` |
| **Project** | `AAP Windows Webinar` |
| **Playbook** | `projects/windows-system-baseline/playbook.yml` |
| **Credentials** | `Windows Machine Credential` |
| **Verbosity** | `1 (Verbose)` |

### Extra Variables recommandées

```yaml
baseline_admin_username: "ansible_svc"
baseline_automation_folder: "C:\\Automation"
baseline_choco_packages:
  - git
  - notepadplusplus
  - 7zip
baseline_enable_rdp_firewall: true
```

## Résultat Attendu

Après exécution :
- Compte `automation_admin` créé et membre du groupe Administrateurs
- Dossier `C:\Automation\` avec sous-dossiers `logs/`, `scripts/`, `reports/`, `backups/`
- Chocolatey installé
- Git, Notepad++, 7-Zip installés
- 4 règles firewall Ansible créées
- Rapport généré dans `C:\Automation\reports\system-report.txt`

## Dépannage

| Erreur | Cause | Solution |
|---|---|---|
| Chocolatey timeout | Téléchargement lent | Augmenter le timeout dans ansible.cfg |
| `win_acl` permission denied | Compte insuffisant | Vérifier que le compte est Administrateur |
| Package Chocolatey introuvable | Nom incorrect | Vérifier sur community.chocolatey.org |

Voir aussi [troubleshooting.md](../../docs/troubleshooting.md)
