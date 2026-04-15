<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=28&pause=1000&color=00FF41&center=true&vCenter=true&width=600&lines=Cyber+Security+Notes;Offensive+Security+%F0%9F%94%A5;Bug+Bounty+%7C+Pentest+%7C+CTF" alt="Typing SVG" />

<br/>

<img src="https://img.shields.io/badge/Kali%20Linux-557C94?style=for-the-badge&logo=kalilinux&logoColor=white"/>
<img src="https://img.shields.io/badge/Arch%20Linux-1793D1?style=for-the-badge&logo=archlinux&logoColor=white"/>
<img src="https://img.shields.io/badge/Bug%20Bounty-FF6B6B?style=for-the-badge&logo=hackerone&logoColor=white"/>
<img src="https://img.shields.io/badge/Pentest-00FF41?style=for-the-badge&logo=metasploit&logoColor=black"/>
<img src="https://img.shields.io/badge/CTF-FFD700?style=for-the-badge&logo=hackthebox&logoColor=black"/>

<br/><br/>

> **Repositório de anotações, payloads e metodologias de segurança ofensiva.**  
> Estudo contínuo de Bug Bounty, Web Hacking, Linux PrivEsc e mais.

</div>

---

## 📁 Estrutura do Repositório

```
📂 Cyber-security/
├── 📄 Process of Hacking.md          → Metodologia completa de Bug Bounty
├── 📄 Essential Web Hacking.md       → Teoria das vulnerabilidades web
├── 📄 Payloads - Web Hacking.md      → Payloads prontos (XSS, SQLi, SSRF, WAF bypass...)
├── 📄 Linux Privilege Escalation.md  → Guia completo de PrivEsc em Linux
├── 📄 Kali linux & Arch - Anotações  → Comandos essenciais de Linux para hacking
├── 📄 REDES -.md                     → Fundamentos de redes (TCP/IP, OSI, DNS, HTTP)
└── 🗺️  REDES - GERAL.canvas          → Diagrama visual de redes (Obsidian)
```

---

## 📚 Conteúdo

### 🎯 [Process of Hacking](./Process%20of%20Hacking.md)
Metodologia estruturada para Bug Bounty e pentest:

| Fase | Descrição |
|------|-----------|
| **1. Recon** | Subdomain discovery, content discovery, application mapping |
| **2. Análise** | Vulnerability assessment com base no OWASP WSTG |
| **3. PoC** | Construção de prova de conceito reproduzível |
| **4. Report** | Template profissional de relatório de vulnerabilidade |

**Ferramentas cobertas:** `subfinder`, `httpx`, `gau`, `katana`, `ffuf`, `kxss`, `SecretFinder`

---

### 🌐 [Essential Web Hacking](./Essential%20Web%20Hacking.md)
Teoria aprofundada das principais vulnerabilidades web:

`SQLi` • `XSS` • `CSRF` • `SSRF` • `IDOR` • `LFI/RFI` • `Command Injection` • `Auth Bypass` • `Open Redirect` • `XXE`

---

### 💣 [Payloads - Web Hacking](./Payloads%20-%20Web%20Hacking.md)
Coleção de payloads prontos para uso, incluindo:

- **WAF Bypass** — Técnicas de evasão de firewalls de aplicação
- **XSS** — Reflected, Stored, DOM-based
- **SQL Injection** — Union-based, Blind, Error-based
- **SSRF** — Bypass de filtros e pivoting interno
- **Command Injection** — Linux/Windows
- **Linux PrivEsc Payloads** — One-liners de escalação de privilégios

---

### 🐧 [Linux Privilege Escalation](./Linux%20Privilege%20Escalation.md)
Guia completo de escalação de privilégios em Linux:

```
SUID/GUID  →  Sudo Misconfigs  →  Cron Jobs  →  Writable Files
Capabilities  →  NFS  →  PATH Hijacking  →  Kernel Exploits
```

---

### ⌨️ [Comandos Linux para Hacking](./Kali%20linux%20%26%20Arch%20e%20minhas%20anota%C3%A7%C3%B5es%20basicas.md)
Referência rápida cobrindo:

| Categoria | Conteúdo |
|-----------|----------|
| Navegação | `pwd`, `ls`, `cd`, flags essenciais |
| Manipulação | `chmod`, `chown`, `find`, `grep` |
| Rede | `netstat`, `ss`, `curl`, `wget`, `ip` |
| Processos | `ps aux`, `kill`, `htop` |
| Permissões | Tabela octal, SUID/GUID, explicação detalhada |
| **Penelope** | Handler de reverse shell avançado — guia completo |

---

### 🌐 [REDES](./REDES%20-.md)
Fundamentos de redes aplicados à segurança:

`Modelo OSI` • `TCP/IP` • `DNS` • `HTTP/HTTPS` • `ARP` • `Sub-redes`

---

## 🛠️ Stack de Ferramentas

<div align="center">

| Categoria | Ferramentas |
|-----------|-------------|
| **Recon** | `subfinder` `httpx` `amass` `shodan` |
| **Web** | `Burp Suite` `ffuf` `nuclei` `sqlmap` |
| **Shells** | `Penelope` `netcat` `pwncat` |
| **PrivEsc** | `LinPEAS` `LinEnum` `pspy` |
| **OSINT** | `theHarvester` `maltego` `recon-ng` |
| **Platform** | `Kali Linux` `Arch + BlackArch` |

</div>

---

## 📊 Plataformas de Bug Bounty

<div align="center">

[![HackerOne](https://img.shields.io/badge/HackerOne-494649?style=for-the-badge&logo=hackerone&logoColor=white)](https://www.hackerone.com/)
[![Bugcrowd](https://img.shields.io/badge/Bugcrowd-F26822?style=for-the-badge&logo=bugcrowd&logoColor=white)](https://www.bugcrowd.com/)
[![Intigriti](https://img.shields.io/badge/Intigriti-1A1A2E?style=for-the-badge&logo=intigriti&logoColor=white)](https://www.intigriti.com/)
[![BugHunt](https://img.shields.io/badge/BugHunt%20🇧🇷-00C853?style=for-the-badge)](https://bughunt.com.br/)

</div>

---

## 🏆 Writeups

| Máquina/Desafio | Plataforma | Dificuldade | Link |
|-----------------|------------|-------------|------|
| Laravel-Time | — | Médio | [Writeup](./Writeup%20%E2%80%94%20M%C3%A1quina%20Laravel-Time.md) |

---

## ⚙️ Como usar este repositório

Este repositório foi construído para ser aberto no **[Obsidian](https://obsidian.md/)** — os links entre notas, canvas e callouts funcionam nativamente nele.

```bash
# Clonar o repositório
git clone https://github.com/GabrielMarques1/Cyber-security.git

# Abrir no Obsidian
# File → Open Vault → selecionar a pasta clonada
```

> Também pode ser lido diretamente aqui no GitHub — todo o conteúdo está em Markdown padrão.

---

## 📌 Referências de Estudo

- 🎓 [Hacking Club](https://app.hackingclub.com/) — Plataforma de treinamento ofensivo BR
- 📖 [OWASP WSTG](https://owasp.org/www-project-web-security-testing-guide/) — Guia de testes de segurança web
- 🔧 [Penelope Shell Handler](https://github.com/brightio/penelope) — Handler avançado de reverse shells
- 📡 [NotebookLM](https://notebooklm.google.com/) — IA para estudo e resumo de conteúdos

---

<div align="center">

**Feito com 🖤 e muita curiosidade**

*"Hack the planet — de forma ética e autorizada."*

</div>
