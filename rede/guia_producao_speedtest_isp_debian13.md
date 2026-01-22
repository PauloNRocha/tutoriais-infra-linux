# Guia de Produção (ISP): Ookla Server (Enterprise) + LibreSpeed no Debian 13 — IPv4/IPv6, nftables e Fail2Ban (sets)

*Atualizado em: 13 de janeiro de 2026*

Este tutorial entrega um passo a passo **completo, detalhado e pronto para produção** para implantar um **servidor de teste de velocidade** em um **Provedor de Internet (ISP)**, com duas frentes:

- **Ookla Server (Enterprise)**: servidor **público** (controle e visibilidade; após aprovação pode aparecer na lista oficial do Speedtest).
- **LibreSpeed**: servidor **interno** (diagnóstico e suporte), acessível **somente** por clientes na rede do provedor (CGNAT/PPPoE/BRAS) e infraestrutura autorizada.

Requisitos atendidos neste guia:
- Debian 13 (Trixie)
- Ambiente de produção real de ISP
- Considera IPv4 e IPv6 em todas as configurações
- Firewall **nftables** com política padrão **DROP**
- Fail2Ban integrado ao **nftables via SETS** (não iptables)

---

## Índice
1. [Arquitetura e conceito (papel de cada serviço)](#1)
2. [Variáveis (substitua pelos seus dados)](#2)
3. [Planejamento (DNS, portas e requisitos do Ookla)](#3)
4. [Instalação base do Debian 13 e pacotes](#4)
5. [Tuning e otimização (obrigatório)](#5)
6. [Firewall com nftables (obrigatório)](#6)
7. [HTTPS com Let’s Encrypt (Certbot) (obrigatório)](#7)
8. [LibreSpeed (interno): instalação, performance e bloqueio externo](#8)
9. [Ookla Server (Enterprise): instalação](#9)
10. [OoklaServer.properties: configuração avançada (obrigatório)](#10)
11. [systemd: start automático e restart automático (obrigatório)](#11)
12. [Validação e publicação do Ookla (passo a passo)](#12)
13. [Fail2Ban + nftables sets (obrigatório)](#13)
14. [Logs, visibilidade e operação (obrigatório)](#14)
15. [Checklist final (produção)](#15)
16. [Troubleshooting (problemas comuns)](#16)
17. [Comandos rápidos (operação diária)](#17)
18. [Referências](#18)

---

## Passo a passo (numerado, do zero à produção)

1) Defina hostnames e redes internas (seção 2).  
2) Configure DNS público do Ookla (A/AAAA) e valide resolução (seção 3.1).  
3) Confirme portas e requisitos (8080/5060 TCP+UDP, 80 TCP quando aplicável) (seção 3.2).  
4) Instale pacotes base (Nginx, Certbot, nftables, Fail2Ban) e habilite NTP (seção 4).  
5) Aplique tuning (sysctl + NOFILE) e reinicie o necessário (seção 5).  
6) Aplique nftables (DROP por padrão + sets + limites conservadores) (seção 6).  
7) Emita TLS do **OOKLA_FQDN** (HTTP-01) e do **LIBRE_FQDN** (DNS-01) com Certbot (seção 7).  
8) Instale LibreSpeed (interno) e valide “interno OK / externo bloqueado” (seção 8).  
9) Instale o Ookla Server (Enterprise) e valide portas em escuta (seção 9).  
10) Ajuste `OoklaServer.properties` (IPv6, portas, logs, threads) (seção 10).  
11) Garanta systemd (start/restart automático e limites) (seção 11).  
12) Rode Server Tester, registre, submeta e acompanhe aprovação/publicação (seção 12).  
13) Configure Fail2Ban com actions em sets, `ignoreip` e recidive (seção 13).  
14) Faça as validações finais e monitore logs (seções 14 e 15).

<a id="1"></a>
## 1) Arquitetura e conceito (papel de cada serviço)

### 1.1 Ookla Server (Enterprise) — **público**

- Objetivo: ser um **servidor público controlado** para medições oficiais.
- Característica: precisa estar acessível publicamente nas portas exigidas para testes e monitoramento.
- Resultado esperado: após cadastro, submissão e aprovação, pode aparecer no ecossistema oficial.

### 1.2 LibreSpeed — **interno**

- Objetivo: ferramenta **interna** para diagnóstico (suporte/NOC) e teste do **trecho dentro da rede do ISP**.
- Regras:
  - deve ser acessível apenas por clientes e redes internas autorizadas (CGNAT/PPPoE/BRAS/NOC);
  - não deve ser público/não deve ficar exposto para a internet.

---

<a id="2"></a>
## 2) Variáveis (substitua pelos seus dados)

Defina (exemplos):

- Hostname público do Ookla:
  - `OOKLA_FQDN="speedtest-sp01.seudominio.com.br"`
- Hostname interno do LibreSpeed:
  - `LIBRE_FQDN="speedtest-interno.seudominio.com.br"`
- Porta interna do LibreSpeed (para manter 443 livre no host público do Ookla):
  - `LIBRE_PORT="8443"`
- IPs públicos:
  - `IPV4_PUBLICO="203.0.113.10"`
  - `IPV6_PUBLICO="2001:db8:1234::10"`
- Redes que podem acessar o LibreSpeed (exemplos):
  - CGNAT: `100.64.0.0/10`
  - PPPoE/Clientes públicos (se existirem): `198.51.100.0/24`
  - Gerência/NOC: `192.0.2.0/24`
  - IPv6 clientes: `2001:db8:100::/48`
- Infra crítica que **nunca** deve ser banida:
  - BRAS/BNG/CGNAT: `198.51.100.10/32`, `198.51.100.11/32` (exemplos)

Dica: para copiar/colar os comandos deste tutorial sem ficar editando tudo, exporte as variáveis na sua sessão:

```bash
export OOKLA_FQDN="speedtest-sp01.seudominio.com.br"
export LIBRE_FQDN="speedtest-interno.seudominio.com.br"
export LIBRE_PORT="8443"
export IPV4_PUBLICO="203.0.113.10"
export IPV6_PUBLICO="2001:db8:1234::10"
```

---

<a id="3"></a>
## 3) Planejamento (DNS, portas e requisitos do Ookla)

### 3.1 DNS (obrigatório)

Crie registros DNS **públicos** para o Ookla (IPs não são hostname válido para submissão):

```dns
OOKLA_FQDN.  IN A     IPV4_PUBLICO
OOKLA_FQDN.  IN AAAA  IPV6_PUBLICO
```

Valide:
```bash
dig +short "$OOKLA_FQDN" A
dig +short "$OOKLA_FQDN" AAAA
```

Para o LibreSpeed (**interno**), o recomendado em produção é:
- **não publicar** `A/AAAA` do `LIBRE_FQDN` na internet pública (evita scanners e reduz superfície);
- publicar o `LIBRE_FQDN` **apenas no DNS interno** do provedor (split-horizon) ou via DNS dos clientes do ISP;
- quando usar Let’s Encrypt no `LIBRE_FQDN`, usar **DNS-01** (você só precisa publicar o registro `TXT _acme-challenge...`, sem expor o serviço).

### 3.2 Portas exigidas (obrigatório)

Para hosts públicos, o requisito comum inclui (entrada/saída):

- **TCP/UDP 8080** (OoklaServer)
- **TCP/UDP 5060** (OoklaServer)
- **TCP 80** (HTTP Legacy, quando aplicável)

Além disso, pode haver necessidade de saída TCP 80/443 para atualizações e automação de certificado.  
Essas portas precisam estar abertas para qualquer IP público, pois o cliente conecta diretamente.  

### 3.3 Requisitos mínimos de capacidade (produção)

Ponto de partida típico:
- link com **pelo menos 1 Gbps** simétrico (ou superior, conforme seu objetivo);
- servidor com CPU/RAM suficientes para múltiplos testes simultâneos.

---

<a id="4"></a>
## 4) Instalação base do Debian 13 e pacotes

### 4.1 Atualize o sistema

```bash
sudo apt update
sudo apt -y full-upgrade
```

### 4.2 Garanta hora correta (NTP)

```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony
timedatectl status
```

### 4.3 Instale pacotes necessários

```bash
sudo apt install -y \
  openssl \
  ca-certificates curl wget unzip git \
  nginx \
  certbot \
  nftables \
  fail2ban \
  netcat-openbsd \
  dnsutils iproute2 \
  htop iftop iotop
```

> O LibreSpeed pode rodar como site estático (Nginx). PHP é opcional e não é obrigatório para o teste básico.

### 4.4 (Recomendado) Desabilite serviços desnecessários

Liste serviços ativos:
```bash
sudo systemctl list-units --type=service --state=running
```

Exemplos comuns (desabilite somente se existir e se não for usado):
```bash
sudo systemctl disable --now bluetooth || true
sudo systemctl disable --now cups || true
sudo systemctl disable --now avahi-daemon || true
```

### 4.5 Verifique SSD (obrigatório em produção)

```bash
lsblk -d -o NAME,ROTA,TYPE,SIZE,MODEL
```

- `ROTA=0` costuma indicar SSD/NVMe
- `ROTA=1` indica disco rotacional (HDD)

---

<a id="5"></a>
## 5) Tuning e otimização do sistema (obrigatório)

Esta seção foca em throughput e concorrência, sem “tuning mágico”.

### 5.1 sysctl (kernel/rede) + explicação

Crie o arquivo:
```bash
sudo tee /etc/sysctl.d/99-speedtest-isp.conf >/dev/null <<'EOF'
# 1) Backlog de conexões pendentes (aceite em alta concorrência)
net.core.somaxconn = 65535

# 2) Backlog de pacotes na fila do kernel (picos de tráfego)
net.core.netdev_max_backlog = 16384

# 3) Buffers máximos de socket (permite janelas maiores)
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864

# 4) Buffers TCP (min/def/max) — melhora throughput e estabilidade em testes longos
net.ipv4.tcp_rmem = 4096 1048576 67108864
net.ipv4.tcp_wmem = 4096 1048576 67108864

# 5) Range de portas locais (muitas conexões simultâneas)
net.ipv4.ip_local_port_range = 10240 65535

# 6) BBR + fq (controle de congestionamento moderno)
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# 7) Proteção básica contra SYN flood
net.ipv4.tcp_syncookies = 1
EOF
```

Aplicar:
```bash
sudo sysctl --system
sysctl net.ipv4.tcp_congestion_control
```

Impacto prático (resumo):
- `somaxconn` / `netdev_max_backlog`: reduzem drops/rejeições em pico.
- `rmem_max` / `wmem_max` e `tcp_rmem` / `tcp_wmem`: sustentam throughput em múltiplos fluxos.
- `ip_local_port_range`: evita esgotar portas efêmeras.
- `BBR` + `fq`: melhora uso de banda e reduz bufferbloat em vários cenários.

### 5.2 Limites de arquivos (ulimit/NOFILE)

Em testes simultâneos, sockets e arquivos podem exigir mais `nofile`.

Crie:
```bash
sudo mkdir -p /etc/systemd/system.conf.d
sudo tee /etc/systemd/system.conf.d/99-nofile.conf >/dev/null <<'EOF'
[Manager]
DefaultLimitNOFILE=1048576
EOF
```

Aplique:
```bash
sudo systemctl daemon-reexec
```

> Para serviços específicos (Nginx/Ookla), também é possível usar override com `systemctl edit ...`.

### 5.3 Validação do tuning (faça agora, antes de seguir)

1) Confirme BBR e `fq`:
```bash
sysctl net.ipv4.tcp_congestion_control
sysctl net.core.default_qdisc
```

Esperado:
- `net.ipv4.tcp_congestion_control = bbr`
- `net.core.default_qdisc = fq`

2) Confira se os principais valores foram aplicados:
```bash
sysctl net.core.somaxconn net.core.netdev_max_backlog
sysctl net.core.rmem_max net.core.wmem_max
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
sysctl net.ipv4.ip_local_port_range
```

3) (Opcional) Verifique se o módulo do BBR está carregado:
```bash
lsmod | grep -E '^tcp_bbr\\b' || true
```

Se não aparecer nada (pode acontecer dependendo do kernel), tente:
```bash
sudo modprobe tcp_bbr || true
```

4) Confira o limite de arquivos do systemd (isso afeta serviços):
```bash
systemctl show -p DefaultLimitNOFILE systemd
```

