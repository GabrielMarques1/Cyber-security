# 🌐 REDES — Fundamentos de Redes de Computadores

[[REDES - GERAL.canvas|REDES - GERAL]]

> Anotações sobre protocolos de rede, modelos de comunicação e conceitos fundamentais para hacking e segurança.

**Referência:** https://www.guiafoca.org/guiaonline/avancado/ch04s02.html

---

## 📐 Modelo OSI — 7 Camadas

> O modelo OSI (Open Systems Interconnection) é o modelo **teórico** que explica como dados trafegam entre dois dispositivos em uma rede. Cada camada tem uma responsabilidade específica.

| # | Camada | Função | Protocolo/Exemplo |
|---|---|---|---|
| 7 | **Aplicação** | Interface com o usuário (browser, email) | HTTP, HTTPS, FTP, SSH, DNS, SMTP |
| 6 | **Apresentação** | Criptografia, encoding, compressão | SSL/TLS, JPEG, ASCII |
| 5 | **Sessão** | Gerencia conexões (abertura, manutenção, fechamento) | NetBIOS, RPC |
| 4 | **Transporte** | Entrega confiável (TCP) ou rápida (UDP) dos dados | TCP, UDP |
| 3 | **Rede** | Endereçamento lógico e roteamento entre redes | IP, ICMP, ARP |
| 2 | **Enlace** | Endereçamento físico (MAC) e frames | Ethernet, Wi-Fi, Switch |
| 1 | **Física** | Bits no meio físico (cabos, sinais elétricos) | Cabos, Hubs, Sinais |

> **Dica para memorizar (de cima pra baixo):** **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

> **Por que isso importa em hacking?** Cada camada tem vulnerabilidades diferentes — ARP Spoofing atua na camada 2/3, SQL Injection na camada 7, sniffing na camada 2, etc.

---

## 🔗 Modelo TCP/IP

> O modelo **prático** da internet. Simplifica o OSI em 4 camadas. É o que realmente roda.

| Camada TCP/IP | Equivalente OSI | Protocolos |
|---|---|---|
| **Aplicação** | Camadas 5, 6, 7 | HTTP, FTP, SSH, DNS, SMTP |
| **Transporte** | Camada 4 | TCP, UDP |
| **Internet** | Camada 3 | IP, ICMP, ARP |
| **Acesso à Rede** | Camadas 1, 2 | Ethernet, Wi-Fi |

- **IP** = Protocolo da Internet — Cada dispositivo conectado tem um endereço IP (seu "endereço" na rede)
- **TCP** = Protocolo de Controle de Transmissão — Garante que os dados cheguem completos e na ordem correta

> **Regra fundamental:** Redes externas (públicas) **NÃO conseguem se conectar diretamente** a redes internas (privadas). Para isso existe o NAT.

---

## 📡 TCP vs UDP

### TCP — Transmission Control Protocol
Protocolo **confiável** e **orientado a conexão**. Garante a entrega dos dados.

**Three-Way Handshake (Aperto de mão em 3 passos):**
```
Cliente  ──── SYN ────────→  Servidor     (1. "Quero conectar")
Cliente  ←── SYN/ACK ─────  Servidor     (2. "OK, aceito")
Cliente  ──── ACK ────────→  Servidor     (3. "Confirmado!")
         ═══ CONEXÃO ESTABELECIDA ═══
```

Quando queremos **nos conectar a um servidor**, temos:
- **Porta de origem** (a nossa — aleatória, ex: porta 52341)
- **Porta de destino** (do servidor — porta padrão do serviço, ex: 80 para HTTP)

A conexão é estabelecida entre essas duas portas — é onde acontece a transmissão dos dados.

> **Em hacking:** O SYN Scan do Nmap (`-sS`) envia um SYN mas NÃO completa o handshake (não envia o ACK final) — por isso é chamado de "stealth scan".

### UDP — User Datagram Protocol
Protocolo **rápido** mas **sem garantia de entrega**. Não faz handshake.

| Característica | TCP | UDP |
|---|---|---|
| Confiável? | ✅ Sim (confirma entrega) | ❌ Não |
| Velocidade | Mais lento | Mais rápido |
| Handshake | Sim (3-way) | Não |
| Uso comum | HTTP, SSH, FTP, SMTP | DNS, VoIP, jogos, streaming, DHCP |

