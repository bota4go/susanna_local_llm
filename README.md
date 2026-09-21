# SSH: raspberry (Linux) -> SUSANNA (Windows)

Verified working 2026-09-20. Raw debug history kept in `remote_ssh.md`.

## Topology

- Client: raspberry, Linux user `bota4go`, key `~/.ssh/id_ed25519`
  - pubkey: `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBDUJBJKDzoHGpfHTFHw0exYSuY30fc/VaucPqWrVJ25 bota4go@gmail.com`
- Server: SUSANNA, `192.168.1.112` (WiFi), Windows users `bota4go` and `sorok`, both in Administrators
- Verified: `ssh -i ~/.ssh/id_ed25519 -o PasswordAuthentication=no bota4go@192.168.1.112 "whoami"` -> `susanna\bota4go`

## Windows base setup (Admin PowerShell, once)

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
Get-Service sshd | Select Status, Name
Get-NetFirewallRule -Name sshd | Select Name,Enabled,Direction,Action
```

Localhost working but `192.168.1.112` failing = missing firewall rule above. No rule named `sshd` (`Get-NetFirewallRule: No objects found`) was the original bug.

## Key setup (the important part)

Windows OpenSSH ignores `C:\Users\<admin>\.ssh\authorized_keys` for admin users.
Authoritative file is `C:\ProgramData\ssh\administrators_authorized_keys`, which may not exist — create it.

Run in PowerShell on SUSANNA (as `bota4go`, then `powershell` if you arrived via ssh/cmd):

```powershell
$PubKey = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBDUJBJKDzoHGpfHTFHw0exYSuY30fc/VaucPqWrVJ25 bota4go@gmail.com"
$AdminKeyFile = "C:\ProgramData\ssh\administrators_authorized_keys"
$UserKeyFile = "C:\Users\bota4go\.ssh\authorized_keys"
if (!(Test-Path "C:\Users\bota4go\.ssh")) { New-Item -Type Directory -Force "C:\Users\bota4go\.ssh" }
if (!(Test-Path $UserKeyFile)) { New-Item -Type File -Force $UserKeyFile }
if (!(Test-Path $AdminKeyFile)) { New-Item -Type File -Force $AdminKeyFile }
if (!(Select-String -Path $UserKeyFile -Pattern "VaucPqWrVJ25" -Quiet)) { Add-Content -Path $UserKeyFile -Value $PubKey }
if (!(Select-String -Path $AdminKeyFile -Pattern "VaucPqWrVJ25" -Quiet)) { Add-Content -Path $AdminKeyFile -Value $PubKey }
icacls "C:\Users\bota4go\.ssh" /inheritance:r /grant "bota4go:(F)" /grant "SYSTEM:(F)" /grant "Administrators:(F)"
icacls $UserKeyFile /inheritance:r /grant "bota4go:(R)" /grant "SYSTEM:(F)" /grant "Administrators:(F)"
icacls $AdminKeyFile /inheritance:r /grant "SYSTEM:(F)" /grant "Administrators:(F)"
Get-Content $AdminKeyFile
icacls $AdminKeyFile
```

Gotchas hit here, don't regress:

- `"$User:(F)"` inside double quotes breaks PowerShell (`InvalidVariableReferenceWithDrive`). Use literal `"bota4go:(F)"` or `${User}:(F)`.
- `Select-String` on a nonexistent file errors; guard with `Test-Path` first.
- Over ssh you land in `cmd` (`'$User' is not recognized...`). Type `powershell` first, then paste.
- Logs cmdlet is `Get-WinEvent`, not `Get-WinEventLog`.

## Linux usage

```bash
# first login (password)
ssh bota4go@192.168.1.112
# key-only (no password prompt when working)
ssh -i ~/.ssh/id_ed25519 -o PasswordAuthentication=no bota4go@192.168.1.112 "whoami"
```

## Troubleshooting

```powershell
Get-Service sshd | Select Status, Name
netstat -ano | findstr :22          # expect 0.0.0.0:22 LISTENING
Get-NetFirewallRule -Name sshd | Select Name,Enabled,Direction,Action
Get-LocalGroupMember Administrators | Select Name   # admin? -> ProgramData file wins
Get-WinEvent -LogName OpenSSH/Operational -MaxEvents 20 | Select TimeCreated,Message
```

```bash
ssh -vvv bota4go@192.168.1.112 2>&1 | grep -i -A2 "offer|publickey|denied|auth"
```
