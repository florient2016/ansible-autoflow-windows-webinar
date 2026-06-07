# Configuration Ansible Automation Platform ⚙️

Guide complet pour configurer AAP et exécuter les 3 projets du webinar.

---

## 1. Prérequis AAP

- Ansible Automation Platform 2.x installé et accessible
- Accès administrateur à l'Automation Controller
- Accès Internet depuis les Execution Nodes (pour les collections et Chocolatey)
- Execution Environment avec les collections `ansible.windows`, `community.windows`, `chocolatey.chocolatey`

---

## 2. Importer le Repository GitHub

### Via l'interface Web AAP

1. Aller dans **Projects** (menu gauche)
2. Cliquer sur **Add** (bouton bleu)
3. Remplir le formulaire :

| Champ | Valeur |
|---|---|
| **Name** | `AAP Windows Webinar` |
| **Description** | Repository webinar AAP Windows Server |
| **Organization** | Default (ou votre organisation) |
| **Source Control Type** | Git |
| **Source Control URL** | `https://github.com/<votre-org>/ansible-aap-windows-webinar` |
| **Source Control Branch/Tag/Commit** | `main` |
| **Source Control Credential** | *(laisser vide si repository public)* |
| **Options** | ✅ Clean, ✅ Update Revision on Launch |

4. Cliquer **Save**
5. Cliquer le bouton **Sync** (icône rafraîchissement) pour synchroniser le repository
6. Vérifier que le statut passe à **Successful** (barre verte)

### Vérification

Après synchronisation, dans **Projects > AAP Windows Webinar**, vous devez voir :
- Status: **Successful**
- SCM Revision: hash du dernier commit
- Last Updated: date récente

---

## 3. Créer les Credentials Windows

### Credential Machine (connexion WinRM)

1. Aller dans **Credentials** → **Add**
2. Remplir :

| Champ | Valeur |
|---|---|
| **Name** | `Windows Machine Credential` |
| **Description** | Credential WinRM pour les serveurs Windows |
| **Organization** | Default |
| **Credential Type** | Machine |
| **Username** | `Administrator` (ou votre compte de service) |
| **Password** | *(mot de passe du compte)* |

3. Cliquer **Save**

> ✅ Le mot de passe est chiffré dans AAP et n'apparaît jamais en clair dans les logs.

### Credential Source Control (si repository privé)

1. **Credentials** → **Add**
2. **Credential Type** : Source Control
3. Renseigner username + Personal Access Token GitHub

---

## 4. Créer l'Inventaire

### Créer l'inventaire

1. **Inventories** → **Add** → **Add inventory**
2. Nom : `Windows Dev Inventory`
3. Organization : Default
4. **Save**

### Ajouter le groupe Windows

1. Dans l'inventaire → **Groups** → **Add**
2. Nom : `windows`
3. **Save**

### Ajouter des variables au groupe

Dans le groupe `windows`, onglet **Variables** :

```yaml
ansible_connection: winrm
ansible_winrm_transport: ntlm
ansible_winrm_port: 5985
ansible_winrm_scheme: http
ansible_winrm_server_cert_validation: ignore
ansible_shell_type: powershell
ansible_winrm_operation_timeout_sec: 120
ansible_winrm_read_timeout_sec: 150
```

### Ajouter l'hôte Windows

1. Dans le groupe `windows` → **Hosts** → **Add** → **Add new host**
2. Remplir :