> Observação: `ulimit -n` na sua sessão SSH pode continuar baixo até você abrir uma nova sessão. O que importa mesmo é o limite aplicado nos serviços (ex.: `systemctl show nginx -p LimitNOFILE`).

---

<a id="6"></a>
## 6) Firewall com nftables (obrigatório)

Objetivo:
- **DROP por padrão**
- liberar apenas o necessário
- LibreSpeed: permitir **somente redes do ISP**
- Ookla: portas exigidas abertas publicamente
- Fail2Ban: inserir IPs em **sets** do nftables

### 6.1 Defina as redes internas do ISP (obrigatório)

Liste seus blocos reais (IPv4/IPv6) que podem acessar o LibreSpeed (clientes + NOC).

### 6.2 Ruleset completo (com sets do Fail2Ban)

Arquivo: `/etc/nftables.conf`

```bash
sudo nano /etc/nftables.conf
```

Conteúdo (modelo, ajuste as redes):
```nft
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  # ========= SETS DO FAIL2BAN (OBRIGATÓRIO) =========
  set f2b_ookla_v4      { type ipv4_addr; flags timeout; }
  set f2b_librespeed_v4 { type ipv4_addr; flags timeout; }

  set f2b_ookla_v6      { type ipv6_addr; flags timeout; }
  set f2b_librespeed_v6 { type ipv6_addr; flags timeout; }

  # ========= REDES INTERNAS DO ISP (AJUSTE) =========
  set isp_v4 {
    type ipv4_addr; flags interval;
    elements = {
      100.64.0.0/10,      # CGNAT (exemplo)
      192.0.2.0/24,       # NOC/gerência (exemplo)
      198.51.100.0/24     # clientes públicos (exemplo)
    }
  }

  set isp_v6 {
    type ipv6_addr; flags interval;
    elements = {
      2001:db8:100::/48,  # clientes IPv6 (exemplo)
      2001:db8:200::/48   # NOC/gerência (exemplo)
    }
  }

  chain input {
    type filter hook input priority 0;
    policy drop;

    # Loopback e conexões estabelecidas
    iif "lo" accept
    ct state established,related accept

    # Bloqueios do Fail2Ban (drop total)
    ip  saddr @f2b_ookla_v4      drop
    ip  saddr @f2b_librespeed_v4 drop
    ip6 saddr @f2b_ookla_v6      drop
    ip6 saddr @f2b_librespeed_v6 drop

    # ICMP/ICMPv6 (diagnóstico e PMTU)
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept

    # SSH (recomendado: restrinja por IP)
    tcp dport 22 accept

    # HTTP (ACME/Certbot do OOKLA_FQDN e, se necessário, HTTP legacy)
    tcp dport 80 accept

    # HTTPS (443) do OOKLA_FQDN (landing page + “higiene operacional”)
    tcp dport 443 accept

    # LibreSpeed (interno): use porta dedicada para manter “internal-only” no firewall
    tcp dport 8443 ip  saddr @isp_v4 accept
    tcp dport 8443 ip6 saddr @isp_v6 accept

    # ====== Proteção conservadora contra scanner/flood (ookla) ======
    # Regra “pega” só quando excede taxa — para NÃO degradar testes normais.
    tcp dport { 8080, 5060 } ct state new meter ookla_tcp_over  { ip  saddr timeout 1m limit rate over 200/second } log prefix "F2B_OOKLA_TCP " level info drop
    tcp dport { 8080, 5060 } ct state new meter ookla_tcp6_over { ip6 saddr timeout 1m limit rate over 200/second } log prefix "F2B_OOKLA_TCP " level info drop

    udp dport { 8080, 5060 } meter ookla_udp_over  { ip  saddr timeout 1m limit rate over 500/second } log prefix "F2B_OOKLA_UDP " level info drop
    udp dport { 8080, 5060 } meter ookla_udp6_over { ip6 saddr timeout 1m limit rate over 500/second } log prefix "F2B_OOKLA_UDP " level info drop

    # Ookla Server (público): TCP/UDP 8080 e 5060
    tcp dport { 8080, 5060 } accept
    udp dport { 8080, 5060 } accept
  }

  chain forward {
    type filter hook forward priority 0;
    policy drop;
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
```

