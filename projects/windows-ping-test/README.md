# Projet 1 — Windows Ping Test 🔌

## Description

Ce projet valide la **connectivité WinRM** entre Ansible Automation Platform et un serveur Windows. C'est la première étape indispensable avant toute automatisation Windows.

## Ce que fait ce playbook

1. **Test `win_ping`** — Vérifie que la connexion WinRM est opérationnelle
2. **Collecte des facts** — Informations système (OS, RAM, CPU, IP...)
3. **Exécution PowerShell** — Valide l'exécution de commandes via `win_command`
4. **Informations réseau** — Liste des interfaces réseau via `win_shell`
5. **Statut WinRM** — Confirme que le service WinRM tourne correctement
6. **Résumé** — Affichage synthétique du résultat

## Prérequis

- WinRM activé sur le serveur Windows (voir [setup-winrm.md](../../docs/setup-winrm.md))
- Credentials Windows configurés dans AAP
- Collection `ansible.windows` installée dans l'Execution Environment

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `ping_retries` | `3` | Nombre de tentatives avant échec |
| `ping_delay` | `5` | Délai entre les tentatives (s) |

## Exécution locale (test)

```bash
# Avec fichier d'inventaire
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-ping-test/playbook.yml \
  -e "ansible_user=Administrator" \
  -e "ansible_password=VotreMotDePasse"

# Mode verbose pour le debug
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-ping-test/playbook.yml -vvv
```

## Configuration dans AAP

### Job Template

| Paramètre | Valeur |
|---|---|
| **Name** | `01 - Windows Ping Test` |
| **Job Type** | Run |
| **Inventory** | `Windows Dev Inventory` |
| **Project** | `AAP Windows Webinar` |
| **Playbook** | `projects/windows-ping-test/playbook.yml` |
| **Credentials** | `Windows Machine Credential` |
| **Verbosity** | `1 (Verbose)` pour le webinar |

## Résultat attendu

```
TASK [✅ Test de connectivité WinRM (win_ping)] ***
ok: [windows-server-01]

TASK [📊 Afficher les informations système collectées] ***
ok: [windows-server-01] => {
    "msg": [
        "Hostname         : WIN-SERVER-01",
        "OS               : Windows 2022 10.0.20348.0",
        ...
    ]
}

TASK [🎉 Résumé — Test de connectivité réussi !] ***
ok: [windows-server-01] => {
    "msg": [
        "======================================================",
        " ✅ Connexion WinRM établie avec succès !",
        ...
    ]
}
```

## Dépannage

Si le ping échoue, consultez [troubleshooting.md](../../docs/troubleshooting.md).

Les erreurs les plus courantes :
- `winrm.exceptions.InvalidCredentialsError` → Vérifier le login/mot de passe
- `socket.error: [Errno 111] Connection refused` → WinRM non démarré ou port bloqué
- `ntlm: NTLM authentication failure` → Vérifier le transport WinRM
