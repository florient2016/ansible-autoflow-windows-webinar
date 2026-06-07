# Scénario du Webinar — Étape par Étape 🎬

## Vue d'ensemble

**Durée estimée** : 60–90 minutes  
**Public** : Administrateurs système, DevOps, architectes infrastructure  
**Niveau** : Intermédiaire (connaissance de base d'Ansible)

---

## Introduction (10 min)

### 0.1 Présentation du webinar

> "Aujourd'hui, nous allons voir comment **Ansible Automation Platform** permet de gérer des serveurs Windows de manière complète, reproductible et sécurisée. Nous allons démontrer 3 cas d'usage progressifs : du simple test de connectivité jusqu'à la configuration baseline d'un serveur."

**Points clés à aborder** :
- Pourquoi Ansible pour Windows ?
  - Agentless (pas d'agent à installer)
  - WinRM natif (protocole Microsoft)
  - Idempotent (rejouable sans risque)
  - Plateforme unifiée Linux + Windows
- Architecture : AAP → WinRM → Windows Server
- Différence entre playbooks locaux et AAP en tant que moteur d'orchestration

### 0.2 Présentation de l'architecture du webinar

```
┌─────────────────────────────────────────────────────┐
│         Ansible Automation Platform (AAP)           │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Project  │  │Inventory │  │   Credentials    │  │
│  │(GitHub)  │  │(Windows) │  │ (WinRM/Password) │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                     │
│  ┌────────────────────────────────────────────────┐ │
│  │              Execution Engine                  │ │
│  │  [Job 1: Ping] → [Job 2: IIS] → [Job 3: Base] │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
                         │ WinRM (5985/5986)
                         ▼
          ┌──────────────────────────┐
          │    Windows Server 2022   │
          │  192.168.1.100           │
          └──────────────────────────┘
```

### 0.3 Présentation du repository GitHub

- Ouvrir le repository dans un navigateur
- Expliquer la structure des dossiers
- Montrer les 3 projets et leur logique progressive

---

## Partie 1 : Import du Repository dans AAP (10 min)

### 1.1 Accéder à l'interface AAP

```
URL : https://votre-aap.votre-domaine.com
```

### 1.2 Créer le Project

> "La première étape est d'importer notre repository GitHub dans AAP. AAP va synchroniser le code et le rendre disponible pour les Job Templates."

**Actions dans l'UI** :
1. Menu **Projects** → **Add**
2. Remplir le formulaire :
   - Name : `AAP Windows Webinar`
   - SCM Type : Git
   - SCM URL : `https://github.com/<votre-org>/ansible-aap-windows-webinar`
   - Branch : `main`
   - Options : ✅ Update Revision on Launch
3. **Save** puis **Sync**
4. Montrer le statut **Successful** et le hash du commit

**Point pédagogique** :
> "Avec 'Update Revision on Launch', chaque exécution de job synchronise d'abord le repository. Cela garantit qu'on joue toujours le code le plus récent."

---

## Partie 2 : Configuration de l'Inventaire (5 min)

### 2.1 Créer l'inventaire

1. **Inventories** → **Add inventory**
2. Name : `Windows Dev Inventory`
3. **Save**

### 2.2 Ajouter le groupe Windows

1. Onglet **Groups** → **Add**
2. Name : `windows`
3. Variables du groupe :
```yaml
ansible_connection: winrm
ansible_winrm_transport: ntlm
ansible_winrm_port: 5985
ansible_winrm_scheme: http
ansible_winrm_server_cert_validation: ignore
ansible_shell_type: powershell
```

### 2.3 Ajouter l'hôte Windows

1. Dans le groupe `windows` → **Hosts** → **Add**
2. Name : `windows-server-01`
3. Variables : `ansible_host: 192.168.1.100`

**Point pédagogique** :
> "On centralise toutes les variables de connexion WinRM dans le groupe. Chaque hôte hérite de ces variables automatiquement."

---

## Partie 3 : Configuration des Credentials (5 min)

### 3.1 Créer le credential Windows

1. **Credentials** → **Add**
2. Credential Type : **Machine**
3. Name : `Windows Machine Credential`
4. Username : `Administrator`
5. Password : *(saisir le mot de passe)*
6. **Save**

**Point pédagogique — Sécurité** :
> "Le mot de passe est chiffré dans la base de données AAP. Il n'apparaît jamais en clair dans les logs ni dans le code. C'est le principe fondamental : le code dans GitHub ne contient aucun secret."

---

## Partie 4 : Démonstration du Projet 1 — Ping Test (10 min)

### 4.1 Créer le Job Template

1. **Templates** → **Add job template**
2. Remplir selon [aap-configuration.md](aap-configuration.md)
3. **Save**

### 4.2 Exécuter le Ping Test

1. Cliquer **Launch** ▶️
2. Observer la progression en temps réel
3. Commenter les logs :

```
TASK [✅ Test de connectivité WinRM (win_ping)]
ok: [windows-server-01] ← "ok" = connexion établie, pas de changement

TASK [📊 Afficher les informations système collectées]
ok: [windows-server-01] => {
    "msg": [
        "Hostname : WIN-SERVER-01",
        "OS       : Windows 2022 10.0.20348.0",
        ...
    ]
}
```

**Points à souligner** :
- Le `ok` en vert = idempotent, rien n'a changé sur la cible
- La collecte automatique de facts Windows
- Le temps d'exécution affiché à la fin

---

## Partie 5 : Démonstration du Projet 2 — IIS Management (20 min)

### 5.1 Créer le Job Template IIS

1. **Templates** → **Add job template**
2. Configuration selon [aap-configuration.md](aap-configuration.md)
3. **Save**

### 5.2 Expliquer le playbook avant l'exécution

Montrer dans GitHub :
- `projects/windows-iis-management/playbook.yml` → appel du rôle
- `roles/iis_management/tasks/main.yml` → les étapes
- `roles/iis_management/templates/index.html.j2` → le template Jinja2
- `roles/iis_management/defaults/main.yml` → les variables configurables

### 5.3 Lancer avec les variables par défaut

1. **Launch** ▶️
2. Observer les `changed` en jaune (installation IIS, déploiement page web)

**Commenter les étapes clés** :
```
TASK [🗑️ [IIS] Désinstaller IIS et ses composants]
changed: [windows-server-01] ← IIS était présent, il a été désinstallé

TASK [📦 [IIS] Installer IIS et les composants Web]
changed: [windows-server-01] ← Réinstallation

TASK [🌐 [IIS] Déployer la page web par défaut]
changed: [windows-server-01] ← Page HTML générée depuis le template Jinja2

TASK [🔍 [IIS] Vérifier que le site web répond]
ok: [windows-server-01] ← HTTP 200 OK !
```

### 5.4 Démontrer l'idempotence

1. Changer `iis_force_reinstall` à `false` dans les Extra Variables
2. Relancer le job
3. Montrer que tout est `ok` (aucun `changed`) → IIS était déjà dans le bon état

### 5.5 Personnaliser la page web (démo interactive)

1. Cliquer **Launch** ▶️
2. Dans la fenêtre de prompt, modifier les Extra Variables :
```yaml
iis_page_title: "🚀 Webinar AAP Live Demo"
iis_page_message: "Page personnalisée en direct depuis AAP !"
iis_force_reinstall: false
```
3. Lancer et montrer la nouvelle page dans le navigateur

---

## Partie 6 : Démonstration du Projet 3 — System Baseline (15 min)

### 6.1 Créer le Job Template Baseline

1. **Templates** → **Add job template**
2. Configuration selon [aap-configuration.md](aap-configuration.md)
3. **Save**

### 6.2 Expliquer le rôle avant l'exécution

Montrer dans GitHub :
- Les 6 sections du rôle (user, dossier, Chocolatey, packages, firewall, rapport)
- Le template Jinja2 du rapport système
- La gestion sécurisée du mot de passe (`no_log: true`)

### 6.3 Lancer le Baseline

1. **Launch** ▶️
2. Observer l'exécution (plus longue car Chocolatey installe des packages)

**Commenter les étapes** :
```
TASK [👤 [Baseline] Créer l'utilisateur 'automation_admin']
changed: [windows-server-01] ← Utilisateur créé

TASK [🍫 [Baseline] Installer Chocolatey]
changed: [windows-server-01] ← Chocolatey installé

TASK [🛠️ [Baseline] Installer les outils de base]
changed: [windows-server-01] ← git, notepad++, 7zip installés

TASK [🔥 [Baseline] Activer les règles firewall]
changed: [windows-server-01] ← Règles WinRM, ICMP créées

TASK [📄 [Baseline] Générer le rapport système]
changed: [windows-server-01] ← Rapport généré dans C:\Automation\reports\
```

### 6.4 Vérifier le rapport généré

Montrer sur le serveur Windows :
```powershell
Get-Content "C:\Automation\reports\system-report.txt"
```

---

## Partie 7 : Démonstration du Workflow (10 min)

### 7.1 Créer le Workflow Template

1. **Templates** → **Add workflow template**
2. Name : `🪟 Windows Server — Full Deployment Workflow`
3. **Visualizer** pour configurer la chaîne des 3 jobs

### 7.2 Expliquer la logique du Workflow

```
[01 Ping] ──Success──► [02 IIS] ──Success──► [03 Baseline] ──Success──► ✅
    │                      │                       │
   Fail                   Fail                    Fail
    │                      │                       │
    └──────────────────────┴───────────────────────┘
                    [Handler d'erreur]
```

**Point pédagogique** :
> "Le Workflow garantit qu'on ne tente pas d'installer IIS si la connectivité WinRM est coupée. C'est de la gestion d'erreur déclarative."

### 7.3 Lancer le Workflow

1. **Launch** ▶️
2. Observer le Workflow Visualizer en temps réel
3. Voir les nœuds passer au vert l'un après l'autre

### 7.4 Simuler un échec (optionnel)

1. Éteindre temporairement le service WinRM sur le serveur Windows
2. Relancer le workflow
3. Montrer que le nœud 1 échoue et que les nœuds suivants ne s'exécutent pas

---

## Conclusion (5 min)

### 8.1 Récapitulatif

> "Nous avons vu comment Ansible Automation Platform permet de :"
> - "Gérer la connectivité Windows via WinRM sans agent"
> - "Déployer et configurer IIS de manière déclarative et idempotente"
> - "Configurer une base système complète (utilisateurs, outils, firewall)"
> - "Orchestrer plusieurs projets via un Workflow avec gestion des erreurs"

### 8.2 Points clés à retenir

1. **Agentless** : Ansible se connecte via WinRM — aucune installation sur les serveurs cibles
2. **Idempotence** : Les playbooks peuvent être rejoués — le résultat est toujours le même
3. **Sécurité** : Les credentials sont gérés par AAP — jamais dans le code
4. **Templates Jinja2** : Personnalisation dynamique des fichiers déployés
5. **Workflows** : Orchestration avec gestion des succès/échecs entre les jobs

### 8.3 Prochaines étapes suggérées

- Ajouter l'authentification Kerberos pour les environnements AD
- Configurer des notifications (email, Slack) en cas d'échec
- Planifier les jobs (scheduling) pour des vérifications régulières
- Intégrer avec un CMDB ou ServiceNow pour la gestion des changements
- Explorer les Dynamic Inventories pour des environnements cloud (AWS, Azure)

### 8.4 Resources

- [docs.ansible.com/windows](https://docs.ansible.com/ansible/latest/os_guide/windows_usage.html)
- [Red Hat AAP Documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/)
- Ce repository GitHub : `https://github.com/<votre-org>/ansible-aap-windows-webinar`

---

## Checklist Pré-Webinar

- [ ] Serveur Windows accessible et WinRM configuré
- [ ] AAP opérationnel et accessible
- [ ] Repository GitHub synchronisé dans AAP
- [ ] Inventaire créé avec l'hôte Windows
- [ ] Credentials Windows créés dans AAP
- [ ] Les 3 Job Templates créés et testés
- [ ] Workflow Template créé
- [ ] Test complet du workflow en privé avant le webinar
- [ ] Navigateur ouvert sur le serveur Windows cible (pour la démo IIS)
- [ ] PowerShell ouvert sur le serveur Windows (pour la démo rapport)