Aplicar e persistir:
```bash
sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
sudo nft list ruleset
```

> O prefixo `F2B_OOKLA_*` está aqui de propósito: ele gera logs raros (somente quando excede taxa) para o Fail2Ban banir scanners/flood sem derrubar throughput normal.

### 6.3 Validação do firewall (faça agora, antes de seguir)

1) Confirme que a política padrão é `DROP` no `input`:
```bash
sudo nft list chain inet filter input
```

2) Confirme que as regras-chave existem (procure por portas/sets):
```bash
sudo nft list ruleset | grep -E \"(policy drop|f2b_ookla|f2b_librespeed|isp_v4|isp_v6|dport (80|443|8443|8080|5060))\"
```

3) Confirme que as portas locais em escuta fazem sentido:
```bash
sudo ss -tulpn | grep -E ':(80|443|8443|8080|5060)\\b' || true
```

4) Teste de fora (use outra máquina na internet):
```bash
nc -vz "$OOKLA_FQDN" 8080
nc -vz "$OOKLA_FQDN" 5060
nc -vz "$OOKLA_FQDN" 443
```

E valide que o LibreSpeed (porta interna) **não** está exposto:
```bash
nc -vz "$OOKLA_FQDN" "$LIBRE_PORT"
```

> Esperado: as portas 8080/5060/443 respondem; a porta do LibreSpeed (ex.: 8443) deve falhar/timeout quando testada de fora.

---

<a id="7"></a>
## 7) DNS e HTTPS com Let’s Encrypt (Certbot) (obrigatório)

Este guia usa **Certbot** com dois métodos distintos (cada um no seu cenário ideal):

- **OOKLA_FQDN (público)**: HTTP-01 (webroot) → simples, automático e “de mercado”.
- **LIBRE_FQDN (interno, não público)**: DNS-01 → emite certificado **sem expor** o serviço na internet.

> Por design, o LibreSpeed não deve ficar público. Por isso, o método HTTP-01 (porta 80 aberta para o mundo) é adequado para o `OOKLA_FQDN`, mas **não** é o ideal para o `LIBRE_FQDN`.

### 7.1 Preparar webroot do ACME (para o OOKLA_FQDN)

```bash
sudo mkdir -p /var/www/letsencrypt/.well-known/acme-challenge
sudo chown -R www-data:www-data /var/www/letsencrypt
```

### 7.2 Nginx: vhost do OOKLA_FQDN (ACME + redirect) e “default drop” no 80

Objetivo:
- manter a porta 80 aberta para o mundo (por causa do ACME/Let’s Encrypt e, quando aplicável, HTTP legacy);
- **reduzir superfície** no 80: tudo que não for `/.well-known/acme-challenge/` vira redirect ou é descartado;
- evitar que o `LIBRE_FQDN` responda “por acidente” no 80.

Arquivo: `/etc/nginx/sites-available/speedtest-public`

```bash
sudo nano /etc/nginx/sites-available/speedtest-public
```

Conteúdo (ajuste `server_name`):
```nginx
# Catch-all no 80: descarta qualquer coisa que não seja o seu vhost
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}

# OOKLA_FQDN no 80: ACME + redirect para HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name speedtest-sp01.seudominio.com.br;

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

Ativar:
```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/speedtest-public /etc/nginx/sites-enabled/speedtest-public
sudo nginx -t
sudo systemctl reload nginx
```

### 7.3 Certbot: emitir TLS do OOKLA_FQDN (HTTP-01) e validar renovação

1) Emita o certificado:
```bash
sudo certbot certonly --webroot -w /var/www/letsencrypt -d "$OOKLA_FQDN"
```

2) Valide renovação automática:
```bash
sudo certbot renew --dry-run
```

> No Debian, o Certbot normalmente usa `certbot.timer` (systemd) para renovação automática. O `--dry-run` valida o fluxo.

### 7.4 (Recomendado) Nginx: HTTPS “landing page” no OOKLA_FQDN (443)

Por que isso ajuda em produção:
- você padroniza o subdomínio com **HTTPS válido** (página simples com contato/status);
- fica mais “limpo” em auditoria humana e em processos de aprovação (mesmo que o OoklaServer rode em 8080/5060).

1) Crie a landing page:
```bash
sudo mkdir -p /var/www/speedtest-public
sudo tee /var/www/speedtest-public/index.html >/dev/null <<'EOF'
<!doctype html>
<html lang="pt-br">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>Servidor Speedtest do Provedor</title>
  </head>
  <body>
    <h1>Servidor de Speedtest do Provedor</h1>
    <p>Status: operacional</p>
    <p>Contato: noc@seudominio.com.br</p>
  </body>
