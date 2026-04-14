# Módulo: Metodologia de Pentest

## Fluxo de fases
- Sempre siga a ordem: Recon → Enumeração → Exploração → Pós-Exploração → Reporting.
- Não pule fases. Se a exploração falhar, volte para enumeração antes de tentar outra coisa.
- Documente cada descoberta assim que confirmada.

## Recon
- Ao receber um alvo (IP, domínio, URL), execute automaticamente:
  - nmap (portas, versões, scripts padrão)
  - WHOIS e DNS (subdomínios, registros, zone transfer)
  - Headers HTTP e detecção de tecnologias (Wappalyzer, whatweb)
  - Busca por informações públicas (Google dorks, Shodan, crt.sh)

## Enumeração por serviço
- **HTTP/HTTPS (80/443):** diretórios (gobuster/ffuf), vhosts, parâmetros, endpoints de API, robots.txt, sitemap, tecnologias do backend, headers de segurança.
- **SMB (445):** shares anônimos (smbclient, enum4linux), versão do Samba, permissões, null session.
- **FTP (21):** anonymous login, versão, arquivos expostos, writable dirs.
- **SSH (22):** versão, auth methods, brute-force se permitido, chaves expostas.
- **MySQL/MSSQL (3306/1433):** credenciais default, databases, acesso remoto, UDF.
- **DNS (53):** zone transfer (dig axfr), subdomínios, registros MX/TXT/CNAME.
- **SNMP (161):** community strings, system info, interfaces, processos.
- **RDP (3389):** versão, NLA, brute-force se permitido.
- **Redis (6379):** acesso sem auth, config get, escrita de arquivos.
- **WinRM (5985):** credenciais, evil-winrm, PowerShell remoting.

## Exploração
- Antes de gerar exploit, mapeie: CVE relacionado caso exista, vetor de ataque, dependências e ambiente alvo (OS, versão do serviço).
- Sempre teste o exploit em ambiente controlado antes de declarar sucesso.
- Priorize exploits confiáveis e conhecidos antes de tentar zero-days ou variações instáveis.

## Pós-Exploração
- Após acesso inicial, execute imediatamente:
  - `whoami`, `id`, `hostname`, `ip a`, `uname -a`
  - Verificar shell estável (upgrade com python pty se necessário)
  - Persistência se aplicável
  - Enumeração local para privesc
  - Coleta de credenciais e hashes
  - Mapeamento de rede interna para lateral movement

## Reporting
- Ao finalizar, gere um mini-relatório com: vulnerabilidade, severidade (CVSS estimado), prova de conceito, impacto real e mitigação recomendada.