> **Em hacking:** UDP scan no Nmap é `nmap -sU`. É muito mais lento porque o UDP não responde quando a porta está aberta (silêncio = aberta/filtrada).

---

## 🔌 Portas — Números Importantes

Cada serviço de rede escuta em uma porta específica. Conhecer as portas é essencial para enumeração.

| Porta | Protocolo | Serviço | Notas para Hacking |
|---|---|---|---|
| 21 | TCP | **FTP** | Login anônimo? `anonymous:anonymous` |
| 22 | TCP | **SSH** | Brute force, chaves expostas |
| 23 | TCP | **Telnet** | Sem criptografia! Fácil de sniffar |
| 25 | TCP | **SMTP** | Envio de emails, enumeração de usuários |
| 53 | TCP/UDP | **DNS** | Zone transfer? DNS rebinding? |
| 80 | TCP | **HTTP** | Aplicação web (sem criptografia) |
| 110 | TCP | **POP3** | Recebimento de emails |
| 139/445 | TCP | **SMB** | Compartilhamento de arquivos Windows, EternalBlue |
| 443 | TCP | **HTTPS** | HTTP com criptografia (TLS/SSL) |
| 3306 | TCP | **MySQL** | Banco de dados exposto? |
| 3389 | TCP | **RDP** | Remote Desktop (Windows) — BlueKeep? |
| 5432 | TCP | **PostgreSQL** | Banco de dados exposto? |
| 6379 | TCP | **Redis** | Sem autenticação por padrão! |
| 8080 | TCP | **HTTP Alt** | Tomcat, painéis de admin |
| 27017 | TCP | **MongoDB** | Sem autenticação por padrão! |

> **Lembre:** Portas de 0-1023 são **well-known** (precisam de root). De 1024-65535 são portas dinâmicas/registradas.

---

## 🏠 Endereçamento IP e Subnetting

### IPs Privados vs Públicos

**Endereços Privados** (rede interna — sua casa, empresa):
| Faixa | Classe | Uso Comum |
|---|---|---|
| `10.0.0.0` – `10.255.255.255` | Classe A | Redes grandes/corporativas |
| `172.16.0.0` – `172.31.255.255` | Classe B | Redes médias |
| `192.168.0.0` – `192.168.255.255` | Classe C | Redes domésticas |

**Endereços Públicos:** Google, Instagram, etc. — roteáveis na internet.

> **Regra:** Endereços públicos **NÃO podem se conectar diretamente** a endereços privados. Para se conectar a redes públicas, **precisamos ter uma conexão pré-aberta** para que seja estabelecida. O NAT faz essa tradução.

### CIDR — Máscara de Sub-rede

O `/XX` depois do IP indica quantos bits são da **rede** (os restantes são de **hosts**).

| CIDR | Máscara | Hosts disponíveis | Exemplo |
|---|---|---|---|
| `/8` | 255.0.0.0 | ~16 milhões | `10.0.0.0/8` |
| `/16` | 255.255.0.0 | ~65.000 | `172.16.0.0/16` |
| `/24` | 255.255.255.0 | 254 | `192.168.1.0/24` |
| `/32` | 255.255.255.255 | 1 (host único) | `10.10.10.5/32` |

> **Em hacking:** Quando vê `0.0.0.0/0` em uma regra de firewall, significa **qualquer endereço** (toda a internet).

---

## 📡 Protocolos de Rede

---

### ARP — Address Resolution Protocol

ARP funciona como uma **tabela** que mapeia endereços IP (virtuais) para endereços MAC (físicos). Resolve a pergunta: *"Qual é o MAC do dispositivo com IP X.X.X.X?"*

- **MAC** = Endereço físico (gravado no hardware da placa de rede, ex: `AA:BB:CC:DD:EE:FF`)
- **IP** = Endereço virtual/lógico (atribuído pela rede)

**Como funciona:**
1. O computador (Porta 2) quer falar com a TV (Porta 1)
2. Envia um **ARP Request** broadcast: *"Quem tem o IP 192.168.0.5?"*
3. A rede toda recebe, mas apenas a TV responde
4. TV envia **ARP Reply**: *"Sou eu! Meu MAC é XX:XX:XX:XX"*
5. A partir de agora, os pacotes vão direto para aquele MAC — sem broadcast desnecessário

