# Guide de Dépannage 🔧

Résolution des problèmes courants lors de l'automatisation Windows avec Ansible Automation Platform.

---

## 1. WinRM Inaccessible

### Symptômes

```
UNREACHABLE! => {"changed": false, "msg": "winrm.exceptions.InvalidSchemeError ...", ...}
fatal: [windows-server-01]: UNREACHABLE! => {
    "msg": "HTTPConnectionPool(host='192.168.1.100', port=5985): Max retries exceeded"
}
```

### Causes et Solutions

**Cause 1 : Service WinRM non démarré**
```powershell
# Sur le serveur Windows — vérifier l'état du service
Get-Service WinRM

# Démarrer WinRM
Start-Service WinRM
Set-Service WinRM -StartupType Automatic

# Reconfigurer WinRM
winrm quickconfig -Force
```

**Cause 2 : Pare-feu bloque le port 5985**
```powershell
# Vérifier les règles firewall
Get-NetFirewallRule | Where-Object { $_.DisplayName -like "*WinRM*" -or $_.DisplayName -like "*Remote*" }

# Ajouter la règle manuellement
New-NetFirewallRule -Name "WinRM-HTTP-Ansible" `
  -DisplayName "WinRM HTTP pour Ansible" `
  -Protocol TCP `
  -LocalPort 5985 `
  -Direction Inbound `
  -Action Allow

# Ou avec netsh
netsh advfirewall firewall add rule name="WinRM HTTP" `
  protocol=TCP dir=in localport=5985 action=allow
```

**Cause 3 : Mauvaise IP dans l'inventaire**
```bash
# Tester la connectivité réseau
ping 192.168.1.100
nc -zv 192.168.1.100 5985    # Linux/macOS
Test-NetConnection -ComputerName 192.168.1.100 -Port 5985    # PowerShell
```

**Cause 4 : Listener WinRM non configuré**
```powershell
# Lister les listeners
winrm enumerate winrm/config/listener

# Si aucun listener HTTP, en créer un
winrm create winrm/config/listener?Address=*+Transport=HTTP
```

**Cause 5 : Variables de connexion incorrectes**
```yaml
# Vérifier dans l'inventaire AAP ou group_vars/windows.yml
ansible_connection: winrm         # Obligatoire
ansible_winrm_port: 5985          # 5985=HTTP, 5986=HTTPS
ansible_winrm_scheme: http        # http ou https
ansible_winrm_transport: ntlm     # basic, ntlm, kerberos
```

---

## 2. Authentification Refusée

### Symptômes

```
fatal: [windows-server-01]: UNREACHABLE! => {
    "msg": "ntlm: NTLM authentication failure",
    "unreachable": true
}

winrm.exceptions.InvalidCredentialsError: the specified credentials were rejected by the server
```

### Causes et Solutions

**Cause 1 : Mauvais login/mot de passe**
```bash
# Tester l'authentification directement
ansible windows-server-01 -i inventories/dev/hosts.yml -m win_ping \
  -e "ansible_user=Administrator" \
  -e "ansible_password=MotDePasseCorrect" \
  -vvv
```

**Cause 2 : Transport incompatible**
```yaml
# Pour un serveur hors domaine, utiliser Basic ou NTLM (pas Kerberos)
ansible_winrm_transport: ntlm

# Pour Basic, l'activer côté Windows d'abord
# winrm set winrm/config/service/auth '@{Basic="true"}'
ansible_winrm_transport: basic
```

**Cause 3 : Compte verrouillé ou désactivé**
```powershell
# Vérifier l'état du compte
Get-LocalUser -Name Administrator

# Activer le compte Administrator
Enable-LocalUser -Name Administrator

# Déverrouiller le compte
$account = [ADSI]"WinNT://$env:COMPUTERNAME/Administrator"
$account.IsAccountLocked = $false
$account.SetInfo()
```

**Cause 4 : UAC bloque l'accès réseau pour les comptes locaux**
```powershell
# Désactiver la restriction UAC pour les comptes locaux (lab uniquement)
# Ne pas faire en production !
New-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -Name LocalAccountTokenFilterPolicy `
  -Value 1 `
  -PropertyType DWORD `
  -Force
```

**Cause 5 : NTLM désactivé sur le serveur**
```powershell
# Vérifier la politique NTLM
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "LmCompatibilityLevel"

# Réactiver NTLM (si niveau trop restrictif)
# 0 = envoyer LM & NTLM, 3 = NTLMv2 seulement (recommandé)
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "LmCompatibilityLevel" -Value 3
```

---

## 3. Module IIS Non Reconnu / Erreur win_feature

### Symptômes

```
fatal: [windows-server-01]: FAILED! => {
    "msg": "One or more roles, role services, or features specified in the
            command do not exist on this computer"
}

The term 'Install-WindowsFeature' is not recognized
```

### Causes et Solutions

**Cause 1 : Windows Server Feature non disponible (édition minimale)**
```powershell
# Vérifier les fonctionnalités disponibles
Get-WindowsFeature | Where-Object { $_.Name -like "*Web*" } | Format-Table Name, DisplayName, Installed

# Si Get-WindowsFeature n'existe pas (Windows 10/11), utiliser :
Get-WindowsOptionalFeature -Online | Where-Object { $_.FeatureName -like "*IIS*" }
```

**Cause 2 : Windows 10/11 au lieu de Windows Server**

Le module `win_feature` requiert **Windows Server**. Pour Windows 10/11 :
```yaml
# Utiliser win_optional_feature pour les clients Windows
- name: Installer IIS sur Windows client
  ansible.windows.win_optional_feature:
    name: IIS-WebServerRole
    state: present