</html>
EOF
sudo chown -R www-data:www-data /var/www/speedtest-public
```

2) Adicione o bloco HTTPS no mesmo arquivo:

Arquivo: `/etc/nginx/sites-available/speedtest-public`
```bash
sudo nano /etc/nginx/sites-available/speedtest-public
```

Acrescente (abaixo dos blocos 80):
```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name speedtest-sp01.seudominio.com.br;

    ssl_certificate     /etc/letsencrypt/live/speedtest-sp01.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/speedtest-sp01.seudominio.com.br/privkey.pem;

    root /var/www/speedtest-public;
    index index.html;

    access_log /var/log/nginx/speedtest-public.access.log;
    error_log  /var/log/nginx/speedtest-public.error.log;
}
```

3) Recarregue:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

4) Valide:
```bash
curl -I "https://$OOKLA_FQDN"
```

### 7.5 Certbot: emitir TLS do LIBRE_FQDN (DNS-01) sem expor o serviço

Este é o ponto-chave para manter o LibreSpeed realmente interno:
- você **não precisa** publicar `A/AAAA` do `LIBRE_FQDN` na internet;
- você só publica (temporariamente) o TXT de validação do ACME: `_acme-challenge`.

1) Rode Certbot em modo manual DNS-01:
```bash
sudo certbot certonly --manual --preferred-challenges dns -d "$LIBRE_FQDN"
```

2) O Certbot vai mostrar algo como:
- Nome do registro: `_acme-challenge.speedtest-interno.seudominio.com.br`
- Valor (token): uma string longa

3) Crie o registro **TXT** no seu DNS público (zona do domínio), aguarde propagar e continue.

Validação de propagação (substitua o nome):
```bash
dig +short TXT "_acme-challenge.$LIBRE_FQDN"
```

Dica didática (para não errar no painel do DNS):
- **Tipo**: `TXT`
- **Nome/Host**: geralmente `_acme-challenge` (alguns painéis pedem o FQDN completo)
- **Valor/Conteúdo**: o token que o Certbot mostrar
- **TTL**: baixo (ex.: 60–300s) ajuda na renovação/teste

> Automação: o modo `--manual` exige intervenção a cada renovação. Para produção, use um plugin DNS do seu provedor (procure por pacotes `python3-certbot-dns-*`) ou implemente `--manual-auth-hook`/`--manual-cleanup-hook` integrando com a API do seu DNS.

#### 7.5.1 (Recomendado) Automação de DNS-01 com plugin (exemplo Cloudflare)

Se seu domínio está no Cloudflare, você consegue automatizar a renovação com o plugin DNS.

1) Instale o plugin:
```bash
sudo apt update
sudo apt install -y python3-certbot-dns-cloudflare
```

2) Crie um token no Cloudflare com permissão mínima para editar DNS da zona e salve as credenciais:

Arquivo: `/root/.secrets/certbot/cloudflare.ini`
```bash
sudo mkdir -p /root/.secrets/certbot
sudo nano /root/.secrets/certbot/cloudflare.ini
sudo chmod 600 /root/.secrets/certbot/cloudflare.ini
```

Conteúdo (exemplo):
```ini
dns_cloudflare_api_token = SEU_TOKEN_AQUI
```

3) Emita o certificado do `LIBRE_FQDN` via DNS:
```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  -d "$LIBRE_FQDN"
```

4) Valide renovação:
```bash
sudo certbot renew --dry-run
```

### 7.6 Validação de TLS e renovação (faça agora, antes de seguir)

1) Confirme que o timer do Certbot existe e está ativo:
```bash
systemctl status certbot.timer --no-pager || true
systemctl list-timers | grep -i certbot || true
```

2) Veja os certificados instalados e a data de expiração:
```bash
sudo certbot certificates
```

3) Valide o TLS do `OOKLA_FQDN` (de preferência a partir de uma máquina externa):
```bash
openssl s_client -connect "$OOKLA_FQDN:443" -servername "$OOKLA_FQDN" </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

4) Valide localmente o vhost interno do LibreSpeed (SNI + porta):
```bash
curl -kI --resolve "$LIBRE_FQDN:$LIBRE_PORT:127.0.0.1" "https://$LIBRE_FQDN:$LIBRE_PORT"
```

---

<a id="8"></a>
## 8) LibreSpeed (interno): instalação, performance e bloqueio externo

### 8.1 Instalar o LibreSpeed (Nginx, estático)

Publicar em `/var/www/librespeed`:
```bash
sudo test -d /var/www/librespeed && sudo mv /var/www/librespeed "/var/www/librespeed.bak.$(date +%F)" || true
sudo git clone --depth=1 https://github.com/librespeed/speedtest.git /var/www/librespeed
sudo chown -R www-data:www-data /var/www/librespeed
```

### 8.2 Configurar Nginx HTTPS com ACL interna (obrigatório)

Crie um vhost dedicado (porta interna 8443):

> Alternativa (quando você tem IP público dedicado ou VIP interno para o LibreSpeed): você pode manter o LibreSpeed em `443` nesse IP dedicado e liberar no nftables **apenas** para as redes do ISP. Isso permite URL sem `:8443`, mas exige IP extra ou roteamento interno bem definido.

Arquivo: `/etc/nginx/sites-available/librespeed-interno`

```bash
sudo nano /etc/nginx/sites-available/librespeed-interno
```

Conteúdo (HTTPS + ACL):
```nginx
server {
    listen 8443 ssl http2;
    listen [::]:8443 ssl http2;
    server_name speedtest-interno.seudominio.com.br;

    ssl_certificate     /etc/letsencrypt/live/speedtest-interno.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/speedtest-interno.seudominio.com.br/privkey.pem;

    # ACL (ajuste para seus blocos reais)
    allow 100.64.0.0/10;      # CGNAT (exemplo)
    allow 198.51.100.0/24;    # clientes públicos (exemplo)
    allow 192.0.2.0/24;       # NOC/gerência (exemplo)
    allow 2001:db8:100::/48;  # clientes IPv6 (exemplo)
    deny all;

    root /var/www/librespeed;
    index index.html;

    # Performance básica
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 30;

    # Logs (mantidos; use logrotate padrão do Debian)
    access_log /var/log/nginx/librespeed.access.log;
    error_log  /var/log/nginx/librespeed.error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Ativar e recarregar:
```bash
sudo ln -sf /etc/nginx/sites-available/librespeed-interno /etc/nginx/sites-enabled/librespeed-interno
sudo nginx -t
sudo systemctl reload nginx
```

### 8.3 Validar acesso interno e bloqueio externo (obrigatório)

De um IP interno permitido:
```bash
curl -I "https://$LIBRE_FQDN:$LIBRE_PORT"
```

De fora (IP público não pertencente ao ISP), deve falhar (timeout/refused/403 conforme regra):
```bash
curl -I "https://$LIBRE_FQDN:$LIBRE_PORT"
```

> O bloqueio deve existir em duas camadas: **nftables** (porta 8443 só para redes do ISP) + **ACL do Nginx** (allow/deny).

### 8.4 Validações rápidas do LibreSpeed (faça agora)

1) Teste local (no próprio servidor), garantindo SNI correto:
```bash
curl -kI --resolve "$LIBRE_FQDN:$LIBRE_PORT:127.0.0.1" "https://$LIBRE_FQDN:$LIBRE_PORT"
```

2) Verifique se o endpoint está respondendo e se há tráfego:
```bash
sudo tail -n 50 /var/log/nginx/librespeed.access.log
sudo tail -n 50 /var/log/nginx/librespeed.error.log
```

---

<a id="9"></a>
## 9) Ookla Server (Enterprise): instalação (obrigatório)

**Não confundir** com:
- Speedtest CLI (cliente)
- Speedtest Custom

### 9.1 Instalar conforme documentação oficial (obrigatório)

O guia oficial está em:
```text
https://support.ookla.com/hc/en-us/articles/234578528-OoklaServer-Installation-Linux-Unix
```

> Se o portal oficial estiver protegido por challenge (ex.: Cloudflare), use as páginas públicas do portal de suporte e siga os passos equivalentes.

### 9.2 Instalação via script oficial (fluxo comum)

Crie um usuário dedicado:
```bash
sudo useradd -m -s /bin/bash ookla
```

Baixe e execute o instalador:
```bash
sudo -u ookla -i
cd ~
wget https://install.speedtest.net/ooklaserver/ooklaserver.sh
chmod +x ooklaserver.sh
./ooklaserver.sh install
exit
```

> O caminho e o método exato podem variar por versão/contrato. O objetivo aqui é chegar a um `OoklaServer` funcional e um `OoklaServer.properties` no mesmo diretório.

### 9.3 Validar que o serviço está em escuta (IPv4/IPv6)

```bash
sudo ss -tulpn | grep -E ':(8080|5060)\\b'
```

### 9.4 Validações rápidas do Ookla (faça agora)

1) Teste local (porta 8080):
```bash
curl -v http://127.0.0.1:8080/ 2>&1 | head -n 20
```

2) Confirme que o firewall não está bloqueando as portas do Ookla:
```bash
sudo nft list ruleset | grep -E 'dport \\{ 8080, 5060 \\}.*accept'
```

3) Teste externo (de outra máquina na internet):
```bash
curl -v "http://$OOKLA_FQDN:8080/" 2>&1 | head -n 20
```

---

<a id="10"></a>
## 10) OoklaServer.properties: configuração avançada (obrigatório)

O OoklaServer lê as opções do arquivo `OoklaServer.properties` (no mesmo diretório do binário).

### 10.1 Portas e IPv6 (obrigatório)

Pontos importantes:
- hosts públicos exigem **porta 8080** (não altere se quiser listagem pública).
- `tcpPorts` e `udpPorts` costumam ter **5060 e 8080** por padrão.
- IPv6 pode vir comentado e precisa ser explicitamente habilitado.

Exemplo (trechos típicos):
```properties
# Portas
OoklaServer.tcpPorts = 5060, 8080
OoklaServer.udpPorts = 5060, 8080