| Champ | Valeur |
|---|---|
| **Name** | `windows-server-01` |
| **Description** | Serveur Windows de démonstration |
| **Variables** | `ansible_host: 192.168.1.100` *(adapter l'IP)* |

3. **Save**

---

## 5. Créer l'Execution Environment (si nécessaire)

Si les collections Windows ne sont pas dans votre EE par défaut, créer un EE dédié.

### Option A : Utiliser un EE existant avec les collections Windows

Vérifier que votre EE contient :
```bash
ansible-galaxy collection list | grep -E "ansible.windows|community.windows|chocolatey"
```

### Option B : Créer un EE personnalisé

Fichier `execution-environment.yml` :
```yaml
---
version: 3
dependencies:
  galaxy:
    collections:
      - name: ansible.windows
      - name: community.windows
      - name: chocolatey.chocolatey
  python_interpreter:
    package_system: python3
  python:
    - pywinrm>=0.4.3
    - requests-ntlm
    - requests-credssp
```

Construire avec `ansible-builder` :
```bash
pip install ansible-builder
ansible-builder build -t my-windows-ee:latest -v 3
```

---

## 6. Créer les Job Templates

### Job Template 1 — Windows Ping Test

**Templates** → **Add** → **Add job template**

| Champ | Valeur |
|---|---|
| **Name** | `01 - Windows Ping Test` |
| **Job Type** | Run |
| **Inventory** | `Windows Dev Inventory` |
| **Project** | `AAP Windows Webinar` |
| **Playbook** | `projects/windows-ping-test/playbook.yml` |
| **Credentials** | `Windows Machine Credential` |
| **Verbosity** | 1 - Verbose |
| **Options** | ✅ Enable Privilege Escalation (si nécessaire) |

**Extra Variables** :
```yaml
ping_retries: 3
ping_delay: 5
```

### Job Template 2 — Windows IIS Management

| Champ | Valeur |
|---|---|
| **Name** | `02 - Windows IIS Management` |
| **Job Type** | Run |
| **Inventory** | `Windows Dev Inventory` |
| **Project** | `AAP Windows Webinar` |
| **Playbook** | `projects/windows-iis-management/playbook.yml` |
| **Credentials** | `Windows Machine Credential` |
| **Verbosity** | 1 - Verbose |
| **Options** | ✅ Prompt on launch (Variables) — pour personnaliser la page web |

**Extra Variables** :
```yaml
iis_site_name: "Default Web Site"
iis_site_port: 80
iis_page_title: "🚀 Ansible AAP — Webinar Windows"
iis_page_message: "IIS déployé par Ansible Automation Platform !"
iis_force_reinstall: true
```

### Job Template 3 — Windows System Baseline

| Champ | Valeur |
|---|---|
| **Name** | `03 - Windows System Baseline` |
| **Job Type** | Run |
| **Inventory** | `Windows Dev Inventory` |
| **Project** | `AAP Windows Webinar` |
| **Playbook** | `projects/windows-system-baseline/playbook.yml` |
| **Credentials** | `Windows Machine Credential` |
| **Verbosity** | 1 - Verbose |

**Extra Variables** :
```yaml
baseline_admin_username: "automation_admin"
baseline_automation_folder: "C:\\Automation"
baseline_choco_packages:
  - git
  - notepadplusplus
  - 7zip
```

---

## 7. Créer le Workflow Template

**Templates** → **Add** → **Add workflow template**

| Champ | Valeur |
|---|---|
| **Name** | `🪟 Windows Server — Full Deployment Workflow` |
| **Inventory** | `Windows Dev Inventory` |
| **Options** | ✅ Prompt on launch (Inventory, Limit) |

Après sauvegarde, cliquer **Visualizer** pour configurer le flux :

1. Cliquer **Start** → Choisir `01 - Windows Ping Test` → **Save**
2. Sur le nœud `01`, cliquer **+** (succès) → Choisir `02 - Windows IIS Management` → **Save**
3. Sur le nœud `02`, cliquer **+** (succès) → Choisir `03 - Windows System Baseline` → **Save**
4. Sur les nœuds d'échec : configurer des notifications si disponibles
5. Cliquer **Save** pour sauvegarder le workflow

---

## 8. Lancer les Jobs

### Lancement individuel

1. **Templates** → Cliquer le bouton ▶️ (Launch) à côté du Job Template
2. Confirmer l'inventaire et les variables
3. **Launch**

### Lancement du Workflow

1. **Templates** → Cliquer ▶️ à côté du Workflow Template
2. **Launch**
3. Observer la progression dans le Workflow Visualizer en temps réel

---

## 9. Surveillance et Logs

### Consulter les logs d'un job

1. **Jobs** → Cliquer sur un job en cours ou terminé
2. Onglet **Output** : logs en temps réel
3. Onglet **Details** : métadonnées du job

### Filtrer par hôte

Dans le log d'un job, utiliser le sélecteur d'hôtes pour filtrer les sorties par serveur.

### Historique

**Jobs** → Filtrer par Template, Status, Date pour voir l'historique d'exécution.