```

**Cause 3 : Droits insuffisants pour installer des fonctionnalités**
```bash
# Vérifier que le compte a les droits d'installation
# Le compte doit être membre du groupe Administrators local
```

**Cause 4 : Collection ansible.windows manquante dans l'EE**
```bash
# Dans l'EE, vérifier les collections disponibles
ansible-galaxy collection list

# Installer si manquant
ansible-galaxy collection install ansible.windows
```

---

## 4. Erreur Chocolatey

### Symptômes

```
fatal: [windows-server-01]: FAILED! => {
    "msg": "Failed to install chocolatey from https://chocolatey.org/install.ps1"
}

PackageManagement\Get-Package : No match was found for the specified search criteria
```

### Causes et Solutions

**Cause 1 : Pas d'accès Internet depuis le serveur Windows**
```powershell
# Tester l'accès Internet depuis le serveur
Test-NetConnection -ComputerName chocolatey.org -Port 443

# Tester le téléchargement du script d'installation
Invoke-WebRequest -Uri https://chocolatey.org/install.ps1 -UseBasicParsing
```

**Cause 2 : Proxy bloque l'accès**
```powershell
# Configurer le proxy PowerShell
$proxy = "http://proxy.votre-entreprise.com:8080"
[System.Net.WebRequest]::DefaultWebProxy = New-Object System.Net.WebNetProxy($proxy)
[System.Net.WebRequest]::DefaultWebProxy.Credentials = [System.Net.CredentialCache]::DefaultNetworkCredentials
```

**Cause 3 : Politique d'exécution PowerShell restrictive**
```powershell
# Vérifier la politique
Get-ExecutionPolicy -List

# Assouplir pour Chocolatey (lab uniquement)
Set-ExecutionPolicy Bypass -Scope Process -Force
Set-ExecutionPolicy RemoteSigned -Scope LocalMachine
```

**Cause 4 : Package Chocolatey introuvable (nom incorrect)**
```powershell
# Rechercher le bon nom de package
choco search notepadplusplus
choco search git
# Ou sur https://community.chocolatey.org/packages
```

**Cause 5 : TLS 1.2 non activé (Windows Server 2016)**
```powershell
# Activer TLS 1.2 si nécessaire
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

**Cause 6 : Timeout Chocolatey trop court**
```yaml
# Augmenter le timeout dans ansible.cfg ou dans la task
- name: Installer package via Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: git
    state: present
  async: 600   # 10 minutes max
  poll: 10
```

---

## 5. Problème de Droits Administrateur

### Symptômes

```
fatal: [windows-server-01]: FAILED! => {
    "msg": "Access to the path 'C:\\Windows\\System32\\...' is denied."
}

Access is denied
The requested operation requires elevation.
```

### Causes et Solutions

**Cause 1 : Compte non administrateur**
```powershell
# Vérifier les groupes du compte utilisé par Ansible
net localgroup Administrators

# Ajouter le compte au groupe Administrators
net localgroup Administrators moncompte /add
```

**Cause 2 : UAC empêche l'élévation via WinRM**
```powershell
# Solution permanente (lab) : désactiver UAC pour les comptes locaux
Set-ItemProperty `
  -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -Name "LocalAccountTokenFilterPolicy" `
  -Value 1

# Ou créer un compte de domaine Administrateur (recommandé en production)
```

**Cause 3 : Utiliser `become` avec WinRM**
```yaml
# Dans le playbook ou la tâche, forcer l'élévation
- name: Tâche nécessitant des droits élevés
  ansible.windows.win_shell: |
    # commande admin
  become: true
  become_method: runas
  become_user: SYSTEM
```

**Cause 4 : Permissions sur un fichier ou dossier spécifique**
```powershell
# Vérifier les ACL
Get-Acl "C:\Automation" | Format-List

# Accorder les droits
$acl = Get-Acl "C:\Automation"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "automation_admin", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow"
)
$acl.SetAccessRule($rule)
Set-Acl "C:\Automation" $acl
```

---

## 6. Erreurs Communes — Référence Rapide

| Message d'erreur | Cause probable | Solution rapide |
|---|---|---|
| `Max retries exceeded` | Port 5985 bloqué | Vérifier pare-feu + service WinRM |
| `NTLM authentication failure` | Mauvais credentials | Vérifier login/password dans AAP |
| `InvalidSchemeError` | Mauvais scheme | `ansible_winrm_scheme: http` |
| `certificate verify failed` | Cert SSL invalide | `ansible_winrm_server_cert_validation: ignore` |
| `One or more roles do not exist` | Pas Windows Server | `win_feature` requiert Windows Server |
| `Access is denied` | Droits insuffisants | Compte doit être Administrateur local |
| `Chocolatey install failed` | Pas d'accès Internet | Vérifier proxy/DNS/connectivité |
| `Module not found` | Collection manquante | Vérifier l'Execution Environment |
| `Connection timeout` | Réseau lent / chargé | Augmenter `ansible_winrm_operation_timeout_sec` |

---

## 7. Mode Debug Complet

```bash
# Activer le debug maximum (verbosité 5)
ansible-playbook -i inventories/dev/hosts.yml \
  projects/windows-ping-test/playbook.yml \
  -vvvvv 2>&1 | tee /tmp/ansible-debug.log

# Variables d'environnement de debug WinRM
export PYWINRM_LOG_LEVEL=DEBUG
export REQUESTS_CA_BUNDLE=""

# Dans AAP : Verbosity → 5 - WinRM Debug
```

---

## 8. Ressources Utiles

- [Documentation WinRM Ansible](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html)
- [Collection ansible.windows](https://docs.ansible.com/ansible/latest/collections/ansible/windows/)
- [Collection community.windows](https://docs.ansible.com/ansible/latest/collections/community/windows/)
- [Chocolatey Packages](https://community.chocolatey.org/packages)
- [Troubleshooting AAP](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/)