# IPv6 (remova comentário para habilitar)
# OoklaServer.useIPv6 = True

# Restringir domínios (opcional)
# OoklaServer.allowedDomains = *.ookla.com, *.speedtest.net

# Threads (ajuste com cuidado)
OoklaServer.maxThreads = 512
```

### 10.2 Logs (obrigatório)

Para logs em arquivo (recomendado em produção), o OoklaServer permite alternar o canal:
- `ConsoleChannel` → log no console/journal
- `FileChannel` → log em arquivo (com rotação/compactação)

Exemplo (conceito):
```properties
logging.loggers.app.channel.class = FileChannel
logging.loggers.app.channel.path = ${application.dir}/ooklaserver.log
logging.loggers.app.level = information
```

> Configure rotação/purge no próprio OoklaServer (quando usar FileChannel) para evitar lotar disco.

---

<a id="11"></a>
## 11) systemd: start automático e restart automático (obrigatório)

### 11.1 Se o instalador já criou um serviço

Procure unidades existentes:
```bash
systemctl list-unit-files | grep -i ookla || true
systemctl status ooklaserver --no-pager || true
```

Se existir, crie um override para garantir restart e limites:
```bash
sudo systemctl edit ooklaserver
```

Exemplo de override:
```ini
[Service]
Restart=always
RestartSec=5
LimitNOFILE=1048576
```

### 11.2 Se não existir serviço: criar unit file (modelo)

1) Descubra o diretório do binário e do `OoklaServer.properties`:
```bash
sudo -u ookla -H bash -lc 'find ~ -maxdepth 5 -type f -name OoklaServer -o -name OoklaServer.properties 2>/dev/null'
```

2) Crie um service unit:

Arquivo: `/etc/systemd/system/ooklaserver.service`

```bash
sudo nano /etc/systemd/system/ooklaserver.service
```

Conteúdo (ajuste paths):
```ini
[Unit]
Description=OoklaServer (Enterprise)
Wants=network-online.target
After=network-online.target

[Service]
User=ookla
Group=ookla
WorkingDirectory=/home/ookla/ooklaserver

# Use o método de start recomendado pela sua instalação:
# (A) se houver script oficial:
ExecStart=/home/ookla/ooklaserver/ooklaserver.sh start
ExecStop=/home/ookla/ooklaserver/ooklaserver.sh stop

Restart=always
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

3) Habilitar:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ooklaserver
sudo systemctl status ooklaserver --no-pager
```

### 11.3 Validação do systemd (faça agora)

1) Confirme start automático e logs:
```bash
sudo systemctl is-enabled ooklaserver
sudo journalctl -u ooklaserver -n 50 --no-pager
```

2) Confirme limite de arquivos no serviço:
```bash
systemctl show ooklaserver -p LimitNOFILE
```

---

<a id="12"></a>
## 12) Validação e publicação do Ookla (passo a passo) (obrigatório)

### 12.1 Sequência recomendada (instalar → testar → registrar → submeter)

1) Confirme que o servidor atende aos requisitos mínimos.
2) Instale o OoklaServer.
3) Valide tudo no **Server Tester** (host-tester).
4) Registre uma conta de Speedtest Servers.
5) Submeta o formulário do servidor e aguarde revisão (até ~48h).

### 12.2 Server Tester (obrigatório)

Use a ferramenta oficial de teste:
```text
https://www.ookla.com/host-tester
```

> Se aparecer erro “Connection Refused”, confirme:
> - processo OoklaServer rodando;
> - TCP/UDP 8080 acessível;
> - `OoklaServer.properties` usando as portas corretas.

### 12.3 Cadastro e submissão (obrigatório)

Crie conta:
```text
https://account.ookla.com/register/servers
```

Submeta:
```text
https://account.ookla.com/servers/create
```

Observação importante: a automação de certificado do Ookla pode começar apenas após submissão/aprovação, e o teste de HTTPS no host-tester pode ser ignorado durante o período inicial de submissão (quando aplicável).

---

<a id="13"></a>
## 13) Fail2Ban + nftables sets (obrigatório)

Objetivo:
- banir IPs em **sets** do nftables;
- evitar banir CGNAT/BRAS/infra crítica;
- começar com thresholds conservadores;
- ter escalonamento (10 → 60 minutos).

### 13.1 Aviso crítico (CGNAT/BRAS)

Se o tráfego chega com o **IP do BRAS/CGNAT** como origem, banir esse IP derruba milhares de clientes.  
Por isso:
- coloque BRAS/CGNAT/core em `ignoreip`;
- use `maxretry` alto no início;
- use bans curtos (10–60 min) e observe.

### 13.2 Action do Fail2Ban: inserir IP em set do nftables (obrigatório)

Arquivo: `/etc/fail2ban/action.d/nftables-set.conf`

```bash
sudo nano /etc/fail2ban/action.d/nftables-set.conf
```

Conteúdo:
```ini
[Definition]
# Parâmetros esperados:
# - family: inet
# - table: filter
# - setname: f2b_ookla_v4 / f2b_ookla_v6 / ...

actionstart =
actionstop =

# Valida se o set existe (evita “achar que está banindo”, mas não estar).
actioncheck = nft list set <family> <table> <setname> >/dev/null 2>&1