> **⚠️ Problema de segurança:** Em empresas sem firewalls bem estruturados, mesmo com firewall no gateway (roteador), se os IPs estiverem na **mesma rede**, a comunicação ocorre diretamente — **negligenciando o firewall**. Por isso existem VLANs e segmentação de rede.

> **ARP Spoofing/Poisoning:** Um atacante pode enviar ARP Replies falsos, dizendo "Eu sou o gateway!" — fazendo todo o tráfego da rede passar por ele (Man-in-the-Middle).

---

### Firewall — Regras Básicas

Formato padrão de regra:
```
FROM  →  TO  →  SRC_PORT  →  DEST_PORT  →  ACTION
```

**Implicit Deny** — Regra padrão que rejeita TUDO que não estiver explicitamente permitido:
```
0.0.0.0/0  →  0.0.0.0/0  →  0-65535  →  0-65535  →  DROP/REJECT
```
> **DROP** = Descarta silenciosamente (o atacante não sabe se a porta existe)
> **REJECT** = Recusa e avisa (o atacante sabe que o firewall bloqueou)

---

### NAT — Network Address Translation

A função do NAT é **mascarar** o IP privado, porque o IP externo não se comunica com IP privado diretamente.

**Como funciona:**
1. Computador (`192.168.0.4`) quer acessar o Google
2. Roteador recebe o pacote e **troca o IP de origem** pelo IP público
3. Cria uma entrada na tabela NAT para lembrar de quem enviou
4. Google recebe com o endereço público e responde
5. Roteador recebe a resposta, consulta a tabela, **desmascara** e envia de volta para `192.168.0.4`

| IP_ORIGEM | PORTA_ORIGEM | NOVO_IP (Público) | NOVA_PORTA |
|---|---|---|---|
| 192.168.0.4 | 80 | 2.2.2.5 | 570 |

---

### ISP — Internet Service Provider

Provedor de internet (ex: Vivo, Claro, NET). Conecta nosso roteador à internet pública. É onde geralmente está o BGP.

---

### BGP — Border Gateway Protocol

Protocolo que conecta **redes autônomas** (ISPs, data centers, etc.) entre si. É o protocolo que faz a internet funcionar como um todo.

- **ASN** (Autonomous System Number) — Funciona como um CPF para cada rede autônoma. Identifica e permite troca de rotas.
- BGP é **dinâmico** — as rotas se atualizam automaticamente.

> **Em hacking:** BGP Hijacking é quando um atacante anuncia rotas falsas, desviando tráfego de grandes redes. É um ataque de nível de infraestrutura.

---

### DHCP — Dynamic Host Configuration Protocol

Atribui IPs **automaticamente** para dispositivos que entram na rede. Quando você conecta no Wi-Fi, é o DHCP que te dá um IP.

**O que o DHCP fornece:**
- Endereço IP
- Máscara de sub-rede
- Gateway padrão (endereço do roteador)
- Servidor DNS

> **Em hacking:** DHCP Starvation — esgotar todos os IPs do pool DHCP, impedindo novos dispositivos de se conectarem. DHCP Spoofing — responder antes do servidor real e fornecer um gateway falso (seu PC).

---

### DNS — Domain Name System

Serve como um **"catálogo telefônico"** da internet — traduz nomes de domínio para endereços IP.

```
google.com     →  142.250.79.14
crowsec.com.br →  231.25.12.64
```

**Estrutura:** `NOME-DO-DOMINIO.TLD`
- **TLD** (Top Level Domain) = `.com`, `.br`, `.org`, `.uk`, etc.

**Tipos de registro DNS:**

| Registro | Função | Exemplo |
|---|---|---|
| **A** | Aponta para IPv4 | `google.com → 142.250.79.14` |
| **AAAA** | Aponta para IPv6 | `google.com → 2607:f8b0::` |
| **CNAME** | Aponta para outro domínio | `www.google.com → google.com` |
| **MX** | Servidor de e-mail (Mail Exchange) | `gmail.com → mx.google.com` |
| **NS** | Name Server (quem responde pelo domínio) | `google.com → ns1.google.com` |
| **TXT** | Anotações e validações (SPF, DKIM) | Verificação de domínio |
| **SOA** | Start of Authority (info sobre a zona) | Dados administrativos |
| **PTR** | DNS Reverso (IP → nome) | `142.250.79.14 → google.com` |

