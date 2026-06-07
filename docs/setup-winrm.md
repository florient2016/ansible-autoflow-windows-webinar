# Configuration WinRM — Windows Server 🔧

## Vue d'ensemble

WinRM (Windows Remote Management) est le protocole utilisé par Ansible pour communiquer avec les serveurs Windows. Il est basé sur le standard SOAP/HTTP et supporte plusieurs méthodes d'authentification.

---

## Configuration Rapide (Lab / Développement)

> ⚠️ **Usage lab uniquement** — HTTP non chiffré, authentification Basic

Exécuter en **PowerShell Administrateur** sur le serveur Windows :

```powershell
# ============================================================
# Script de configuration WinRM rapide pour environnement lab
# ============================================================

# 1. Activer WinRM et créer le listener HTTP
winrm quickconfig -Force

# 2. Autoriser toutes les origines (lab uniquement)
winrm set winrm/config/client '@{TrustedHosts="*"}'

# 3. Augmenter le timeout
winrm set winrm/config '@{MaxTimeoutms="1800000"}'

# 4. Activer l'authentification Basic (lab — non recommandé en prod)
winrm set winrm/config/service/auth '@{Basic="true"}'

# 5. Autoriser le trafic non chiffré (HTTP — lab uniquement)
winrm set winrm/config/service '@{AllowUnencrypted="true"}'

# 6. Règles pare-feu
netsh advfirewall firewall add rule `
  name="WinRM HTTP Ansible" `
  protocol=TCP `
  dir=in `
  localport=5985 `
  action=allow

# 7. Vérifier la configuration
winrm enumerate winrm/config/listener
winrm get winrm/config/service
```

**Test de connexion depuis Ansible** :
```bash
ansible windows-server-01 -i inventories/dev/hosts.yml -m win_ping
```

---

## Configuration Sécurisée (Production)

### Option 1 : HTTPS avec certificat auto-signé

```powershell
# ============================================================
# Configuration WinRM HTTPS avec certificat auto-signé
# ============================================================

# 1. Créer un certificat auto-signé
$cert = New-SelfSignedCertificate `
  -DnsName $env:COMPUTERNAME `
  -CertStoreLocation "Cert:\LocalMachine\My" `
  -KeyExportPolicy Exportable `
  -KeySpec Signature `
  -Subject "CN=$env:COMPUTERNAME"

# 2. Créer le listener HTTPS
$thumbprint = $cert.Thumbprint
winrm create winrm/config/listener?Address=*+Transport=HTTPS `
  "@{Hostname=`"$env:COMPUTERNAME`";CertificateThumbprint=`"$thumbprint`"}"

# 3. Activer l'authentification NTLM ou Negotiate
winrm set winrm/config/service/auth '@{Negotiate="true"}'
winrm set winrm/config/service/auth '@{Basic="false"}'  # Désactiver Basic

# 4. Règle pare-feu HTTPS
netsh advfirewall firewall add rule `
  name="WinRM HTTPS Ansible" `
  protocol=TCP `
  dir=in `
  localport=5986 `
  action=allow

# 5. Récupérer le thumbprint pour Ansible
Write-Host "Thumbprint certificat : $thumbprint"
```

**Configuration Ansible pour HTTPS** :
```yaml
ansible_winrm_port: 5986
ansible_winrm_scheme: https
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: ignore   # Ou 'validate' avec un CA de confiance
```

### Option 2 : HTTPS avec certificat CA (Recommandé)

```powershell
# Demander un certificat depuis votre CA Active Directory
# Puis configurer le listener HTTPS avec le certificat CA

# 1. Demander le certificat (adapter le template)
$cert = Get-Certificate `
  -Template "WebServer" `
  -SubjectName "CN=$env:COMPUTERNAME.$env:USERDNSDOMAIN" `
  -CertStoreLocation "Cert:\LocalMachine\My"

# 2. Configurer WinRM avec ce certificat
winrm create winrm/config/listener?Address=*+Transport=HTTPS `
  "@{Hostname=`"$env:COMPUTERNAME.$env:USERDNSDOMAIN`";CertificateThumbprint=`"$($cert.Certificate.Thumbprint)`"}"
