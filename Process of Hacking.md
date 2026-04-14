# 🎯 Process of Hacking — Metodologia de Bug Bounty

> Metodologia completa para encontrar e reportar vulnerabilidades em programas de Bug Bounty.

---

## 💰 Plataformas de Bug Bounty

| Plataforma | Região | Link |
|---|---|---|
| **HackerOne** | Global (maior do mundo) | https://www.hackerone.com/ |
| **Bugcrowd** | Global | https://www.bugcrowd.com/ |
| **Intigriti** | Europa | https://www.intigriti.com/ |
| **BugHunt** | Brasil 🇧🇷 | https://bughunt.com.br/ |

> [!important] Links da Aula
> https://app.hackingclub.com/training/learning-paths/1/lesson/111
> https://app.hackingclub.com/training/learning-paths/3/lesson/130

---

## 🗺️ Metodologia — Visão Geral

```
┌─────────────┐    ┌──────────────────────┐    ┌───────────┐    ┌──────────┐
│   1. RECON  │ →  │ 2. ANÁLISE/EXPLORAÇÃO│ →  │  3. PoC   │ →  │ 4. REPORT│
│ Reconheci-  │    │ Identificar e testar │    │ Provar a  │    │ Enviar o │
│ mento total │    │ vulnerabilidades     │    │ falha     │    │ relatório│
└─────────────┘    └──────────────────────┘    └───────────┘    └──────────┘
```

---

## 1️⃣ Recon — Reconhecimento

> **A parte mais importante!** Quanto mais informações coletamos, mais vulnerabilidades encontramos.

### Scope e Out of Scope
Primeiro, leia as regras do programa:
- **Scope** — Domínios, IPs e endpoints que você **pode** testar
- **Out of Scope** — O que **não** pode testar (risco de ação legal!)

### Subdomain Discovery
Identificar todos os subdomínios do alvo. Quanto mais subdomínios → mais superfície de ataque.

```bash
# Descoberta passiva (sem tocar o alvo)
subfinder -d alvo.com -all -silent

# Histórico de subdomínios
# SecurityTrails: https://securitytrails.com/
# Wayback Machine: https://web.archive.org/

# Brute Force de subdomínios
# Sublist3r: https://github.com/aboul3la/Sublist3r
# SubBrute: https://github.com/TheRook/subbrute
```

Vai vir **muitos** subdomínios — por isso usamos automações para filtrar quais estão ativos e merecem teste:

```bash
# Filtrar subdomínios ativos e verificar tecnologias
subfinder -d alvo.com -all -silent | httpx -sc -td

# Ver URLs históricas para cada subdomínio
gau alvo.com

# Crawler ativo — busca URLs, parâmetros e endpoints JS
katana -u http://alvo.com -d 3 -jc
```

### Content Discovery
Identificar informações e dados ocultos que podem ser interessantes:
- Diretórios e arquivos escondidos (fuzzing)
- Arquivos de backup (`.bak`, `.old`, `.swp`)
- Painéis de admin
- Arquivos de configuração expostos (`.env`, `.git/`)

```bash
# Fuzzing com ffuf
ffuf -u http://alvo.com/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,403
```

### Application Discovery
Identificar se o alvo usa **software de terceiros** (WordPress, Jenkins, Tomcat, etc.). Se usar → verificar se a versão tem vulnerabilidades conhecidas (CVEs não patcheados).

**Objetivo:** Obter informações sobre as aplicações rodando nos subdomínios — URLs, parâmetros, endpoints de APIs.

**Ferramentas:**
| Ferramenta | Função |
|---|---|
| **httpx** | Verificar subdomínios ativos + tech detect |
| **gau** | URLs históricas do domínio |
| **katana** | Crawler ativo de URLs e JS |
| **waymore** | Versão avançada do gau (mais fontes) |
| **kxss** | Verificar se parâmetros refletem (potencial XSS) |
| **uro** | Filtrar/deduplicar URLs |
| **ffuf** | Fuzzing de diretórios e endpoints |
| **SecretFinder** | Extrair segredos de arquivos JS |

**Fluxo recomendado:**
```
subfinder → httpx (filtrar ativos) → gau/katana (coletar URLs) → kxss (testar reflexão) → ffuf (fuzzing)
```

---

## 2️⃣ Análise e Exploração

### Filtro e Validação
Identificar quais alvos estão rodando servidores HTTP, quais tecnologias usam, e priorizar os mais promissores.

### Vulnerability Assessment
Processo de identificar vulnerabilidades nos ativos encontrados. Em Bug Bounty, **quase 100% das vezes são testes manuais** — scanners automáticos encontram apenas as falhas mais óbvias.

**Referência:** Use o [WSTG da OWASP](https://owasp.org/www-project-web-security-testing-guide/) como checklist de testes.

**O que testar (por prioridade):**

| Vulnerabilidade | Onde testar | Impacto |
|---|---|---|
| **SQLi** | Parâmetros em URLs, formulários, APIs | Crítico |
| **XSS** | Campos de input, URLs, headers | Alto |
| **IDOR** | IDs em URLs, APIs, cookies | Alto |
| **SSRF** | Campos que aceitam URLs | Alto |
| **LFI/RFI** | Parâmetros que referenciam arquivos | Alto |
| **Command Injection** | Campos que interagem com o sistema | Crítico |
| **Auth Bypass** | Login, reset de senha, tokens | Crítico |
| **Open Redirect** | Parâmetros de redirect | Médio |
| **Info Disclosure** | Headers, erros, arquivos expostos | Baixo-Médio |

---

## 3️⃣ PoC — Proof of Concept

A PoC é a **prova** de que a vulnerabilidade existe e funciona. Deve ser **clara, reproduzível e minimamente invasiva**.

**Uma boa PoC contém:**
- URL exata onde a vulnerabilidade foi encontrada
- Passos para reproduzir (step-by-step)
- Payload utilizado
- Screenshot ou vídeo da exploração
- Impacto real (o que um atacante poderia fazer)

> **Dica:** Nunca acesse dados reais de outros usuários para "provar" a falha. Use apenas seus próprios dados de teste.

---

## 4️⃣ Report — Relatório Final

Última parte — tem tudo pronto, é só reportar de forma organizada.

**Estrutura de um bom relatório:**

```markdown
## Título
[Tipo da Vuln] - Descrição curta em uma linha

## Severity
Critical / High / Medium / Low

## Descrição
O que é a vulnerabilidade e por que ela é um problema.

## Passos para Reproduzir
1. Acesse http://...
2. Intercepte a requisição com Burp Suite
3. Modifique o parâmetro X para Y
4. Observe o resultado

## Impacto
O que um atacante pode fazer com essa vulnerabilidade.

## Payload Utilizado
[Payload exato]

## Evidência
[Screenshots / Vídeo]

## Recomendação de Correção
Como a empresa pode resolver o problema.
```

> **Dica de ouro:** Um relatório bem escrito pode valer **mais bounty** que um relatório confuso sobre a mesma vulnerabilidade. Empresas valorizam reports claros!
