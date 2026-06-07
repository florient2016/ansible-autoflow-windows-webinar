# Projet 2 — Windows IIS Management 🌐

## Description

Ce projet gère le **cycle de vie complet d'IIS** (Internet Information Services) sur Windows Server. Il est conçu pour être idempotent et démonstratif : il désinstalle IIS s'il est présent, puis le réinstalle proprement avec une page web personnalisée.

## Ce que fait ce playbook

1. **Vérification** — État actuel d'IIS (installé ou non)
2. **Arrêt** — Arrêt propre du service W3SVC si actif
3. **Désinstallation** — Suppression complète d'IIS (si `iis_force_reinstall: true`)
4. **Installation** — Installation propre avec les composants Web essentiels
5. **Redémarrage** — Si nécessaire après installation des fonctionnalités Windows
6. **Démarrage** — Lancement du service W3SVC en mode automatique
7. **Déploiement** — Page HTML personnalisée via template Jinja2
8. **Configuration** — Port et liaison du site via appcmd
9. **Validation** — Test HTTP pour confirmer que le site répond

## Structure du Rôle

```
roles/iis_management/
├── tasks/
│   └── main.yml          # Orchestration des étapes
├── templates/
│   └── index.html.j2     # Template HTML de la page web
├── defaults/
│   └── main.yml          # Variables configurables (priorité basse)
└── vars/
    └── main.yml          # Variables techniques fixes (priorité haute)
```

## Variables Configurables

| Variable | Défaut | Description |
|---|---|---|
| `iis_site_name` | `Default Web Site` | Nom du site dans IIS |
| `iis_site_port` | `80` | Port HTTP d'écoute |
| `iis_site_root` | `C:\inetpub\wwwroot` | Répertoire racine |
| `iis_page_title` | `🚀 Ansible AAP — Webinar Windows` | Titre de la page HTML |
| `iis_page_message` | *(voir defaults)* | Message principal de la page |
| `iis_force_reinstall` | `true` | Désinstaller puis réinstaller IIS |

## Surcharger les Variables dans AAP

Dans le Job Template, section **Extra Variables** :

```yaml
iis_site_name: "MonApplication"
iis_site_port: 8080
iis_page_title: "Mon Application d'Entreprise"
iis_page_message: "Bienvenue sur notre portail !"
iis_force_reinstall: false
```

## Exécution locale (test)

```bash
# Installation standard
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-iis-management/playbook.yml

# Avec variables personnalisées
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-iis-management/playbook.yml \
  -e "iis_page_title='Mon Site' iis_site_port=8080"

# Mode dry-run (vérifier sans appliquer)
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-iis-management/playbook.yml \
  --check --diff
```

## Configuration dans AAP

### Job Template

| Paramètre | Valeur |
|---|---|
| **Name** | `02 - Windows IIS Management` |
| **Job Type** | Run |
| **Inventory** | `Windows Dev Inventory` |
| **Project** | `AAP Windows Webinar` |
| **Playbook** | `projects/windows-iis-management/playbook.yml` |
| **Credentials** | `Windows Machine Credential` |
| **Verbosity** | `1 (Verbose)` |

## Prérequis

- Windows Server avec accès administrateur
- WinRM configuré (HTTP port 5985 ou HTTPS 5986)
- Collection `ansible.windows` dans l'EE
- Aucune dépendance externe (IIS est une fonctionnalité Windows intégrée)

## Résultat Attendu

Après exécution, accéder à `http://<IP_DU_SERVEUR>` affiche la page web personnalisée déployée par Ansible.

## Dépannage

| Erreur | Cause probable | Solution |
|---|---|---|
| `win_feature` échoue | Droits insuffisants | Vérifier que le compte est Administrateur local |
| `win_uri` timeout | IIS démarre lentement | Augmenter `iis_http_check_timeout` |
| Page 403 Forbidden | Permissions sur wwwroot | Vérifier les ACL sur `C:\inetpub\wwwroot` |

Voir aussi [troubleshooting.md](../../docs/troubleshooting.md)