# Observação importante:
# - Se o IP já estiver no set, o `add element` pode falhar com “File exists”.
# - Se o IP não estiver no set, o `delete element` pode falhar.
# Em produção, isso não deve derrubar a jail, então usamos `|| true`.
actionban = nft add element <family> <table> <setname> { <ip> timeout <bantime> } || true
actionunban = nft delete element <family> <table> <setname> { <ip> } || true
```

### 13.3 Filtros

#### 13.3.1 Ookla (ban por flood/scanner detectado no nftables log)

O ruleset do nftables já gera logs com prefixo `F2B_OOKLA_TCP` e `F2B_OOKLA_UDP` quando a taxa excede o limite.

Arquivo: `/etc/fail2ban/filter.d/ookla-nftables-flood.conf`

```bash
sudo nano /etc/fail2ban/filter.d/ookla-nftables-flood.conf
```

Conteúdo:
```ini
[Definition]
failregex = ^.*F2B_OOKLA_TCP .* SRC=<HOST> .*$
            ^.*F2B_OOKLA_UDP .* SRC=<HOST> .*$
ignoreregex =
```

#### 13.3.2 LibreSpeed (ban por varredura/abuso no Nginx)

Arquivo: `/etc/fail2ban/filter.d/nginx-librespeed-badbots.conf`

```bash
sudo nano /etc/fail2ban/filter.d/nginx-librespeed-badbots.conf
```

Conteúdo (exemplos de patterns típicos de scanner):
```ini
[Definition]
failregex = ^<HOST> - .*\"(GET|POST) /(wp-login\\.php|xmlrpc\\.php|\\.env|cgi-bin/).* HTTP/.*\" (400|401|403|404) .*
            ^<HOST> - .*\"(GET|POST) /\\.git/.* HTTP/.*\" (400|401|403|404) .*
ignoreregex =
```

### 13.4 Jails (com ignoreip e bantime 10–60 min)

Arquivo: `/etc/fail2ban/jail.d/speedtest-isp.conf`

```bash
sudo nano /etc/fail2ban/jail.d/speedtest-isp.conf
```

Conteúdo (ajuste redes/IPs):
```ini
[DEFAULT]
backend = auto
usedns = no
findtime = 10m
bantime  = 10m

# Escalonamento (recidive): ban maior quando o IP reincide

ignoreip = 127.0.0.1/8 ::1 192.0.2.0/24 198.51.100.10/32 198.51.100.11/32

[ookla-flood]
enabled = true
filter  = ookla-nftables-flood
maxretry = 3

# Opção A (recomendado): pegar do journald (kernel logs)
backend = systemd
journalmatch = _TRANSPORT=kernel

action = nftables-set[family=inet, table=filter, setname=f2b_ookla_v4]
         nftables-set[family=inet, table=filter, setname=f2b_ookla_v6]

# Opção B (se você usa rsyslog gravando kernel em arquivo):
# logpath = /var/log/kern.log

[librespeed-badbots]
enabled = true
filter  = nginx-librespeed-badbots
logpath = /var/log/nginx/librespeed.access.log
maxretry = 5

action = nftables-set[family=inet, table=filter, setname=f2b_librespeed_v4]
         nftables-set[family=inet, table=filter, setname=f2b_librespeed_v6]

[recidive]
enabled = true
logpath = /var/log/fail2ban.log
findtime = 1d
bantime  = 60m
maxretry = 5
```

> Observação: `logpath=/var/log/kern.log` depende de rsyslog estar gravando kernel logs. Se sua distro estiver usando apenas journald, use backend systemd + journalmatch, ou habilite rsyslog.

### 13.5 Subir e validar Fail2Ban

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status ookla-flood
sudo fail2ban-client status librespeed-badbots
```

Ver sets:
```bash
sudo nft list set inet filter f2b_ookla_v4
sudo nft list set inet filter f2b_librespeed_v4
```

#### 13.5.1 Teste controlado (sem depender de ataque real)

1) Force um ban manual (use um IP “de laboratório”, não um cliente real):
```bash
sudo fail2ban-client set librespeed-badbots banip 203.0.113.123
```

2) Confira se entrou no set:
```bash
sudo nft list set inet filter f2b_librespeed_v4
```

3) Faça o unban manual:
```bash
sudo fail2ban-client set librespeed-badbots unbanip 203.0.113.123
```

4) Confira novamente:
```bash
sudo nft list set inet filter f2b_librespeed_v4
```

---

<a id="14"></a>
## 14) Logs, visibilidade e operação (obrigatório)

### 14.1 Logs do Nginx (LibreSpeed)

```bash
sudo tail -f /var/log/nginx/librespeed.access.log
sudo tail -f /var/log/nginx/librespeed.error.log
```

Logs da landing page do `OOKLA_FQDN` (se você ativou a seção 7.4):
```bash
sudo tail -f /var/log/nginx/speedtest-public.access.log
sudo tail -f /var/log/nginx/speedtest-public.error.log
```

### 14.2 Logs do OoklaServer

Se o Ookla estiver logando no journal:
```bash
sudo journalctl -u ooklaserver -f
```

Se o Ookla estiver logando em arquivo (FileChannel), valide o caminho configurado no `OoklaServer.properties` e monitore:
```bash
sudo tail -f /caminho/do/ooklaserver.log
```

### 14.3 Comandos essenciais (status)

```bash
sudo systemctl status nftables --no-pager
sudo systemctl status nginx --no-pager
sudo systemctl status fail2ban --no-pager
sudo ss -tulpn
```

### 14.4 Testar jails sem impactar produção

```bash
sudo fail2ban-regex /var/log/nginx/librespeed.access.log /etc/fail2ban/filter.d/nginx-librespeed-badbots.conf
```

> Em ambiente sem tráfego/sem ataques, `0 matched` é normal.

### 14.5 Logrotate (obrigatório em produção)

Objetivo: evitar que logs cresçam indefinidamente e matem disco/IO.

#### 14.5.1 Nginx (LibreSpeed + landing page)

No Debian, normalmente já existe:
```bash
sudo ls -la /etc/logrotate.d/nginx
sudo sed -n '1,200p' /etc/logrotate.d/nginx
```

Se você quiser **retenção e política próprias** só para os logs do Speedtest, crie um arquivo dedicado:

Arquivo: `/etc/logrotate.d/nginx-speedtest`
```bash
sudo nano /etc/logrotate.d/nginx-speedtest
```

Conteúdo (exemplo conservador):
```conf
/var/log/nginx/librespeed.*.log /var/log/nginx/speedtest-public.*.log {
  daily
  rotate 14
  missingok
  notifempty
  compress
  delaycompress
  sharedscripts
  postrotate
    # Reabre os arquivos de log do Nginx (sem derrubar o serviço)
    systemctl kill -s USR1 nginx 2>/dev/null || true
  endscript
}
```

> Se sua distro já rotaciona `*.log` do Nginx globalmente, não duplique regras (pode rotacionar 2x). Neste caso, ajuste apenas o `/etc/logrotate.d/nginx`.

#### 14.5.2 OoklaServer (quando usar FileChannel)

Recomendação prática:
- se possível, prefira logar no **journald** (ConsoleChannel) e controlar retenção do journal;
- se você usar `FileChannel`, use as opções de rotação/purge do próprio OoklaServer (quando disponíveis).

Opcional (fallback): rotação com `logrotate` usando `copytruncate` (não é perfeito, mas funciona sem sinalizar o processo).

