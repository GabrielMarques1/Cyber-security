<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=28&pause=1000&color=00FF41&center=true&vCenter=true&width=600&lines=Cyber+Security+Notes;Offensive+Security+%F0%9F%94%A5;Bug+Bounty+%7C+Pentest+%7C+CTF" alt="Typing SVG" />

<br/>

<img src="https://img.shields.io/badge/Kali%20Linux-557C94?style=for-the-badge&logo=kalilinux&logoColor=white"/>
<img src="https://img.shields.io/badge/Arch%20Linux-1793D1?style=for-the-badge&logo=archlinux&logoColor=white"/>
<img src="https://img.shields.io/badge/Bug%20Bounty-FF6B6B?style=for-the-badge&logo=hackerone&logoColor=white"/>
<img src="https://img.shields.io/badge/Pentest-00FF41?style=for-the-badge&logo=metasploit&logoColor=black"/>
<img src="https://img.shields.io/badge/CTF-FFD700?style=for-the-badge&logo=hackthebox&logoColor=black"/>
<img src="https://img.shields.io/badge/OWASP-API%20%26%20Web-blue?style=for-the-badge&logo=owasp&logoColor=white"/>

<br/><br/>

> **Base de Conhecimento, Metodologias e Arsenal de Segurança Ofensiva Ética.**  
> Estudos contínuos de Web Hacking, API Security, Linux PrivEsc, Evasão de WAF e Metodologias PTES/MITRE ATT&CK.  
> Sincronizado com o portfólio oficial em: **[gabrielsec.live](https://www.gabrielsec.live)**

</div>

---

## 📁 Estrutura do Vault

```
📂 Cyber-security/
├── 📄 Cybersec -.md                                 → Índice central do vault (Home no Obsidian)
├── 📄 Process of Hacking.md                         → Metodologia estruturada de Pentest e Bug Bounty
├── 📄 Essential Web Hacking.md                      → Arquitetura e teoria profunda de vulnerabilidades web
├── 📄 OWASP API Top 10.md                           → Análise analítica e exploração da OWASP API Security Top 10
├── 📄 Payloads - Web Hacking.md                     → Dicionário tático de payloads práticos e variações
├── 📄 WAF Bypass.md                                 → Técnicas avançadas de evasão de Web Application Firewalls
├── 📄 Linux Privilege Escalation.md                 → Guia exaustivo de escalação de privilégios em GNU/Linux
├── 📄 Kali linux & Arch e minhas anotações basicas.md→ Comandos de terminal, dotfiles, redes e pivoting
├── 📄 REDES -.md                                    → Fundamentos de redes (TCP/IP, OSI, DNS, HTTP/S, ARP)
├── 📄 plano de Estudos.md                           → Trilha de capacitação técnica (HTB, Hacking Club, PortSwigger)
└── 🗺️ REDES - GERAL.canvas                         → Diagrama conceitual visual de topologia de redes
```

---

## 📚 Módulos de Conhecimento

### 🗂️ [Cybersec - (Índice Central)](./Cybersec%20-.md)
Ponto de partida do vault no Obsidian. Centraliza o mapeamento relacional entre notas através de links bidirecionais (`[[...]]`), servindo como MOC (*Map of Content*).

---

### 🎯 [Process of Hacking](./Process%20of%20Hacking.md)
Framework operacional e sistemático alinhado às fases do **PTES** (*Penetration Testing Execution Standard*):

| Fase | Foco Operacional | Metodologia / Padrão |
|------|------------------|----------------------|
| **1. Intelligence Gathering** | Reconhecimento passivo e ativo, OSINT, ASN, DNS | Subfinder, Httpx, Amass, Shodan |
| **2. Threat Modeling** | Mapeamento de endpoints, superfícies de ataque e ativos | OWASP WSTG, Katana, Gau |
| **3. Vulnerability Analysis** | Identificação e priorização de vetores de entrada | Ffuf, Burp Suite, Nuclei |
| **4. Exploitation** | Prova de Conceito (PoC) e exploração controlada | Payloads dedicados, scripts custom |
| **5. Post-Exploitation & Reporting** | Documentação, impacto de negócio e remediação | CVSS v3.1, Relatórios executivos |

---

### 🌐 [Essential Web Hacking](./Essential%20Web%20Hacking.md)
Abordagem teórica aprofundada, funcionamento sob a ótica de arquitetura e mitigação de vulnerabilidades web:

- **Injeções de Código:** SQL Injection (UNION-based, Time-Based Blind, Error-based, WebShell via `INTO OUTFILE`), Command Injection (quebra de comandos e sanitização de shell).
- **Controle de Acesso & Lógica:** IDOR, CSRF (tokens e atributos `SameSite`), Autenticação e Autorização.
- **Inclusão de Arquivos & SSRF:** Local File Inclusion (LFI com wrappers PHP), Remote File Inclusion (RFI), Server-Side Request Forgery (SSRF interno).
- **Client-Side:** Cross-Site Scripting (XSS Reflected, Stored e DOM-based, impacto de CSP e cookies `HttpOnly`).
- **Pós-Exploração Web:** Estabilização e upgrade de reverse shells para sessões interativas completas (TTY sanitizado com `python pty` + `stty`).

---

### 🔌 [OWASP API Security Top 10](./OWASP%20API%20Top%2010.md)
Guia dedicado à segurança de APIs REST, GraphQL e microserviços:

| Identificador | Vulnerabilidade | Foco de Exploração |
|---------------|-----------------|--------------------|
| **API1:2023** | **BOLA (Broken Object Level Authorization)** | Manipulação de IDs sequenciais/UUIDs em rotas, bodies e métodos HTTP |
| **API2:2023** | **Broken Authentication** | Falhas em JWT (`alg: none`, secrets fracos, key confusion), falta de rate limit |
| **API3:2023** | **BOPLA (Broken Object Property Level Authorization)** | Mass Assignment e exposição excessiva de dados sensíveis em responses |
| **API4:2023** | **Unrestricted Resource Consumption** | DoS de API por ausência de paginação estrita, memory exhaustion, rate limit bypass |
| **API5:2023** | **BFLA (Broken Function Level Authorization)** | Acesso horizontal e vertical a endpoints administrativos |
| **API6:2023** | **Unrestricted Access to Sensitive Business Flows** | Automação e abuso de regras de negócio (compras, cupons, scrapers) |
| **API7:2023** | **Server-Side Request Forgery (SSRF)** | Injeção em webhooks, exportadores de PDF e pivoting para o `docker.sock` |
| **API8:2023** | **Security Misconfiguration** | Headers inseguros, CORS permissivo, endpoints legados e verbos não filtrados |
| **API9:2023** | **Improper Inventory Management** | Shadow APIs, rotas antigas/beta desprotegidas (`/v1` vs `/v2`) |
| **API10:2023** | **Unsafe Consumption of APIs** | Confiança cega em integrações terceiras sem higienização de payload |

---

### 💣 [Payloads - Web Hacking](./Payloads%20-%20Web%20Hacking.md)
Arsenal tático estruturado para testes práticos, categorizado por contexto, tecnologia e blindagem de WAF:

- **SQL Injection:** Cheatsheets de MySQL, PostgreSQL, MSSQL e SQLite; payloads para extração cega, error-based e injeções de arquivo.
- **XSS:** Vetores modernos contornando filtros comuns, polyglots e vetores sem tags `<script>` (event handlers inline).
- **LFI / Path Traversal:** Payloads com URL encode, bypass de extensão nula, wrappers PHP (`php://filter/convert.base64-encode/resource=...`) e log poisoning.
- **Command Injection:** Bypass de filtros de espaço em branco via `$IFS`, expansão de chaves `{cat,/etc/passwd}` e entrega em Base64.
- **Reverse Shells:** One-liners validados em Bash, Python, PHP, Perl, Ruby, Netcat e Socat.

---

### 🛡️ [WAF Bypass](./WAF%20Bypass.md)
Metodologia técnica de evasão contra Web Application Firewalls (*Cloudflare, AWS WAF, Akamai, Imperva, ModSecurity*):

- **Modelos de Discrepância de Parsing:** Diferença de interpretação entre o proxy/WAF e o servidor de aplicação back-end.
- **Técnicas de Evasão:** Múltiplos encodings (URL, Double URL, Unicode, Hex, HTML Entities), comentários SQL inline (`/*!50000SELECT*/`), variação de case e fragmentação de pacotes HTTP.
- **HTTP Request Smuggling:** Desalinhamento entre cabeçalhos `Content-Length` e `Transfer-Encoding: chunked`.

---

### 🐧 [Linux Privilege Escalation](./Linux%20Privilege%20Escalation.md)
Trilha exaustiva de auditoria e escalação de privilégios de usuário comum para `root` em ambientes GNU/Linux:

- **Reconhecimento Interno:** Enumeração com scripts automatizados (`LinPEAS`, `pspy`) e checagens manuais essenciais.
- **Vulnerabilidades de Permissão:** SUID/SGID abusáveis via [GTFOBins](https://gtfobins.github.io/), configurações incorretas em `/etc/sudoers`.
- **Linux Capabilities:** Exploração de flags especiais (`cap_setuid`, `cap_dac_read_search`, `cap_net_admin`).
- **Automações e Tarefas:** Cron jobs, timers do systemd, scripts graváveis em diretórios do sistema e PATH hijacking.
- **Ambientes Containerizados:** Identificação de containers Docker, montagem indevida de sockets (`/var/run/docker.sock`) e técnicas de container escape.

---

### ⌨️ [Comandos Linux & Ambiente de Ataque](./Kali%20linux%20%26%20Arch%20e%20minhas%20anota%C3%A7%C3%B5es%20basicas.md)
Dotfiles, configurações operacionais e comandos táticos para sistemas Arch Linux e Kali:

| Domínio | Comandos / Ferramental |
|---------|------------------------|
| **Navegação & Busca** | `find`, `grep`, `awk`, `sed`, `cut`, `sort -u` |
| **Rede & Probing** | `ss -tulpn`, `ip a`, `curl -I`, `wget`, `tcpdump` |
| **Handlers de Shell** | `nc -lvnp`, `Penelope`, `pwncat-cs` |
| **Pivoting & Tunelamento** | SSH Local/Remote/Dynamic port forwarding, `chisel`, `socat` |
| **Auditoria de Permissões** | Tabela octal, visualização de permissões e checagens de ACL |

---

### 🌐 [REDES](./REDES%20-.md)
Fundamentos de redes aplicados a testes de intrusão e análise de tráfego:

- **Camadas e Protocolos:** Modelo OSI vs. TCP/IP, Three-Way Handshake (`SYN`, `SYN-ACK`, `ACK`).
- **Protocolos de Aplicação e Transporte:** DNS, HTTP/1.1 vs. HTTP/2, HTTPS/TLS, UDP, TCP.
- **Resolução de Conflitos e Roteamento:** ARP Poisoning, CIDR, sub-redes e análise de headers com Wireshark/tcpdump.
- **Mapa Mental:** Integração com o arquivo visual interativo [`REDES - GERAL.canvas`](./REDES%20-%20GERAL.canvas).

---

### 📅 [Plano de Estudos](./plano%20de%20Estudos.md)
Trilha técnica contínua estruturada:

- **HackTheBox & Academy:** Foco na certificação CWES (*Certified Web Exploitation Specialist*) e resolução de máquinas da comunidade.
- **Hacking Club:** Resoluções práticas de CTFs e explorações com cenários realistas de infraestrutura e aplicações corporativas.
- **PortSwigger Web Security Academy:** Laboratórios cirúrgicos para aprofundamento das falhas reportadas no OWASP Top 10.

---

## 🛠️ Stack & Arsenal de Ferramental

<div align="center">

| Categoria | Ferramental Homologado |
|-----------|------------------------|
| **Reconhecimento & OSINT** | `subfinder` `httpx` `amass` `shodan` `theHarvester` `crt.sh` |
| **Crawling & Content Discovery** | `katana` `gau` `ffuf` `gobuster` `dirsearch` |
| **Web & API Assessment** | `Burp Suite Professional` `nuclei` `sqlmap` `kxss` `SecretFinder` |
| **Listeners & Shell Handlers** | `Penelope` `pwncat-cs` `netcat` `rlwrap` |
| **Escalação de Privilégios** | `LinPEAS` `pspy` `GTFOBins` |
| **Sistemas Operacionais** | `Arch Linux` • `Kali Linux` |

</div>

---

## 🏆 Writeups Documentados

Writeups completos com cadeia de ataque, PoCs e mitigação disponíveis no repositório irmão [GabrielMarques1/WriteUps](https://github.com/GabrielMarques1/WriteUps) e no site oficial:

| Máquina / Alvo | Plataforma | Dificuldade | Vetores Principais | Status |
|----------------|------------|:-----------:|---------------------|:------:|
| **Cyber-News** | Hacking Club | Difícil | BOPLA, Mass Assignment, BOLA, WebDAV RCE, Docker Escape | 🟢 Rooted |
| **Retro** | Hacking Club | Médio | OTP Bypass, IDOR, Command Injection, ROM Hijacking | 🟢 Rooted |
| **Poisoning** | Hacking Club | Médio | LFI, Log Poisoning, RCE, `cap_setuid` PrivEsc | 🟢 Rooted |
| **Lion** | Hacking Club | Médio | SQLi UNION-based, WebShell via `INTO OUTFILE`, Crontab | 🟢 Rooted |
| **Calc** | Hacking Club | Médio | Command Injection (PHP `eval`), Python Library Hijacking | 🟢 Rooted |
| **Laravel-Time** | Hacking Club | Médio | Time-Based Blind SQLi, `LOAD_FILE`, SMB, Crontab | 🟢 Rooted |

---

## ⚙️ Como Utilizar Este Repositório

O repositório foi arquitetado para navegação via **[Obsidian](https://obsidian.md/)**, preservando links bidirecionais (`[[...]]`), tags e diagramas Canvas:

```bash
# Clonar o repositório
git clone https://github.com/GabrielMarques1/Cyber-security.git

# Abrir no Obsidian:
# File -> Open Vault -> Selecionar a pasta ~/Cyber-security/
```

> **Acesso Web:** Todo este vault é sincronizado automaticamente e publicado em formato de artigos e cheatsheets no portfólio oficial: **[https://www.gabrielsec.live/notes](https://www.gabrielsec.live/notes)**.

---

## 📌 Referências & Recursos Oficiais

- 📖 [OWASP Web Security Testing Guide (WSTG)](https://owasp.org/www-project-web-security-testing-guide/)
- 🔌 [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- 🎯 [MITRE ATT&CK Framework](https://attack.mitre.org/)
- 📜 [PTES — Penetration Testing Execution Standard](http://www.pentest-standard.org/)
- 🎓 [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- 🎓 [Hack The Box Academy](https://academy.hackthebox.com/)
- 🎓 [Hacking Club](https://app.hackingclub.com/)

---

<div align="center">

**Gabriel (edimargabriel)**  
*Segurança Ofensiva Ética • Pentest • CTF*  
🌐 [gabrielsec.live](https://www.gabrielsec.live) • 💼 [LinkedIn](https://linkedin.com/in/edimargabriel) • 🐙 [GitHub](https://github.com/GabrielMarques1)

</div>