```

### Option 3 : Kerberos (Active Directory — Recommandé en entreprise)

```yaml
# Inventaire Ansible pour Kerberos
ansible_winrm_transport: kerberos
ansible_winrm_port: 5985           # HTTP ou 5986 pour HTTPS
ansible_winrm_scheme: http
ansible_kerberos_delegation: true  # Si nécessaire
```

Prérequis :
- Hôte Ansible joint au domaine AD ou krb5 configuré
- Package Python `pywinrm[kerberos]` installé sur le nœud de contrôle
- SPN enregistré pour les serveurs cibles

---

## Vérification de la Configuration

### Sur le serveur Windows

```powershell
# État du service WinRM
Get-Service WinRM | Select-Object Name, Status, StartType

# Listeners actifs
winrm enumerate winrm/config/listener

# Configuration complète
winrm get winrm/config

# Test local
Test-WSMan -ComputerName localhost
```

### Depuis le nœud de contrôle Ansible

```bash
# Test de connectivité réseau
nc -zv <IP_WINDOWS> 5985

# Test WinRM avec Python
python3 -c "
import winrm
session = winrm.Session('<IP>', auth=('Administrator', 'Password'))
result = session.run_cmd('hostname')
print(result.std_out.decode())
"

# Test Ansible
ansible <hostname> -i inventories/dev/hosts.yml -m win_ping -vvv
```

---

## Méthodes d'Authentification — Comparaison

| Méthode | Sécurité | Environnement | Prérequis |
|---|---|---|---|
| Basic | ⚠️ Faible | Lab uniquement | Aucun |
| NTLM | ✅ Bon | Lab / Dev | Aucun (natif Windows) |
| Kerberos | ✅✅ Excellent | Production AD | Domain join + krb5 |
| Certificate | ✅✅ Excellent | Production | PKI / CA |
| CredSSP | ⚠️ Moyen | Double-hop | Activation manuelle |

---

## Variables Ansible par Méthode

```yaml
# Basic (lab uniquement — HTTP non chiffré)
ansible_winrm_transport: basic
ansible_winrm_port: 5985
ansible_winrm_scheme: http

# NTLM (lab / dev)
ansible_winrm_transport: ntlm
ansible_winrm_port: 5985

# NTLM sur HTTPS
ansible_winrm_transport: ntlm
ansible_winrm_port: 5986
ansible_winrm_scheme: https
ansible_winrm_server_cert_validation: ignore  # ou validate

# Kerberos (production)
ansible_winrm_transport: kerberos
ansible_winrm_port: 5985

# CredSSP (double-hop)
ansible_winrm_transport: credssp
ansible_winrm_port: 5985
```

---

## Ports Firewall Requis

| Port | Protocole | Usage |
|---|---|---|
| 5985 | TCP | WinRM HTTP |
| 5986 | TCP | WinRM HTTPS |
| 3389 | TCP | RDP (administration) |
| 445 | TCP | SMB (optionnel) |

---

## Script PowerShell Complet — Vérification WinRM

```powershell
# ============================================================
# Vérification complète de la configuration WinRM
# Copier et exécuter sur le serveur Windows cible
# ============================================================

Write-Host "=== Vérification WinRM ===" -ForegroundColor Cyan

# Service
$svc = Get-Service WinRM
Write-Host "Service WinRM   : $($svc.Status)" -ForegroundColor $(if ($svc.Status -eq 'Running') {'Green'} else {'Red'})

# Listeners
$listeners = winrm enumerate winrm/config/listener 2>&1
Write-Host "Listeners       :"
$listeners | ForEach-Object { Write-Host "  $_" }

# Auth
$auth = winrm get winrm/config/service/auth 2>&1
Write-Host "Auth config     :"
$auth | ForEach-Object { Write-Host "  $_" }

# Pare-feu
$fw5985 = Get-NetFirewallRule | Where-Object {
    $_ | Get-NetFirewallPortFilter | Where-Object LocalPort -eq 5985
}
Write-Host "Firewall 5985   : $(if ($fw5985) {'Règle trouvée'} else {'⚠️ Aucune règle'})" -ForegroundColor $(if ($fw5985) {'Green'} else {'Yellow'})

Write-Host "==========================" -ForegroundColor Cyan
```