Arquivo: `/etc/logrotate.d/ooklaserver`
```bash
sudo nano /etc/logrotate.d/ooklaserver
```

Conteúdo (ajuste o caminho do log):
```conf
/home/ookla/ooklaserver/ooklaserver.log {
  daily
  rotate 14
  missingok
  notifempty
  compress
  delaycompress
  copytruncate
}
```

#### 14.5.3 Journald (retenção)

Em produção, defina limites para o journal para não lotar disco:
```bash
sudo nano /etc/systemd/journald.conf
```

Exemplo (ajuste conforme seu disco):
```ini
[Journal]
SystemMaxUse=1G
SystemMaxFileSize=100M
MaxRetentionSec=14day
```

Aplique:
```bash
sudo systemctl restart systemd-journald
sudo journalctl --disk-usage
```

### 14.6 (Operação) Comandos úteis de nftables (backup/restore/sets)

Backup do ruleset atual:
```bash
sudo nft list ruleset > "/root/nftables.ruleset.$(date +%F_%H%M%S).nft"
```

Inspecionar sets do Fail2Ban:
```bash
sudo nft list set inet filter f2b_ookla_v4
sudo nft list set inet filter f2b_librespeed_v4
```

Remover manualmente um IP do set (desbanir “na marra”):
```bash
sudo nft delete element inet filter f2b_librespeed_v4 { 203.0.113.123 } || true
```

Esvaziar um set inteiro (use com MUITA cautela):
```bash
sudo nft flush set inet filter f2b_librespeed_v4
```

Restore de um backup (use com MUITA cautela):
```bash
sudo nft -f /root/nftables.ruleset.SEUBACKUP.nft
```

### 14.7 (Opcional) Scripts de operação (monitoramento, backup e diagnóstico)

Esta seção transforma o tutorial em um “runbook” simples para o NOC.

> Dependências:
> - scripts usam apenas utilitários padrão (`bash`, `systemctl`, `ss`, `df`, `tar`, `journalctl`, `openssl`);
> - se você quiser enviar e-mail automático, instale `mailutils` ou integre com seu sistema de alertas (Zabbix/Prometheus/Telegram/webhook).

#### 14.7.1 Script: verificação de saúde (healthcheck)

Arquivo: `/usr/local/bin/speedtest-healthcheck.sh`
```bash
sudo nano /usr/local/bin/speedtest-healthcheck.sh
sudo chmod +x /usr/local/bin/speedtest-healthcheck.sh
```

Conteúdo (ajuste os FQDNs/porta):
```bash
#!/usr/bin/env bash
set -euo pipefail

OOKLA_FQDN="${OOKLA_FQDN:-speedtest-sp01.seudominio.com.br}"
LIBRE_FQDN="${LIBRE_FQDN:-speedtest-interno.seudominio.com.br}"
LIBRE_PORT="${LIBRE_PORT:-8443}"

log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }

check_service() {
  local svc="$1"
  if systemctl is-active --quiet "$svc"; then
    log "OK service=$svc"
  else
    log "ERRO service=$svc (inactive/failed)"
  fi
}

check_listen() {
  local port="$1"
  if ss -lntp 2>/dev/null | grep -qE ":${port}\\b"; then
    log "OK listen=:$port"
  else
    log "ERRO listen=:$port (não está em escuta)"
  fi
}

check_disk_root() {
  local usage
  usage="$(df -P / | awk 'NR==2 {gsub(/%/,"",$5); print $5}')"
  if [ "${usage}" -ge 90 ]; then
    log "ERRO disco=/ uso=${usage}% (acima de 90%)"
  elif [ "${usage}" -ge 80 ]; then
    log "WARN disco=/ uso=${usage}% (acima de 80%)"
  else
    log "OK disco=/ uso=${usage}%"
  fi
}

check_cert_days() {
  local host="$1" port="$2"
  if ! command -v openssl >/dev/null 2>&1; then
    log "WARN openssl não encontrado (pulando check de TLS)"
    return 0
  fi
  local enddate epoch_now epoch_end days_left
  enddate="$(openssl s_client -connect "${host}:${port}" -servername "$host" </dev/null 2>/dev/null | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2 || true)"
  if [ -z "${enddate}" ]; then
    log "WARN tls=${host}:${port} (não consegui ler enddate)"
    return 0
  fi
  epoch_now="$(date +%s)"
  epoch_end="$(date -d "${enddate}" +%s)"
  days_left="$(( (epoch_end - epoch_now) / 86400 ))"
  if [ "${days_left}" -lt 7 ]; then
    log "ERRO tls=${host}:${port} expira_em=${days_left}d"
  elif [ "${days_left}" -lt 14 ]; then
    log "WARN tls=${host}:${port} expira_em=${days_left}d"
  else
    log "OK tls=${host}:${port} expira_em=${days_left}d"
  fi
}

log "INICIO healthcheck"
check_service nftables
check_service nginx
check_service fail2ban
check_service ooklaserver

check_listen 80
check_listen 443
check_listen 8080
check_listen 5060
check_listen "${LIBRE_PORT}"

check_disk_root
check_cert_days "${OOKLA_FQDN}" 443
log "FIM healthcheck"
```

Execute manualmente:
```bash
sudo /usr/local/bin/speedtest-healthcheck.sh
```

Agende via cron (a cada 15 minutos):
```bash
sudo crontab -e
```

Exemplo de entrada:
```cron
*/15 * * * * /usr/local/bin/speedtest-healthcheck.sh >> /var/log/speedtest-healthcheck.log 2>&1
```

#### 14.7.2 Script: backup das configurações

Arquivo: `/usr/local/bin/speedtest-backup-config.sh`
```bash
sudo nano /usr/local/bin/speedtest-backup-config.sh
sudo chmod +x /usr/local/bin/speedtest-backup-config.sh
```

Conteúdo (ajuste o diretório de backup):
```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="${BACKUP_DIR:-/root/backups/speedtest}"
TS="$(date +%F_%H%M%S)"
OUT="${BACKUP_DIR}/speedtest-config-${TS}.tar.gz"

mkdir -p "${BACKUP_DIR}"

tar -czf "${OUT}" \
  /etc/nftables.conf \
  /etc/sysctl.d/99-speedtest-isp.conf \
  /etc/fail2ban/jail.d/ \
  /etc/fail2ban/filter.d/ \
  /etc/fail2ban/action.d/nftables-set.conf \
  /etc/nginx/sites-available/speedtest-public \
  /etc/nginx/sites-available/librespeed-interno \
  /etc/systemd/system/ooklaserver.service \
  2>/dev/null || true

echo "Backup criado: ${OUT}"
ls -lah "${BACKUP_DIR}" | tail -n 5
```

Agende semanalmente (domingo 02:00):
```bash
sudo crontab -e
```

```cron
0 2 * * 0 /usr/local/bin/speedtest-backup-config.sh >> /var/log/speedtest-backup.log 2>&1
```

#### 14.7.3 Script: diagnóstico (para enviar ao NOC)

Arquivo: `/usr/local/bin/speedtest-diag.sh`
```bash
sudo nano /usr/local/bin/speedtest-diag.sh
sudo chmod +x /usr/local/bin/speedtest-diag.sh
```