**Comandos para consultar DNS:**
```bash
dig A google.com            # Buscar registro A (IPv4)
dig MX google.com           # Buscar servidores de email
dig ANY google.com          # Buscar todos os registros
dig AXFR @ns1.alvo.com alvo.com    # Zone Transfer (se permitido)
nslookup google.com         # Alternativa ao dig
host google.com             # Mais simples ainda
```

> **Em hacking:** Sempre procure pelo registro **A** primeiro. Use `dig` ou `nslookup`. Zone Transfer (`AXFR`) pode expor TODOS os subdomínios se o servidor estiver mal configurado.

---

## 🌐 HTTP — Hyper Text Transfer Protocol

HTTP é o protocolo padrão **cliente-servidor** da Web. Dividido em dois tipos de pacote:
- **HTTP Request** (o que o cliente envia)
- **HTTP Response** (o que o servidor responde)

Usando `nc` (Linux) ou `ncat` (Windows), conseguimos fazer requisições brutas ao servidor.

### HTTP Request
Formato:
```
METHOD URI VERSION (CRLF)
HEADER-NAME: HEADER-VALUE (CRLF)
HEADER-NAME: HEADER-VALUE (CRLF)
(CRLF)
REQUEST-BODY
```

**Exemplo:**
```
GET / HTTP/1.1
Host: google.com
Connection: close

```

### HTTP Response
Formato:
```
VERSION STATUS_CODE MSG_RESPOSTA (CRLF)
HEADER-NAME: HEADER-VALUE (CRLF)
HEADER-NAME: HEADER-VALUE (CRLF)
(CRLF)
RESPONSE-BODY
```

### Métodos HTTP

| Método | Função |
|---|---|
| **GET** | Ler/buscar dados do servidor |
| **POST** | Enviar/gravar dados no servidor |
| **PUT** | Gravar ou atualizar informações (substitui o recurso inteiro) |
| **PATCH** | Atualizar parcialmente um recurso |
| **DELETE** | Deletar informações |
| **OPTIONS** | Descobrir métodos permitidos (útil para recon!) |
| **HEAD** | Igual ao GET mas retorna só os headers (sem body) |

### Status Codes Importantes

| Código | Significado | Importância em Hacking |
|---|---|---|
| **200** | OK — Sucesso | Página acessível |
| **301/302** | Redirect | Para onde redireciona? Open Redirect? |
| **400** | Bad Request | Input malformado |
| **401** | Unauthorized | Precisa de autenticação |
| **403** | Forbidden | Existe mas é proibido — tentar bypass! |
| **404** | Not Found | Página não existe |
| **405** | Method Not Allowed | Tente outro método HTTP |
| **500** | Internal Server Error | Possível vulnerabilidade (SQL Error?) |
| **503** | Service Unavailable | Servidor sobrecarregado |

### Headers HTTP Importantes

| Header | Função | Exemplo |
|---|---|---|
| **Host** | Site que estamos acessando (Virtual Host) | `Host: google.com` |
| **Connection** | Manter ou fechar a conexão | `close` ou `keep-alive` |
| **Content-Type** | Tipo de conteúdo enviado/recebido | `application/json`, `text/html` |
| **Content-Length** | Tamanho do conteúdo em bytes | `Content-Length: 348` |
| **User-Agent** | Identifica quem está acessando (browser/bot) | `Mozilla/5.0...` |
| **Cookie** | Dados de sessão enviados ao servidor | `Cookie: PHPSESSID=abc123` |
| **Set-Cookie** | Servidor define um cookie no cliente | `Set-Cookie: session=xyz` |
| **Authorization** | Token/credencial de autenticação | `Bearer eyJhbGci...` |
| **X-Forwarded-For** | IP real do cliente (atrás de proxy) | `X-Forwarded-For: 127.0.0.1` |
| **Referer** | De onde veio a requisição | `Referer: https://google.com` |

> **Em hacking:** Headers como `X-Forwarded-For: 127.0.0.1` podem ser manipulados para bypass de restrição de IP. O `User-Agent` pode ser usado para Log Poisoning (LFI → RCE).