Conteúdo (gera um relatório em `/tmp`):
```bash
#!/usr/bin/env bash
set -euo pipefail

REPORT="/tmp/speedtest-diag-$(date +%F_%H%M%S).txt"

{
  echo "==== SPEEDTEST DIAG ($(date)) ===="
  echo
  echo "-- OS --"
  uname -a
  cat /etc/os-release | grep -E '^(PRETTY_NAME|VERSION)=|^ID=' || true
  echo
  echo "-- SERVICOS --"
  systemctl status nftables nginx fail2ban ooklaserver --no-pager -l || true
  echo
  echo "-- PORTAS --"
  ss -tulpn || true
  echo
  echo "-- NFTABLES --"
  nft list ruleset || true
  echo
  echo "-- FAIL2BAN --"
  fail2ban-client status || true
  echo
  echo "-- LOGS (recentes) --"
  journalctl -u ooklaserver -n 50 --no-pager || true
  tail -n 50 /var/log/nginx/librespeed.error.log 2>/dev/null || true
  tail -n 50 /var/log/fail2ban.log 2>/dev/null || true
  echo
  echo "-- TUNING --"
  sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc || true
  sysctl net.core.somaxconn net.core.netdev_max_backlog || true
  sysctl net.core.rmem_max net.core.wmem_max || true
  echo
  echo "-- DISCO --"
  df -h / || true
  echo
} > "${REPORT}"

echo "Relatório gerado em: ${REPORT}"
```

---

<a id="15"></a>
## 15) Checklist final (produção)

- DNS OK: A/AAAA resolvendo para o hostname público do Ookla
- Portas exigidas abertas publicamente: TCP/UDP 8080 e 5060; TCP 80 (ACME/legacy); TCP 443 (landing page do OOKLA_FQDN)
- nftables ativo (DROP por padrão) e LibreSpeed acessível apenas por redes do ISP (porta 8443)
- LibreSpeed com HTTPS válido (Certbot DNS-01) e bloqueio externo testado
- Ookla em execução e validado no Server Tester
- Conta criada e servidor submetido (aguardando aprovação quando aplicável)
- Fail2Ban ativo e inserindo IPs em sets (`nft list set ...`)

---

<a id="16"></a>
## 16) Troubleshooting (problemas comuns)

### 16.1 Ookla “Connection refused” no Server Tester (host-tester)

Checklist rápido:
```bash
sudo systemctl status ooklaserver --no-pager
sudo ss -tulpn | grep -E ':(8080|5060)\\b'
sudo nft list chain inet filter input | grep -E 'dport (8080|5060)' || true
```

De fora (outra máquina):
```bash
curl -v "http://$OOKLA_FQDN:8080/" 2>&1 | head -n 30
```

### 16.2 `OOKLA_FQDN` não resolve (DNS)

```bash
dig +short "$OOKLA_FQDN" A
dig +short "$OOKLA_FQDN" AAAA
```

Se não resolver:
- verifique se o registro A/AAAA existe na zona correta;
- verifique se o TTL já propagou;
- teste com um resolvedor público: `dig +short "$OOKLA_FQDN" A @8.8.8.8`.

### 16.3 LibreSpeed ficou público (CRÍTICO)

Objetivo: o LibreSpeed deve estar **apenas** na porta interna (ex.: 8443) e **apenas** para redes do ISP.

1) Confirme a regra do firewall (porta interna somente para `@isp_v4/@isp_v6`):
```bash
sudo nft list chain inet filter input | grep -E "dport ${LIBRE_PORT}|dport 8443" || true
```

2) Confirme a ACL do Nginx (allow/deny) no vhost do LibreSpeed:
```bash
sudo nginx -T 2>/dev/null | grep -nE "server_name ${LIBRE_FQDN}|allow |deny all" || true
```

3) Teste de fora (tem que falhar):
```bash
timeout 5 curl -I "https://$LIBRE_FQDN:$LIBRE_PORT" && echo "ERRO: público" || echo "OK: bloqueado"
```

### 16.4 Fail2Ban não está banindo

1) Status:
```bash
sudo fail2ban-client status
sudo fail2ban-client status librespeed-badbots
```

2) Teste do filtro (sem banir):
```bash
sudo fail2ban-regex /var/log/nginx/librespeed.access.log /etc/fail2ban/filter.d/nginx-librespeed-badbots.conf
```

3) Teste controlado (ban manual + set):
```bash
sudo fail2ban-client set librespeed-badbots banip 203.0.113.123
sudo nft list set inet filter f2b_librespeed_v4
sudo fail2ban-client set librespeed-badbots unbanip 203.0.113.123
```

### 16.5 Certbot não renova

```bash
sudo certbot renew --dry-run
systemctl status certbot.timer --no-pager || true
```

Se falhar:
- para `OOKLA_FQDN` (HTTP-01), confirme a porta 80 acessível publicamente;
- para `LIBRE_FQDN` (DNS-01), confirme o plugin/credenciais de DNS ou repita a validação manual TXT.

### 16.6 IPv6 não funciona

Checklist:
```bash
ip -6 addr
ping6 -c 3 2001:4860:4860::8888 || true
dig +short "$OOKLA_FQDN" AAAA
sudo ss -tulpn | grep -E ':8080\\b|:5060\\b' | grep -E ':::|\\[::\\]' || true
```

### 16.7 Logs lotando disco

```bash
sudo du -sh /var/log/* 2>/dev/null | sort -h | tail -n 15
sudo journalctl --disk-usage
```

Revise `logrotate` (seção 14.5) e limites do journald.

---

<a id="17"></a>
## 17) Comandos rápidos (operação diária)

Status geral:
```bash
systemctl status nftables nginx fail2ban ooklaserver --no-pager
```

Ver portas importantes:
```bash
ss -tulpn | grep -E ':(80|443|8443|8080|5060)\\b' || true
```

Logs:
```bash
journalctl -u ooklaserver -f
tail -f /var/log/nginx/librespeed.access.log
tail -f /var/log/fail2ban.log
journalctl -k -f | grep -E 'F2B_OOKLA_' || true
```

Fail2Ban:
```bash
fail2ban-client status
fail2ban-client status librespeed-badbots
nft list set inet filter f2b_ookla_v4
nft list set inet filter f2b_librespeed_v4
```

---

<a id="18"></a>
## 18) Referências

```text
Ookla (requisitos e portas):
https://zdtm.my.site.com/ooklasupport/s/article/Speedtest-Server-Requirements

Ookla (instalar e submeter):
https://zdtm.my.site.com/ooklasupport/s/article/how-to-install-submit-server

Ookla (configuração avançada do OoklaServer.properties):
https://zdtm.my.site.com/ooklasupport/s/article/OoklaServer-Daemon-Advanced-Configuration

Doc oficial (pode exigir acesso/challenge):
https://support.ookla.com/hc/en-us/articles/234578528-OoklaServer-Installation-Linux-Unix

Server Tester:
https://www.ookla.com/host-tester

LibreSpeed:
https://github.com/librespeed/speedtest/

Certbot (manual DNS-01 / hooks / timers systemd):
https://eff-certbot.readthedocs.io/en/stable/using.html

Certbot DNS Cloudflare (exemplo de automação DNS-01):
https://eff-certbot.readthedocs.io/en/stable/using.html#dns-cloudflare

Nginx (reabrir logs com USR1):
https://nginx.org/en/docs/control.html

nftables (documentação oficial):
https://wiki.nftables.org/wiki-nftables/index.php/Main_Page
```

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
