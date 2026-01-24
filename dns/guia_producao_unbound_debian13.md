# Guia de Produção: Unbound no Debian 13 (Trixie), DNS Recursivo para ISPs com DNSSEC e RPZ (bloqueio por segurança)

*Criado em: 23 de janeiro de 2026*  

Este guia detalha o processo completo para configurar um servidor DNS recursivo de alta performance usando **Unbound** em um ambiente de **Provedor de Internet (ISP)** no **Debian 13 (Trixie)**. A configuração é otimizada para segurança e desempenho, incluindo **DNSSEC**, **QNAME Minimization (Minimização de QNAME)** e **RPZ (Response Policy Zones)** para bloqueio de domínios associados a ameaças (phishing/malware/C2), com atualização automatizada.

Ele combina as seguintes características e boas práticas:

-   **Segurança de Protocolo**: Validação **DNSSEC** ponta a ponta e minimização de consultas (**QNAME Minimization (Minimização de QNAME)**).
-   **Segurança Ativa**: Firewall `nftables` restritivo e proteção contra abuso de clientes com **Fail2Ban**.
-   **Proteção de Rede (RPZ de segurança)**: Bloqueio de domínios de malware, phishing e botnets/C2 via **RPZ (Response Policy Zones)**, com atualização automática e robusta.
-   **Alta Performance para ISP**: Otimização de cache, threads e buffers de rede para alto volume de tráfego.
-   **Automação e Operação**: Scripts e `systemd timers` para manter as listas de bloqueio atualizadas e comandos de gestão para o dia a dia.

> **Objetivo:** Ao final deste tutorial, você terá um servidor DNS recursivo robusto, que não aceita consultas públicas, protege seus clientes contra ameaças conhecidas e se defende de abusos, tudo de forma automatizada e escalável. A infraestrutura será capaz de atuar desde a raiz da internet, com validação DNSSEC e preparado para ambiente de produção de ISP.

> **Nota:** este documento é independente, sem afiliação com NIC.br/CERT.br/CGI.br; links citados são apenas referências públicas.  
> **Nota (jurídico/regulatório):** este material não é aconselhamento jurídico; valide políticas e comunicações com jurídico/regulatório do seu contexto.

---

## Índice
0. [Escopo](#0)
1. [Pré-requisitos](#1)
2. [Instalação de pacotes](#2)
3. [Sincronização de horário](#3)
4. [Estrutura de diretórios e permissões](#4)
5. [Configuração do Unbound](#5)
6. [DNSSEC e root hints](#6)
7. [RPZ](#7)
8. [Automatização](#8)
9. [Ajustes de kernel](#9)
10. [Firewall nftables](#10)
11. [Fail2Ban](#11)
12. [Subir serviços e validação](#12)
13. [Rollback](#13)
14. [Checklist pós-implantação](#14)
15. [Referências](#15)

<a id="0"></a>
## 0. Escopo (o que este tutorial faz, e o que NÃO faz)

### 0.1 O que faz

1. Configura o Unbound como **resolver recursivo** (sem zonas autoritativas).
2. Restringe consultas a **redes do provedor** (ACL do Unbound + firewall nftables).
3. Ativa **DNSSEC validation** e **QNAME Minimization (Minimização de QNAME)**.
4. Implementa uma **política de RPZ focada em ameaças**, somente:
   - phishing
   - malware
   - botnets / C2
5. Faz o bloqueio **exclusivamente via DNS**, usando **Response Policy Zones (RPZ)** do Unbound.
6. Automatiza atualização de listas com **scripts + systemd timers**.
7. Integra **Fail2Ban** para **banir IPs abusivos** via **nftables set**.

### 0.2 O que NÃO faz (propositalmente)

- NÃO configura DNS autoritativo.
- NÃO cria zonas forward nem reversas.
- NÃO faz DPI, proxy, firewall L7, nem bloqueio por categoria (pornografia/apostas/drogas etc).
- NÃO configura RPZ “de censura” (apenas feeds de segurança focados em ameaças).
- NÃO fornece feed/lista de domínios, apenas mostra como **consumir feeds com licença compatível** e gerar zonefiles RPZ.

### 0.3 Por que isso NÃO fere a neutralidade da rede e qual o impacto pro cliente

- **Não há inspeção de conteúdo (DPI)** nem bloqueio por “categoria”. O DNS só decide **se um nome resolve**.
- O bloqueio é baseado em **ameaças objetivas de segurança** (phishing/malware/C2) publicadas em **feeds de segurança**, e não em preferências comerciais/políticas.
- O efeito para o usuário é, em geral, **NXDOMAIN** (como “domínio não existe”). Para falsos positivos, existe **allowlist** e rollback rápido.
- A implementação é **transparente e reversível**: você consegue desativar a política sem parar o DNS (seção 13).

> **Importante (compliance):** este tutorial é técnico e **não substitui parecer jurídico**.  
> Em ISP, a forma “mais defensável” é tratar o bloqueio via RPZ como **medida de segurança** (anti‑fraude/anti‑malware), com:
> - critérios públicos de inclusão/remoção (somente ameaças);
> - registro/auditoria mínima (logs e versão da lista);
> - allowlist/“contestação” interna;
> - rollback rápido.

---

<a id="1"></a>
## 1. Pré-requisitos
### 1.1 Pacotes usados

- Unbound: `unbound`, `unbound-anchor`
- Ferramentas: `dnsutils`, `curl`, `ca-certificates`
- Firewall: `nftables`
- Fail2Ban: `fail2ban`

### 1.2 Redes e IPs

> **Use seus prefixos reais.** Abaixo são **exemplos** (RFC 5737 / RFC 3849 + CGNAT).

- **Rede de clientes (exemplo recomendado):**
  - IPv4 CGNAT dos assinantes: `100.64.0.0/10`
  - IPv6 dos assinantes: `2001:db8:100::/48`
- **Infraestrutura crítica (NÃO pode ser banida):**
  - BRAS/BNG (exemplo): `198.51.100.10/32`, `198.51.100.11/32`
  - Roteadores core (exemplo): `198.51.100.1/32`
- **Rede de gerência (exemplo):**
  - IPv4: `192.0.2.0/24`
  - IPv6: `2001:db8:200::/48`

> **Importante:**  
> O ponto crítico aqui é **qual IP de origem o Unbound enxerga nas consultas DNS** (ou seja: qual `src IP` chega no servidor recursor).
>
> **Cenário 1 — Unbound enxerga o IP real do cliente (ban por IP é seguro):**  
> - Exemplo IPv4: sua rede de assinantes (`100.64.0.0/10`) está **roteada até o recursor** (sem NAT no caminho), então cada cliente chega com um IP próprio.  
> - Exemplo IPv6: o cliente chega com seu prefixo (ex.: `/64`) e o recursor vê o IPv6 real.
>
> Nesse cenário, banir por IP via Fail2Ban faz sentido: você bloqueia **um cliente** sem derrubar outros.
>
> **Cenário 2 — Unbound enxerga IP “mascarado” por BRAS/CGNAT (ban por IP é perigoso):**  
> - O BRAS/BNG/CGNAT faz NAT/mascara a origem, então o recursor vê apenas o IP do concentrador, ou um pool pequeno compartilhado.  
> - Resultado: **muitos assinantes compartilham o mesmo IP de origem** do ponto de vista do Unbound.
>
> **ATENÇÃO:** Nesse cenário, banir por IP via Fail2Ban pode derrubar **vários clientes de uma vez**. Recomendações:
> - inclua IPs de BRAS/BNG/CGNAT no `ignoreip` do Fail2Ban, para não banir infraestrutura;
> - use thresholds mais conservadores (`maxretry/findtime/bantime`) e calibre em produção;
> - trate abuso no plano correto (fora do DNS): PPPoE/DHCP/RADIUS/BRAS/CGNAT (limites, quarentena, identificação do assinante).

### 1.3 Arquitetura recomendada para ISP, alta disponibilidade, sem “gambiarra”

1) **Dois ou mais resolvers recursivos** (ex.: `dns-rec1` e `dns-rec2`)  
   - idealmente em **hosts diferentes**;
   - clientes recebem **dois IPs** de DNS (DHCP/PPPoE/RA/DHCPv6).

2) **Sem forwarders externos**  
   - o Unbound resolve **desde a raiz** usando **root hints** mais controle e menos dependência.

1) **Fechado por design**  
   - camada 1: **nftables** (default drop / política `DROP`, libera 53 só para redes do ISP);
   - camada 2: **ACL do Unbound** (`access-control ... allow/refuse`).

4) **Resiliência e capacidade de auditoria**  
   - RPZ com update atômico + rollback automático (*update atômico*: a lista é baixada/gerada em um arquivo temporário e só substitui a versão ativa após validação, evitando estados inconsistentes);
   - logs mínimos (sem “logar tudo”) e versionamento local das listas.


### 1.4 Dimensionamento (CPU/RAM/Disco) e impacto real em ISP

O Unbound escala muito bem, mas **RAM (cache)** e **CPU (threads)** são os recursos mais críticos.

> **Aviso:** dimensionamento deve ser baseado em **QPS real** (Queries Per Second) observado na sua rede.  
> “Número de clientes” é apenas um **indicador indireto**: dois provedores com a mesma base podem ter QPS totalmente diferente.

Fatores que costumam **aumentar QPS** (mesmo sem aumentar a base):
- **Comportamento de uso** (streaming, games, chamadas, navegação intensa) e horários de pico.
- **CPE/roteadores** que fazem cache ruim, retries agressivos ou não respeitam TTL.
- **IPv6** (dual-stack aumenta volume de perguntas A+AAAA e tráfego de validação/recursão).
- **Aplicações** e dispositivos “falantes” (apps mobile, smart TVs, IoT, antivírus/EDR, navegadores com prefetch/DoH fallback).
- **Desenho de rede** (POP distante, latência, perda) que aumenta retries e “tempestade” de reconsultas.

Recomendação bem pragmática (ponto de partida, conservadora e focada em estabilidade):

- **Pequeno (baixo QPS / 1 POP até ~10k clientes / perfil simples)**: 2–4 vCPU, 4–8 GB RAM, SSD.  
  Tipicamente: QPS baixo a moderado, pouca variabilidade, poucas “tempestades” de consulta.
- **Médio (QPS moderado/alto / 1+ POPs ~10k–50k clientes / pico mais agressivo)**: 4–8 vCPU, 16–32 GB RAM, SSD/NVMe.  
  Tipicamente: picos claros (noite/fins de semana), dual-stack, mais retransmissões, mais tráfego de validação.
- **Grande (QPS alto / múltiplos POPs 50k+ clientes / operação mais complexa)**: 8+ vCPU, 32+ GB RAM, NVMe.  
  Tipicamente: grande volume sustentado, picos fortes, alta concorrência, necessidade de cache maior e margem operacional.

Boas práticas de sizing:
- comece conservador em `msg-cache-size` e `rrset-cache-size` e aumente com observabilidade;
- em geral `rrset-cache-size` fica **maior** que `msg-cache-size`;
- ajuste `num-threads` para CPUs reais (evite exagerar: mais threads ≠ mais performance sempre);
- dimensione disco pensando em **logs + backups de listas** (não em cache).

### 1.5 Sincronização de tempo, OBRIGATÓRIO por causa de DNSSEC

Hora correta não é “detalhe”: é pré-requisito para **DNSSEC**, para **coerência de logs** (auditoria/troubleshooting) e para qualquer automação de banimento (ex.: **Fail2Ban**).

DNSSEC valida assinaturas com janelas de tempo. Se o relógio estiver errado:
- validações podem falhar (SERVFAIL em domínios DNSSEC),
- e você “parece” estar com DNS quebrado sem estar.

Este guia mostra o mínimo para subir NTP na seção **3**, mas **não** aprofunda NTP aqui.  
Para um passo a passo completo e pronto para produção, veja: **[Guia de Produção: Servidor NTP Interno com Chrony no Debian 13](../sistema/guia_producao_ntp_chrony_debian13.md)**.

### 1.6 Mapa rápido de arquivos

- Unbound:
  - config principal: `/etc/unbound/unbound.conf`
  - includes: `/etc/unbound/unbound.conf.d/*.conf`
  - arquivos deste guia (unbound.conf.d):
    - `/etc/unbound/unbound.conf.d/00-server.conf`
    - `/etc/unbound/unbound.conf.d/10-acl.conf`
    - `/etc/unbound/unbound.conf.d/20-dnssec-root-hints.conf`
    - `/etc/unbound/unbound.conf.d/30-privacy-hardening.conf`
    - `/etc/unbound/unbound.conf.d/40-performance-cache.conf`
    - `/etc/unbound/unbound.conf.d/50-logging.conf` (ou `50-logging-journald.conf`)
    - `/etc/unbound/unbound.conf.d/60-rpz-seguranca.conf` (se RPZ estiver habilitado)
    - `/etc/unbound/unbound.conf.d/90-remote-control.conf`
  - root hints: `/etc/unbound/root.hints`
  - trust anchor: `/var/lib/unbound/root.key`
- RPZ:
  - política (unbound.conf.d): `/etc/unbound/unbound.conf.d/60-rpz-seguranca.conf`
  - zonefiles: `/var/lib/unbound/rpz/*.zone`
  - allowlist: `/etc/unbound/rpz/rpz-allowlist.txt`
  - logs do updater: `/var/log/unbound/rpz-update.log`
- Firewall:
  - regras: `/etc/nftables.conf`
- Fail2Ban:
  - jails: `/etc/fail2ban/jail.d/unbound.local`
  - filters: `/etc/fail2ban/filter.d/unbound-*.conf`
  - action: `/etc/fail2ban/action.d/nftables-unbound-set.conf`

---

<a id="2"></a>
## 2. Instalação de pacotes (Debian 13)

### 2.1 Atualize o sistema e instale pacotes

```bash
sudo apt update
sudo apt -y full-upgrade
```

Instale os Pacotes principais
```bash
sudo apt -y install unbound unbound-anchor dnsutils nftables fail2ban curl ca-certificates
```

Ferramentas úteis em operação/testes
```bash
sudo apt -y install dnsperf jq
```

Valide a versão e recursos:

```bash
sudo unbound -V
```

---

<a id="3"></a>
## 3. Sincronização de horário (DNSSEC depende disso)

### Contexto / por quê isso é obrigatório

Hora correta não é “detalhe”: é pré-requisito para:
- **DNSSEC** (validação de assinaturas depende de janelas de tempo);
- **coerência de logs** (auditoria/troubleshooting);
- automações de mitigação (ex.: **Fail2Ban** e timers).

Se o relógio estiver errado, você pode ver **SERVFAIL** em domínios com DNSSEC e “parecer” que o DNS está quebrado.

### O que fazer agora

Opção A (simples, via systemd):

```bash
sudo timedatectl set-ntp true
timedatectl status
```

Procure por:
- `NTP service: active`
- `System clock synchronized: yes`

Opção B (mais comum em ISP, via Chrony):

```bash
sudo apt -y install chrony
sudo systemctl enable --now chrony
chronyc tracking
```

Para um passo a passo completo e pronto para produção (NTP interno com Chrony), veja:  
**[Guia de Produção: Servidor NTP Interno com Chrony no Debian 13](../sistema/guia_producao_ntp_chrony_debian13.md)**.

---

<a id="4"></a>
## 4. Estrutura de diretórios e permissões

Esta preparação existe porque este guia usa:
- RPZ (zonefiles e allowlist),
- logs em arquivo (`logfile:` do Unbound), e
- scripts de automação (atualização de RPZ + log próprio).

Se você for usar **somente journald** e **sem RPZ**, parte desta seção pode ser tratada como opcional.

O que cada pasta representa neste guia:

- `/etc/unbound/unbound.conf.d` (**recomendado**)  
  Onde ficam os fragmentos de configuração do Unbound neste guia (padrão Debian). O `/etc/unbound/unbound.conf` fica mínimo e apenas inclui este diretório.

- `/var/lib/unbound/rpz` (**obrigatório para RPZ**)  
  Onde ficam os zonefiles gerados/atualizados pelo script.

- `/var/log/unbound` (**necessário apenas se usar `logfile:`**)  
  Se você optar por logs somente via journald, esta pasta pode não ser necessária.

> Em produção, permissões “certas” aqui evitam **erros silenciosos** (ex.: script não consegue escrever zonefile, Unbound não consegue gravar log, reload falha sem visibilidade).  
> Isso não é tuning: é preparação estrutural para o ambiente operar de forma previsível.

Crie diretórios:

```bash
sudo install -d -m 0755 /etc/unbound/unbound.conf.d
sudo install -d -m 0750 /var/log/unbound
sudo install -d -m 0755 /var/lib/unbound/rpz
sudo install -d -m 0750 /etc/unbound/rpz
```

Permissões (usuário do serviço costuma ser `unbound` no Debian):

```bash
sudo chown -R unbound:unbound /var/log/unbound /var/lib/unbound
sudo chown -R root:unbound /etc/unbound
sudo chmod 0750 /etc/unbound/rpz
```

Se você vai usar **log em arquivo** (diretiva `logfile:` no `50-logging.conf`), crie o arquivo de log com permissões corretas:

```bash
sudo install -o unbound -g unbound -m 0640 /dev/null /var/log/unbound/unbound.log
sudo ls -l /var/log/unbound/unbound.log
```

> Se você não fizer isso, um erro típico é:
> `Could not open logfile /var/log/unbound/unbound.log: Permission denied`

Se as permissões estiverem corretas e o erro **persistir**, em Debian a causa mais comum é **AppArmor** bloqueando o Unbound de abrir/criar o logfile.  
Siga o passo **5.5.1** (AppArmor e `logfile:`) para diagnóstico e correção.

Se necessário, libere explicitamente escrita em `/var/log/unbound` com um override, não altera o conteúdo técnico do Unbound, só o sandbox do serviço:

```bash
sudo systemctl edit unbound
```

Cole:

```ini
[Service]
ReadWritePaths=/var/log/unbound
```

E aplique:

```bash
sudo systemctl daemon-reload
sudo systemctl restart unbound
sudo journalctl -u unbound -n 30 --no-pager
```

---

<a id="5"></a>
## 5. Configuração do Unbound

### 5.1 Verifique quem está escutando na porta 53 (DNS)

#### Contexto / por quê

O objetivo aqui é **confirmar quem está escutando na porta 53/UDP e 53/TCP**, para evitar conflito quando você for colocar o Unbound para atender DNS.

> Nesta etapa, o Unbound ainda está **restrito ao loopback** (`127.0.0.1` e `::1`) para você validar a configuração com segurança.  
> A abertura para as redes do ISP acontece mais adiante, depois do firewall e das ACLs (seção 12).

#### O que fazer agora

Antes de subir o Unbound, confirme quem está ouvindo na porta 53:

```bash
sudo ss -lntup | grep -E '(^Netid|:53)'
```

Se aparecer `systemd-resolved` ouvindo em `127.0.0.53:53` e você quer que **o Unbound** escute na porta 53 do host:

```bash
sudo systemctl disable --now systemd-resolved || true
```

> Nota: em instalações padrão do Debian, o `systemd-resolved` geralmente **não** vem habilitado ou nem instalado. Se ele não existir, isso **não é erro**.
>
> Observação: em alguns ambientes, `resolv.conf` é gerenciado por `systemd-resolved`/NetworkManager. A seção **12.6** mostra um caminho seguro para fazer o **próprio servidor** usar o Unbound local sem “quebrar” a rede.

### 5.2 Padrão Debian: `unbound.conf` mínimo + `unbound.conf.d/` arquivos numerados

O ideal é o arquivo `/etc/unbound/unbound.conf` ficar **mínimo** o arquivo padrão do pacote.  
Toda a configuração “real” do recursor deste guia fica em **`/etc/unbound/unbound.conf.d/*.conf`**.

Isso é importante em produção porque:
- evita “um arquivo monstro” difícil de manter;
- facilita rollback e auditoria. configurações por partes;
- segue o padrão do empacotamento Debian.

#### 5.2.1 Verifique o `include-toplevel`

Confirme se o arquivo do pacote tem a linha:

```bash
sudo grep -nE '^[[:space:]]*include-toplevel:[[:space:]]*"/etc/unbound/unbound\.conf\.d/\*\.conf"' /etc/unbound/unbound.conf
```

Se **não** aparecer, restaure o padrão do Debian (sem colar um `server:` gigante nele):

```bash
TS="$(date +%F_%H%M%S)"
sudo cp -av /etc/unbound/unbound.conf "/etc/unbound/unbound.conf.bak.${TS}" \
  || echo "INFO: /etc/unbound/unbound.conf não existe (primeira instalação) ou não foi possível copiar."

sudo apt -y install --reinstall unbound
```

> Se o `dpkg` perguntar sobre “arquivo de configuração foi modificado”, escolha a versão do **mantenedor** para voltar ao padrão do Debian.

#### 5.2.2 Backup rápido do diretório `unbound.conf.d` (recomendado)

```bash
TS="$(date +%F_%H%M%S)"
sudo cp -av /etc/unbound/unbound.conf.d "/etc/unbound/unbound.conf.d.bak.${TS}" \
  || echo "INFO: /etc/unbound/unbound.conf.d ainda não existe (primeira instalação) ou não foi possível copiar."
```

Padrão de backup usado neste guia (arquivo único):

```bash
TS="$(date +%F_%H%M%S)"
sudo cp -av /CAMINHO/ARQUIVO.conf "/CAMINHO/ARQUIVO.conf.bak.${TS}"
```

> Use este mesmo padrão sempre que o guia disser “Backup (recomendado)”, trocando apenas o caminho do arquivo.

#### 5.2.3 Crie os arquivos de configuração em `/etc/unbound/unbound.conf.d/`

Regras deste guia:
- os arquivos são ordenados e fáceis de auditar (`00-`, `10-`, `20-`, …);
- cada arquivo começa com o bloco correto (`server:`, `remote-control:` etc.);
- múltiplos `server:` são somados; quando um parâmetro aparece mais de uma vez, o último valor vence.

Importante: as redes `192.0.2.0/24` e `2001:db8::` abaixo são **EXEMPLO** (RFC 5737 / RFC 3849).  
Troque por **prefixos reais** do seu ISP.

1) Base do recursor (rede/porta).  
Crie `/etc/unbound/unbound.conf.d/00-server.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/00-server.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Identidade / segurança básica
    ########################################################################

    # Usuário que o processo do Unbound vai usar após iniciar.
    # Em Debian, normalmente é "unbound". Ajuda a reduzir impacto caso haja falha.
    username: "unbound"

    # Remove o hostname/identidade do servidor das respostas (CHAOS TXT).
    # Bom para reduzir fingerprinting.
    hide-identity: yes

    # Remove a versão do Unbound das respostas (CHAOS TXT).
    # Bom para reduzir fingerprinting e scans.
    hide-version: yes

    ########################################################################
    # Rede: interfaces, porta e protocolos
    ########################################################################

    # Durante a implantação inicial, mantenha o Unbound restrito ao loopback.
    # Depois de configurar o firewall e validar ACL, você vai abrir as interfaces (seção 12).
    interface: 127.0.0.1
    interface: ::1

    # Porta padrão do DNS.
    port: 53

    # Habilita IPv4/IPv6 e transporte UDP/TCP (TCP é necessário para respostas grandes/DNSSEC).
    do-ip4: yes
    do-ip6: yes
    do-udp: yes
    do-tcp: yes

    # EDNS: 1232 é um valor moderno para reduzir fragmentação (especialmente IPv6).
    edns-buffer-size: 1232
EOF
```

2) ACL (resolver fechado).  
Crie `/etc/unbound/unbound.conf.d/10-acl.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/10-acl.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Resolver FECHADO (ACL)
    #
    # ALERTA (EXEMPLO):
    # - 192.0.2.0/24 e 2001:db8:: são redes de DOCUMENTAÇÃO (EXEMPLO).
    # - Substitua por redes reais do seu ISP antes de abrir o Unbound para a rede.
    #
    # Como ler esta seção:
    # - `access-control: <rede> allow` libera clientes daquela rede/prefixo.
    # - `access-control: <rede> refuse` recusa o resto (resolver fechado).
    #
    # Importante:
    # - o firewall (nftables) também deve restringir 53/udp+tcp para as redes do ISP;
    # - esta ACL é a segunda camada de segurança (defesa em profundidade).
    ########################################################################

    ########################################################################
    # Loopback (o próprio servidor pode consultar o DNS local)
    ########################################################################
    access-control: 127.0.0.0/8 allow
    access-control: ::1 allow

    ########################################################################
    # EXEMPLO: rede de gerência (troque pela sua rede real)
    ########################################################################
    access-control: 192.0.2.0/24 allow
    access-control: 2001:db8:200::/48 allow

    ########################################################################
    # EXEMPLO: rede de assinantes (ver seção 1.2 – CGNAT e origem do IP)
    ########################################################################
    access-control: 100.64.0.0/10 allow
    access-control: 2001:db8:100::/48 allow

    ########################################################################
    # Bloqueia todo o resto (resolver fechado)
    ########################################################################
    access-control: 0.0.0.0/0 refuse
    access-control: ::/0 refuse
EOF
```

3) DNSSEC (trust anchor) e recursão desde a raiz (root hints).  
Crie `/etc/unbound/unbound.conf.d/20-dnssec-root-hints.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/20-dnssec-root-hints.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # DNSSEC validation (trust anchor)
    ########################################################################

    # Caminho do trust anchor (chave raiz) usado para validação DNSSEC.
    # O arquivo root.key será gerado na seção 6.1.
    auto-trust-anchor-file: "/var/lib/unbound/root.key"

    ########################################################################
    # Root hints (recursão desde a raiz, sem forwarders externos)
    ########################################################################

    # Arquivo com a lista de root servers (root hints).
    # O arquivo root.hints será baixado na seção 6.2.
    root-hints: "/etc/unbound/root.hints"
EOF
```

4) Privacidade/hardening e mitigação **no nível do Unbound**
Crie `/etc/unbound/unbound.conf.d/30-privacy-hardening.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/30-privacy-hardening.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Privacidade
    ########################################################################

    # Minimização de QNAME: envia para a internet apenas o necessário,
    # reduzindo vazamento de informações em consultas recursivas.
    qname-minimisation: yes

    # Modo strict pode quebrar domínios mal configurados.
    # Em ISP, costuma-se manter "no" para reduzir chamados.
    qname-minimisation-strict: no

    ########################################################################
    # Hardening (boas práticas)
    ########################################################################

    # Endurece o tratamento de glue e reduz risco de respostas inconsistentes.
    harden-glue: yes

    # Protege contra respostas tentando “remover” DNSSEC no caminho.
    harden-dnssec-stripped: yes

    # Endurece validação do caminho de referrals.
    harden-referral-path: yes

    # Protege contra algumas respostas “abaixo de NXDOMAIN”.
    harden-below-nxdomain: yes

    # Protege contra tentativas de downgrade de algoritmo.
    harden-algo-downgrade: yes

    # Randomiza maiúsculas/minúsculas no QNAME (mitigação de spoofing).
    use-caps-for-id: yes

    # Pode melhorar performance/negativos com DNSSEC (use com cuidado em ambientes muito heterogêneos).
    aggressive-nsec: yes

    # Reduz tamanho das respostas quando possível.
    minimal-responses: yes

    # Alterna ordem de RRsets para balanceamento simples.
    rrset-roundrobin: yes

    ########################################################################
    # Mitigação básica de abuso “no protocolo”
    #
    # Não substitui firewall/Fail2Ban. A ideia é reduzir impacto de reply floods
    # e respostas “indesejadas” em cenários de ruído/ataque.
    ########################################################################
    unwanted-reply-threshold: 100000
EOF
```

5) Cache/performance e resiliência.  
Crie `/etc/unbound/unbound.conf.d/40-performance-cache.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/40-performance-cache.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Cache / performance (perfil conservador)
    #
    # Este exemplo é pensado para um cenário pequeno (ISP pequeno):
    # - 2 vCPU
    # - 2 GB RAM
    #
    # Objetivo: evitar OOM e manter o serviço estável.
    # Para perfis maiores (8/16/32 GB e ajustes de threads/cache), veja a seção 5.3.
    ########################################################################

    # Threads: ponto de partida. Ajuste por métricas (seção 12.2).
    num-threads: 2

    # Permite múltiplos sockets no mesmo port (um por thread). Ajuda escalabilidade.
    so-reuseport: yes

    ########################################################################
    # Concorrência de recursão (consultas de saída)
    ########################################################################

    # `outgoing-range` e `num-queries-per-thread` afetam concorrência de consultas de saída.
    # Em host com pouco recurso, valores muito altos podem aumentar uso de memória.
    outgoing-range: 512
    num-queries-per-thread: 512

    ########################################################################
    # Buffers do Unbound (dependem também dos limites do kernel)
    ########################################################################

    # Ajuste se necessário (em conjunto com sysctl da seção 9, quando aplicável).
    so-rcvbuf: 1m
    so-sndbuf: 1m

    ########################################################################
    # Cache (memória)
    ########################################################################

    # Cache (perfil 2 GB RAM)
    # Cuidado: cache grande melhora hit rate, mas consome RAM.
    # Em 2 GB, comece conservador e aumente aos poucos somente se houver RAM sobrando (seção 12.2).
    msg-cache-size: 64m
    rrset-cache-size: 128m

    ########################################################################
    # Prefetch (mantém cache “quente”)
    ########################################################################

    # Prefetch (mantém cache “quente”)
    prefetch: yes
    prefetch-key: yes

    ########################################################################
    # Resiliência (serve-expired)
    ########################################################################

    # Serve expirado (resiliência): atende do cache mesmo se upstream falhar.
    #
    # Cuidado: TTL muito alto pode mascarar incidente de upstream por tempo demais.
    # Se você não tem observabilidade/alertas, comece com TTL menor (ex.: 3600).
    serve-expired: yes
    serve-expired-ttl: 3600
    serve-expired-reply-ttl: 30

    ########################################################################
    # TTLs do cache
    ########################################################################

    # TTLs (conservador: respeita TTL baixo)
    cache-min-ttl: 0
    cache-max-ttl: 86400
    cache-max-negative-ttl: 3600
    neg-cache-size: 16m

    ########################################################################
    # Controle de abuso (por IP de origem)
    ########################################################################

    # `ip-ratelimit` limita queries por segundo aceitas por IP de origem.
    # Quando excede, o excesso é **dropado** (o cliente pode ver timeout, não SERVFAIL).
    #
    # - 0 = desativado.
    # - valores muito baixos (ex.: 1, 5, 10) tendem a rate-limitar clientes normais em ISP
    #   (principalmente em picos e em apps que abrem muitas conexões/consultas).
    #
    # Ponto de partida (produção), se você quiser usar:
    # - ISP médio (ponto de partida): 100;
    # - se o Unbound enxerga o IP real do cliente/assinante: 50–200 costuma ser uma faixa razoável (comece em 100);
    # - se o tráfego chega “mascarado” (muitos clientes compartilhando o mesmo IP de origem, ex.: BRAS/CGNAT/NAT):
    #   você precisa de valores maiores (ex.: 200–1000+) ou deve manter desativado e tratar abuso fora do DNS.
    #
    # ATENÇÃO (ISP): qualquer mitigação por IP depende do IP de origem que o Unbound enxerga.
    # Veja a seção 1.2 (CGNAT e origem do IP).
    #
    # Importante (operação): logs do tipo `notice: ip_ratelimit exceeded` não são “erro”.
    # Eles indicam que houve burst acima do limite naquele segundo. Em clientes modernos isso pode acontecer
    # (A/AAAA, e em alguns ambientes também HTTPS/SVCB em paralelo). O efeito é pontual: o excesso é dropado
    # naquele instante e depois volta ao normal (não é bloqueio permanente).
    #
    # Calibração (recomendado):
    # 1) comece com 100,
    # 2) observe por algumas horas/dia,
    # 3) ajuste conforme o contador de rate-limit no Unbound:
    #    sudo unbound-control stats_noreset | grep -E 'queries_ip_ratelimited'
    # Se esse contador subir durante uso normal, aumente o `ip-ratelimit`.
    ip-ratelimit: 100

    ########################################################################
    # Métricas (útil para auditoria/observabilidade)
    ########################################################################

    # Estatísticas (útil para auditoria/métricas; pequeno overhead)
    extended-statistics: yes
EOF
```

6) Logs em arquivo (padrão deste guia).  
Crie `/etc/unbound/unbound.conf.d/50-logging.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/50-logging.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Logs
    #
    # Nota: logs por query/reply (`log-queries`/`log-replies`) são pesados.
    # Se você precisar disso para calibração de Fail2Ban (QPS/NXDOMAIN), veja a seção 11.2 (temporário).
    ########################################################################

    # Mantém logs em arquivo (não syslog/journald).
    use-syslog: no

    # Arquivo de log. Garanta que `/var/log/unbound` existe e que o Unbound
    # tem permissão de escrita (seção 4).
    logfile: "/var/log/unbound/unbound.log"

    # Timestamp legível.
    log-time-ascii: yes

    # Verbosidade operacional (1 costuma ser suficiente). Evite aumentar por longos períodos.
    verbosity: 1

    # Visibilidade de falhas (útil para diagnosticar DNSSEC/upstream).
    log-servfail: yes
EOF
```

7) Controle administrativo local (necessário para reload seguro e métricas).  
Crie `/etc/unbound/unbound.conf.d/90-remote-control.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/90-remote-control.conf >/dev/null <<'EOF'
remote-control:
    ########################################################################
    # unbound-control (administração local)
    ########################################################################

    # Ativa o controle administrativo via `unbound-control` (status, stats, reload, etc).
    control-enable: yes

    # Escuta apenas em loopback (mais seguro).
    control-interface: 127.0.0.1
    control-interface: ::1
EOF
```

### 5.3 Tuning “pé no chão” (cache, threads e slabs)

> Não existe “valor mágico”. O objetivo aqui é dar **pontos de partida** e um método de ajuste.

Regras práticas:
- cache grande melhora latência e reduz tráfego externo, mas **consome RAM**;
- `rrset-cache-size` costuma ficar **maior** que `msg-cache-size`;
- se você usa `num-threads > 1`, configurar `*-cache-slabs` ajuda a reduzir contenção de lock.

Sugestões iniciais (ajuste conforme RAM real do host):

- **2 GB RAM / 2 vCPU (perfil conservador)**: `msg-cache-size 64m` / `rrset-cache-size 128m` / `neg-cache-size 16m`  
  Motivo: reduzir risco de OOM e swap em host pequeno. Comece assim e só aumente se houver RAM sobrando e o cache miss estiver alto (seção 12.2).
- **8 GB RAM**: `msg-cache-size 512m` / `rrset-cache-size 1024m`
- **16 GB RAM**: `msg-cache-size 1g` / `rrset-cache-size 2g`
- **32 GB RAM**: `msg-cache-size 2g` / `rrset-cache-size 4g`

Slabs (opcional, mas recomendado quando `num-threads` > 1):  
use potências de 2 (ex.: 2, 4, 8, 16). Um ponto de partida simples é “próxima potência de 2 ≥ num-threads”.

Exemplo A (se `num-threads: 2`): slabs 2 ou 4 (aqui usando 4)

Crie `/etc/unbound/unbound.conf.d/45-cache-slabs.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/45-cache-slabs.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Cache slabs (shards)
    #
    # Perfil pequeno (2 vCPU / 2 GB): `num-threads: 2`
    # Use 2 ou 4; aqui usamos 4 por ser potência de 2 e dar margem.
    ########################################################################
    msg-cache-slabs: 4
    rrset-cache-slabs: 4
    infra-cache-slabs: 4
    key-cache-slabs: 4
EOF
```

Exemplo B (se `num-threads: 4`, típico de host 8 GB+):

Crie `/etc/unbound/unbound.conf.d/45-cache-slabs.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/45-cache-slabs.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Cache slabs (shards)
    #
    # Ajuda a reduzir contenção de lock em hosts com múltiplas threads.
    # Use potências de 2 (4/8/16...) e ajuste por métricas.
    ########################################################################
    msg-cache-slabs: 8
    rrset-cache-slabs: 8
    infra-cache-slabs: 8
    key-cache-slabs: 8
EOF
```

> **Como validar que está melhorando:** na seção **12.2**, use `unbound-control stats_noreset` e acompanhe `total.num.cachehits`/`total.num.cachemiss` e latência média ao longo de dias.

### 5.4 Valide a sintaxe

```bash
sudo unbound-checkconf
```

> Se o `unbound-checkconf` reclamar que `root.key` ou `root.hints` não existe, siga a seção **6** (DNSSEC e root hints) e rode novamente.

### 5.5 Logrotate para `/var/log/unbound/*.log`

> Como este tutorial usa `logfile:` no Unbound, **logrotate** evita crescimento infinito de disco.

#### 5.5.1 (Debian) AppArmor e `logfile:` do Unbound (quando “Permission denied”)

Em Debian, é comum o Unbound rodar com **AppArmor**. Se você usar `logfile:` e observar:
- `/var/log/unbound/unbound.log` não existe (mesmo após criar diretório/arquivo), ou
- `Could not open logfile ... Permission denied`

Faça estas verificações:

```bash
sudo journalctl -u unbound -n 50 --no-pager
sudo journalctl -k --no-pager | grep -i apparmor | grep -i unbound | tail -50
```

Se aparecer `apparmor="DENIED"` para `/var/log/unbound/...`, a correção recomendada é criar um override local do profile:

```bash
sudo nano /etc/apparmor.d/local/usr.sbin.unbound
```

Adicione (linhas):

```text
/var/log/unbound/ rw,
/var/log/unbound/** rw,
```

> O arquivo `/etc/apparmor.d/local/usr.sbin.unbound` pode estar vazio por padrão. Isso é normal.

Recarregue o profile e reinicie o serviço:

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.unbound
sudo systemctl restart unbound
sudo journalctl -u unbound -n 30 --no-pager
```

Alternativa (se preferir recarregar o serviço do AppArmor):

```bash
sudo systemctl reload apparmor
sudo systemctl restart unbound
```

Validação (arquivo e/ou journald):

```bash
sudo tail -n 50 /var/log/unbound/unbound.log
sudo journalctl -u unbound -n 50 --no-pager
```

Crie `/etc/logrotate.d/unbound`:

```bash
sudo tee /etc/logrotate.d/unbound >/dev/null <<'EOF'
/var/log/unbound/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 unbound unbound

    # Reabre o arquivo de log sem derrubar o serviço
    postrotate
        /usr/sbin/unbound-control log_reopen >/dev/null 2>&1 || true
    endscript
}
EOF
```

### 5.6 (Opcional) Logs via journald (em vez de arquivo)

Se você prefere ler tudo com `journalctl -u unbound -f` (e não quer `logfile:`):

1) Desative o arquivo de log do guia e habilite syslog/journald via `unbound.conf.d`:

```bash
TS="$(date +%F_%H%M%S)"
sudo mv /etc/unbound/unbound.conf.d/50-logging.conf "/etc/unbound/unbound.conf.d/50-logging.conf.off.${TS}"

sudo tee /etc/unbound/unbound.conf.d/50-logging-journald.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Logs via syslog/journald (alternativa ao logfile em arquivo)
    ########################################################################
    use-syslog: yes
    # logfile: "/var/log/unbound/unbound.log"
EOF
```

2) Reinicie e acompanhe:

```bash
sudo systemctl restart unbound
sudo journalctl -u unbound -f
```

---

<a id="6"></a>
## 6. DNSSEC e root hints (root.key / named.cache)

> **Por que DNSSEC aqui?** Um resolver com validação DNSSEC ajuda a proteger seus clientes contra respostas DNS adulteradas (ex.: cache poisoning), aumentando a integridade das resoluções.

### 6.1 Gerar/atualizar trust anchor DNSSEC

```bash
sudo unbound-anchor -a /var/lib/unbound/root.key
sudo chown unbound:unbound /var/lib/unbound/root.key
sudo chmod 0644 /var/lib/unbound/root.key
```

> Em geral, o `root.key` muda raramente. Mesmo assim, em operação contínua, é saudável ter um update periódico (mensal) para evitar surpresas em eventos de rotação.

#### 6.1.1 (Opcional) Atualização automática do trust anchor (mensal)

```bash
sudo tee /etc/systemd/system/unbound-rootkey.service >/dev/null <<'EOF'
[Unit]
Description=Atualiza trust anchor DNSSEC (root.key) do Unbound
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStartPre=/usr/bin/install -d -m 0755 -o unbound -g unbound /var/lib/unbound
ExecStart=/usr/sbin/unbound-anchor -a /var/lib/unbound/root.key
ExecStartPost=/bin/chown unbound:unbound /var/lib/unbound/root.key
ExecStartPost=/bin/chmod 0644 /var/lib/unbound/root.key
EOF
```

```bash
sudo tee /etc/systemd/system/unbound-rootkey.timer >/dev/null <<'EOF'
[Unit]
Description=Timer mensal: atualiza trust anchor DNSSEC (root.key)

[Timer]
OnCalendar=monthly
Persistent=true
RandomizedDelaySec=7d

[Install]
WantedBy=timers.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now unbound-rootkey.timer
```

### 6.2 Baixar root hints

```bash
sudo curl -fsSL https://www.internic.net/domain/named.cache -o /etc/unbound/root.hints
sudo chown root:unbound /etc/unbound/root.hints
sudo chmod 0644 /etc/unbound/root.hints
```

> Alternativa: em alguns ambientes, o pacote `dns-root-data` fornece root hints prontos. Mesmo assim, manter um updater (semanal/mensal) ajuda a evitar drift em longo prazo.

### 6.3 (Opcional) Atualização automática de root hints via systemd timer

Crie o script:

```bash
sudo tee /usr/local/bin/update-root-hints.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
umask 022

TMP="$(mktemp)"
trap 'rm -f "$TMP"' EXIT

curl -fsSL \
  --retry 3 --retry-delay 5 \
  --connect-timeout 10 --max-time 60 \
  https://www.internic.net/domain/named.cache -o "$TMP"

# Validação mínima: o arquivo precisa conter ao menos um registro NS do root zone.
grep -qE '^\\.[[:space:]]+.*[[:space:]]+NS[[:space:]]+' "$TMP"

install -m 0644 -o root -g unbound "$TMP" /etc/unbound/root.hints
EOF
sudo chmod 0755 /usr/local/bin/update-root-hints.sh
```

Crie o service:

```bash
sudo tee /etc/systemd/system/unbound-root-hints.service >/dev/null <<'EOF'
[Unit]
Description=Atualiza root hints do Unbound (Internic)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/update-root-hints.sh
EOF
```

Crie o timer (semanal):

```bash
sudo tee /etc/systemd/system/unbound-root-hints.timer >/dev/null <<'EOF'
[Unit]
Description=Timer semanal para atualizar root hints do Unbound

[Timer]
OnCalendar=weekly
Persistent=true
RandomizedDelaySec=12h

[Install]
WantedBy=timers.target
EOF
```

Ative:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now unbound-root-hints.timer
```

### 6.4 Subir o Unbound e habilitar o `unbound-control` (necessário para reload seguro)

Gere as chaves/certs do `unbound-control`:

```bash
sudo unbound-control-setup
```

Suba o serviço e valide:

```bash
sudo systemctl restart unbound
sudo systemctl status unbound --no-pager

sudo unbound-control status
```

> Neste ponto, o Unbound ainda está em loopback (arquivo `00-server.conf`).  
> A habilitação no boot e a abertura de interface para as redes do ISP ficam para a seção **12.1**, depois do firewall estar aplicado.

---

<a id="7"></a>
## 7. RPZ (conceito, configuração e modo sombra)

> **Resumo (RPZ neste guia):**
> - objetivo: bloquear **ameaças** (phishing/malware/C2) via DNS, com governança (allowlist, auditoria e rollback);
> - estratégia: **duas RPZs** (allowlist primeiro, blocklist depois);
> - operação segura: comece em **modo sombra** (log sem bloquear) e só depois ative NXDOMAIN.
>
> **Checklist rápido:**
> 1) `unbound-control` funcional (seção 6.4) para reload sem derrubar cache;
> 2) `/var/lib/unbound/rpz` existe e tem permissão para o usuário `unbound` (seção 4);
> 3) você vai seguir a ordem allowlist → blocklist (seção 7.5);
> 4) você sabe onde mexe: política em `/etc/unbound/unbound.conf.d/60-rpz-seguranca.conf`, zonefiles em `/var/lib/unbound/rpz/` e allowlist em `/etc/unbound/rpz/rpz-allowlist.txt`.

### 7.1 Por que RPZ aqui (e por que não `local-zone`)

Para uma política de bloqueio **focada em ameaças** (“phishing/malware/botnet/C2”), você quer:
- política **explicável e auditável** (zona DNS),
- ação padronizada (NXDOMAIN),
- capacidade de **modo sombra** (logar sem bloquear),
- e updates frequentes com rollback.

**RPZ** (Response Policy Zone) atende isso bem porque:
- é uma **zona DNS** (texto) que vira política;
- dá para versionar/assinar internamente, auditar e comparar diffs;
- permite ações como **NXDOMAIN**, **DROP** e **PASSTHRU**.

`local-zone` também funciona, mas tende a virar um “bloco gigante” menos padronizado para auditoria e menos flexível quando você evolui para múltiplas políticas.

### 7.2 Unbound RPZ vs BIND RPZ

- No **BIND**, RPZ é implementada no próprio named (autoridade + recursor).  
- No **Unbound**, RPZ é implementada via módulo **`respip`** + blocos **`rpz:`**.

O que muda na prática:
- no Unbound você ativa explicitamente `module-config: "respip validator iterator"`;
- o Unbound aplica RPZs **na ordem configurada**;
- ações especiais como **PASSTHRU** podem parar a avaliação de outras zonas.

### 7.3 Tipos de gatilho (triggers) em RPZ 
1) **QNAME trigger** (o mais usado em RPZ de segurança)  
   - bloqueia por **nome** (`dominio.tld` e `*.dominio.tld`);
   - ideal para feeds como URLhaus/PhishTank (domínios maliciosos).

2) **Response IP trigger** (avançado, use com extremo cuidado)  
   - bloqueia quando a resposta resolver para um IP/prefixo “marcado”;
   - pode gerar **falsos positivos** em CDN/hosting compartilhado.

3) **Client IP trigger** (casos específicos)  
   - permite aplicar política por “origem do cliente”.

> Para RPZ de segurança, o guia foca em **QNAME trigger** porque é o modelo mais defensável e com menor risco de dano colateral.

### 7.4 Ações de política o que o cliente “vê”

As ações mais úteis:
- **NXDOMAIN** (recomendado para bloqueio por segurança): “domínio não existe”.
- **NODATA**: existe, mas não tem aquele tipo (A/AAAA).
- **DROP**: “silencia” (pode parecer lentidão; use com cautela).
- **PASSTHRU**: permite explicitamente e **pode** impedir que outras RPZs bloqueiem.
- **DISABLED**: desativa a ação (ótimo para **modo sombra**).

Neste guia, o padrão é:
- **bloqueio = NXDOMAIN**
- **allowlist = PASSTHRU**
- **modo sombra = DISABLED (log sem bloquear)**

### 7.5 Desenho recomendado: allowlist primeiro, blocklist depois

Vamos usar **duas RPZs**:

1) **RPZ Allowlist** (manual)  
   - poucos registros;
   - ação `rpz-passthru.` para “desbloquear” exceções;
   - deve vir **ANTES** da blocklist.

2) **RPZ Blocklist (segurança)** (gerada por feeds)  
   - muitos registros;
   - ação NXDOMAIN (via `rpz-action-override: nxdomain`);
   - logs habilitados para auditoria e Fail2Ban.

### 7.6 Arquivo de configuração RPZ (`unbound.conf.d`)

Crie `/etc/unbound/unbound.conf.d/60-rpz-seguranca.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/60-rpz-seguranca.conf >/dev/null <<'EOF'
# Ordem importa: allowlist primeiro.

server:
    ########################################################################
    # Módulos (RPZ + DNSSEC + recursão)
    #
    # `respip` aplica a política de RPZ.
    # `validator` mantém DNSSEC ativo.
    # `iterator` faz a recursão desde a raiz (root hints).
    ########################################################################
    module-config: "respip validator iterator"

rpz:
    ########################################################################
    # RPZ 1/2: Allowlist (PASSTHRU)
    #
    # Objetivo: permitir exceções (desbloqueios) de forma auditável, antes
    # da blocklist. Deve vir ANTES da RPZ de bloqueio.
    ########################################################################
    name: "rpz.allow.seguranca.exemplo"
    zonefile: "/var/lib/unbound/rpz/rpz.allow.seguranca.exemplo.zone"
    # Allowlist é manual e pequena; por padrão não logamos para não gerar ruído.
    rpz-log: no
    for-downstream: no

rpz:
    ########################################################################
    # RPZ 2/2: Blocklist (segurança)
    #
    # Objetivo: bloquear ameaças (phishing/malware/C2) via NXDOMAIN.
    # A ação é controlada por `rpz-action-override`.
    ########################################################################
    name: "rpz.seguranca.exemplo"
    zonefile: "/var/lib/unbound/rpz/rpz.seguranca.exemplo.zone"

    # Padrão de bloqueio (segurança):
    rpz-action-override: nxdomain

    # Auditoria/visibilidade (e base para Fail2Ban):
    rpz-log: yes
    rpz-log-name: "rpz-seguranca"

    for-downstream: no
EOF
```

### 7.7 Zonefiles iniciais (placeholders)

Crie a RPZ allowlist (mesmo vazia, ela ajuda a manter o desenho consistente):

```bash
sudo tee /var/lib/unbound/rpz/rpz.allow.seguranca.exemplo.zone >/dev/null <<'EOF'
$TTL 60
$ORIGIN rpz.allow.seguranca.exemplo.

@   IN SOA localhost. hostmaster.rpz.allow.seguranca.exemplo. (
        2026012001 ; serial (YYYYMMDDNN)
        3600       ; refresh
        900        ; retry
        604800     ; expire
        60         ; negative caching
)

@   IN NS  localhost.

; Exemplo (descomente para testar):
; banco.exemplo        CNAME rpz-passthru.
; *.banco.exemplo      CNAME rpz-passthru.
EOF
```

Crie a RPZ blocklist (segurança):

```bash
sudo tee /var/lib/unbound/rpz/rpz.seguranca.exemplo.zone >/dev/null <<'EOF'
$TTL 60
$ORIGIN rpz.seguranca.exemplo.

@   IN SOA localhost. hostmaster.rpz.seguranca.exemplo. (
        2026012001 ; serial (YYYYMMDDNN)
        3600       ; refresh
        900        ; retry
        604800     ; expire
        60         ; negative caching
)

@   IN NS  localhost.

; Entradas de teste (remova depois)
teste-malware.exemplo     CNAME .
*.teste-malware.exemplo   CNAME .
EOF
```

Permissões:

```bash
sudo chown unbound:unbound /var/lib/unbound/rpz/rpz.allow.seguranca.exemplo.zone /var/lib/unbound/rpz/rpz.seguranca.exemplo.zone
sudo chmod 0644 /var/lib/unbound/rpz/rpz.allow.seguranca.exemplo.zone /var/lib/unbound/rpz/rpz.seguranca.exemplo.zone
```

### 7.8 (Recomendado) Modo sombra (log sem bloquear)

Antes de “ligar o bloqueio” em produção, rode em **modo sombra** por 24–72h:

1) Edite `/etc/unbound/unbound.conf.d/60-rpz-seguranca.conf` e troque na RPZ blocklist:

```conf
    rpz-action-override: nxdomain
```

por:

```conf
    rpz-action-override: disabled
```

2) Recarregue e observe logs:

```bash
sudo unbound-control reload
sudo unbound-control status
sudo unbound-control stats_noreset | head
sudo tail -f /var/log/unbound/unbound.log
```

> Se o arquivo `/var/log/unbound/unbound.log` não existir ou aparecer “Permission denied”, revise a seção **4** (permissões) e confirme o erro no `journalctl`:
> ```bash
> sudo journalctl -u unbound -n 50 --no-pager
> ```
> Se houver negação de AppArmor, siga a seção **5.5.1** para liberar escrita em `/var/log/unbound/**` no profile.
>
> Se você estiver usando logs via journald (seção 5.6), troque o `tail` por:
> ```bash
> sudo journalctl -u unbound -f
> ```

Quando estiver confiante (falsos positivos sob controle), volte para `nxdomain`.

### 7.9 Wildcards: por que `dominio` + `*.dominio`

Em feed de ameaças, é comum ver:
- domínio raiz malicioso (`exemplo.tld`)
- subdomínios rotativos (`a1.exemplo.tld`, `b2.exemplo.tld`…)

Por isso o gerador cria sempre:
- `dominio CNAME .` (bloqueia o raiz)
- `*.dominio CNAME .` (bloqueia subdomínios)

> Isso é especialmente útil contra famílias DGA e infra de phishing que rotaciona subdomínios.

### 7.10 Nota: Response IP triggers e risco de dano colateral

É tentador “bloquear IPs ruins” via RPZ, mas em Internet moderna muitos IPs são:
- compartilhados (CDN/Cloud/hosting),
- reutilizados,
- e mudam frequentemente.

Use Response IP trigger apenas quando você tiver:
- fonte extremamente confiável,
- e uma estratégia forte de allowlist/observabilidade.

Neste guia, isso fica como **leitura complementar**, sem habilitar por padrão.

---

<a id="8"></a>
## 8. Automatização (scripts + timers)

> **Resumo:**
> - o script gera zonefiles RPZ de forma reprodutível (normalização, dedupe, hash e rollback);
> - o `systemd timer` executa periodicamente com atraso aleatório para evitar “picos sincronizados” em múltiplos hosts.
>
> **Checklist rápido:**
> 1) configure allowlist e opções (seção 8.1);
> 2) execute o script manualmente uma vez e valide logs/zonefiles (seção 8.2);
> 3) só depois habilite o timer (seção 8.4);
> 4) arquivos-chave: `/usr/local/bin/update-unbound-rpz.sh`, `/etc/systemd/system/unbound-rpz-update.{service,timer}`, `/var/log/unbound/rpz-update.log`.

### 8.1 Pastas e arquivos auxiliares (allowlist / PhishTank opcional)

Allowlist (exceções/PASSTHRU), 1 domínio por linha:

```bash
sudo tee /etc/unbound/rpz/rpz-allowlist.txt >/dev/null <<'EOF'
# Coloque aqui exceções que você quer PERMITIR mesmo que apareçam em feeds.
#
# Exemplos:
# - permitir um domínio inteiro:
#   exemplo.com.br
# - permitir apenas um subdomínio específico:
#   login.banco.exemplo
EOF
sudo chown root:unbound /etc/unbound/rpz/rpz-allowlist.txt
sudo chmod 0640 /etc/unbound/rpz/rpz-allowlist.txt
```

PhishTank: este guia usa o **dump público** do PhishTank, sem necessidade de chave/autenticação.

Links oficiais do dump público:
- CSV: http://data.phishtank.com/data/online-valid.csv
- JSON: http://data.phishtank.com/data/online-valid.json
- Página oficial (descrição, limites e boas práticas): https://phishtank.org/developer_info.php

No script deste guia, o PhishTank vem **desativado por padrão** (`ENABLE_PHISHTANK=0`) para respeitar limites/fair use.  
Se você quiser habilitar, altere para `ENABLE_PHISHTANK=1` no topo do script.

> **Nota (informativo):** o PhishTank também oferece uma API de consulta (desenvolvedores).  
> Para uso em **RPZ/DNS de ISP**, não é indicado depender de chamadas de API, nem automatizar agressivamente.  
> Prefira dumps e respeite termos/rate limits.

### 8.2 Script de atualização: `/usr/local/bin/update-unbound-rpz.sh`

> O que este script faz:
> 1) baixa feeds, 
> 2) extrai domínios, 
> 3) valida/deduplica,
> 4) gera **RPZ blocklist** e **RPZ allowlist**, 
> 5) faz update **atômico**, 
> 6) recarrega Unbound, 
> 7) faz rollback automático em falha.

> **IDN (domínios com acentos / Unicode):** este script trabalha com domínios **ASCII** e aceita **punycode** (`xn--...`). Ele **não converte** IDN Unicode para punycode. Se o seu feed vier com Unicode, padronize o feed para ASCII/punycode antes de usar.

```bash
sudo tee /usr/local/bin/update-unbound-rpz.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

umask 027
PATH=/usr/sbin:/usr/bin:/sbin:/bin

LOCK_FILE="/run/unbound-rpz.lock"
exec 9>"$LOCK_FILE"
flock -n 9 || exit 0

USER_AGENT="unbound-rpz/1.0"
SCRIPT_NAME="$(basename "$0")"
SOURCES="URLhaus (abuse.ch hostfile) + PhishTank (dump público online-valid.csv, opcional)"

MIN_BYTES_URLHAUS=2048
MIN_BYTES_PHISHTANK=256

ZONE_BLOCK_NAME="rpz.seguranca.exemplo"
ZONE_BLOCK_FILE="/var/lib/unbound/rpz/${ZONE_BLOCK_NAME}.zone"

ZONE_ALLOW_NAME="rpz.allow.seguranca.exemplo"
ZONE_ALLOW_FILE="/var/lib/unbound/rpz/${ZONE_ALLOW_NAME}.zone"

ALLOWLIST="/etc/unbound/rpz/rpz-allowlist.txt"
ENABLE_PHISHTANK=0
PHISHTANK_URL_PUBLIC="http://data.phishtank.com/data/online-valid.csv"
LOG_FILE="/var/log/unbound/rpz-update.log"

WORKDIR="$(mktemp -d)"
trap 'rm -rf "$WORKDIR"' EXIT

log() {
  echo "[$(date +'%F %T')] $*" | tee -a "$LOG_FILE"
}

curl_fetch() {
  # Centraliza opções do curl e User-Agent (útil para compliance e troubleshooting).
  curl --fail --show-error --silent --location \
    --connect-timeout 10 --max-time 180 \
    --retry 3 --retry-delay 2 --retry-connrefused \
    -A "$USER_AGENT" "$@"
}

file_size_bytes() {
  # Evita diferenças de `stat` entre distros/ambientes.
  wc -c < "$1" | tr -d ' '
}

assert_min_bytes() {
  local file="$1"
  local min_bytes="$2"
  local label="$3"

  local size
  size="$(file_size_bytes "$file")"
  if [[ "$size" -lt "$min_bytes" ]]; then
    log "ERRO: download inválido/pequeno demais para ${label} (${size} bytes; mínimo ${min_bytes})."
    return 1
  fi
}

extract_domains_from_urlhaus_hostfile() {
  # URLhaus hostfile: formato hosts (linhas tipo "127.0.0.1 dominio")
  awk '$1 == "127.0.0.1" && $2 != "" {print $2}' "$1"
}

extract_domains_from_phishtank_csv() {
  # PhishTank retorna URLs; extraímos apenas o host (domínio).
  # Este guia usa o dump público (online-valid.csv). A API existe (informativo),
  # mas não é necessária aqui.
  #
  # Formato esperado: CSV com URL em algum campo; aqui fazemos um parse simples:
  # - filtra somente linhas com http/https
  # - remove aspas
  # - pega substring após esquema
  # - corta no primeiro /
  (grep -E 'https?://' "$1" || true) \
    | sed 's/\"//g' \
    | sed -E 's#.*https?://##g' \
    | sed -E 's#/.*##g' \
    | sed -E 's/:[0-9]+$//g'
}

is_domain() {
  # Regex prática para domínios (ASCII/punycode); evita IP e lixo comum.
  [[ "$1" =~ ^[A-Za-z0-9]([A-Za-z0-9-]{0,61}[A-Za-z0-9])?(\.[A-Za-z0-9]([A-Za-z0-9-]{0,61}[A-Za-z0-9])?)+$ ]]
}

normalize_domains() {
  # Normalização para zona RPZ:
  # - lowercase
  # - remove CRLF
  # - remove comentários (linhas inteiras ou inline) e linhas vazias
  # - remove "*." e trailing dot
  # - sort -u (dedup)
  tr '[:upper:]' '[:lower:]' \
    | tr -d '\r' \
    | sed -E 's/[[:space:]]*[#;].*$//' \
    | sed -E 's/^[[:space:]]+//; s/[[:space:]]+$//' \
    | sed -E 's/^\*\.//' \
    | sed -E 's/\.$//' \
    | awk 'NF' \
    | while IFS= read -r d; do
        if is_domain "$d"; then
          echo "$d"
        fi
      done \
    | sort -u
}

zone_header() {
  local origin="$1"
  local serial="$2"
  local kind="$3"
  local count="$4"

  cat <<EOF2
; Fonte: ${SOURCES}
; Gerado em: ${GEN_TS}
; Script: ${SCRIPT_NAME}
; Tipo: ${kind}
; Entradas: ${count}
\$TTL 60
\$ORIGIN ${origin}.

@   IN SOA localhost. hostmaster.${origin}. (
        ${serial} ; serial
        3600      ; refresh
        900       ; retry
        604800    ; expire
        60        ; negative caching
)
@   IN NS  localhost.
EOF2
}

stage_zone_atomic() {
  local src="$1"
  local dst="$2"

  local staged="${dst}.new"
  install -m 0644 -o unbound -g unbound "$src" "$staged"
  mv -f "$staged" "$dst"
}

sha256_file() {
  sha256sum "$1" | awk '{print $1}'
}

log "Iniciando atualização RPZ (segurança)..."

mkdir -p /var/log/unbound /var/lib/unbound/rpz
touch "$LOG_FILE"
if ! chown unbound:unbound "$LOG_FILE" 2>/dev/null; then
  log "AVISO: não foi possível ajustar owner de ${LOG_FILE} (verifique usuário/grupo 'unbound')."
fi
if ! chown unbound:unbound /var/lib/unbound/rpz 2>/dev/null; then
  log "AVISO: não foi possível ajustar owner de /var/lib/unbound/rpz (verifique usuário/grupo 'unbound')."
fi

DOMAINS_RAW="$WORKDIR/domains.raw"
DOMAINS_BLOCK="$WORKDIR/domains.block"
DOMAINS_ALLOW="$WORKDIR/domains.allow"

ZONE_BLOCK_NEW="$WORKDIR/${ZONE_BLOCK_NAME}.zone.new"
ZONE_ALLOW_NEW="$WORKDIR/${ZONE_ALLOW_NAME}.zone.new"

##############################################################################
# 1) Baixar feeds (somente fontes de segurança, sem listas “controversas”)
##############################################################################

# Fonte 1: URLhaus (abuse.ch), malware/C2 (hostfile)
URLHAUS_URL="https://urlhaus.abuse.ch/downloads/hostfile/"
URLHAUS_FILE="$WORKDIR/urlhaus.hosts"
log "Baixando URLhaus hostfile..."
curl_fetch -o "$URLHAUS_FILE" "$URLHAUS_URL"
assert_min_bytes "$URLHAUS_FILE" "$MIN_BYTES_URLHAUS" "URLhaus hostfile"
extract_domains_from_urlhaus_hostfile "$URLHAUS_FILE" >> "$DOMAINS_RAW"

# Fonte 2: PhishTank (opcional), phishing (dump público)
#
# Este guia não exige chave/autenticação. Por padrão, deixamos desativado para respeitar limites/fair use.
# Para habilitar, altere `ENABLE_PHISHTANK=1` (no topo do script).
if [[ "${ENABLE_PHISHTANK}" == "1" ]]; then
  PHISHTANK_FILE="$WORKDIR/phishtank.csv"
  log "Baixando PhishTank (dump público online-valid.csv)..."
  if curl_fetch -o "$PHISHTANK_FILE" "$PHISHTANK_URL_PUBLIC"; then
    if assert_min_bytes "$PHISHTANK_FILE" "$MIN_BYTES_PHISHTANK" "PhishTank CSV"; then
      extract_domains_from_phishtank_csv "$PHISHTANK_FILE" >> "$DOMAINS_RAW" || true
    else
      log "AVISO: PhishTank retornou pouco/nada (dump indisponível ou rate limit)."
    fi
  else
    log "AVISO: falha ao baixar PhishTank (dump indisponível ou rate limit)."
  fi
fi

##############################################################################
# 2) Normalizar / validar / deduplicar
##############################################################################

log "Normalizando e validando domínios..."
normalize_domains < "$DOMAINS_RAW" > "$DOMAINS_BLOCK"

if [[ -s "$ALLOWLIST" ]]; then
  normalize_domains < "$ALLOWLIST" > "$DOMAINS_ALLOW" || true
else
  : > "$DOMAINS_ALLOW"
fi

TOTAL_BLOCK="$(wc -l < "$DOMAINS_BLOCK" | tr -d ' ')"
TOTAL_ALLOW="$(wc -l < "$DOMAINS_ALLOW" | tr -d ' ')"
log "Domínios na blocklist (após limpeza): $TOTAL_BLOCK"
log "Domínios na allowlist (após limpeza): $TOTAL_ALLOW"

if [[ "$TOTAL_BLOCK" -lt 10 ]]; then
  log "ERRO: blocklist pequena demais (${TOTAL_BLOCK}). Abortando para evitar publicar RPZ vazia."
  exit 1
fi

##############################################################################
# 3) Gerar zonefiles RPZ (QNAME triggers)
##############################################################################

GEN_TS="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
# Serial do zonefile:
# - é um controle interno/auditável para facilitar diff e troubleshooting;
# - não “precisa” seguir um padrão rígido de autoridade (não é zona pública autoritativa).
SERIAL="$(date -u +%Y%m%d%H)"
zone_header "$ZONE_ALLOW_NAME" "$SERIAL" "allowlist (rpz-passthru)" "$TOTAL_ALLOW" > "$ZONE_ALLOW_NEW"
zone_header "$ZONE_BLOCK_NAME" "$SERIAL" "blocklist (NXDOMAIN via rpz-action-override)" "$TOTAL_BLOCK" > "$ZONE_BLOCK_NEW"

# Allowlist: PASSTHRU (deve vir antes na política RPZ em /etc/unbound/unbound.conf.d/60-rpz-seguranca.conf)
while read -r d; do
  [[ -n "$d" ]] || continue
  echo "${d} CNAME rpz-passthru." >> "$ZONE_ALLOW_NEW"
  echo "*.${d} CNAME rpz-passthru." >> "$ZONE_ALLOW_NEW"
done < "$DOMAINS_ALLOW"

# Blocklist: NXDOMAIN (será aplicado via rpz-action-override na política RPZ em /etc/unbound/unbound.conf.d/60-rpz-seguranca.conf)
while read -r d; do
  [[ -n "$d" ]] || continue
  echo "${d} CNAME ." >> "$ZONE_BLOCK_NEW"
  echo "*.${d} CNAME ." >> "$ZONE_BLOCK_NEW"
done < "$DOMAINS_BLOCK"

##############################################################################
# 4) Swap atômico + reload seguro (com rollback)
##############################################################################

log "Validando unbound.conf..."
unbound-checkconf >/dev/null

if [[ -f "$ZONE_ALLOW_FILE" ]]; then
  cp -a "$ZONE_ALLOW_FILE" "${ZONE_ALLOW_FILE}.bak"
fi
if [[ -f "$ZONE_BLOCK_FILE" ]]; then
  cp -a "$ZONE_BLOCK_FILE" "${ZONE_BLOCK_FILE}.bak"
fi

stage_zone_atomic "$ZONE_ALLOW_NEW" "$ZONE_ALLOW_FILE"
stage_zone_atomic "$ZONE_BLOCK_NEW" "$ZONE_BLOCK_FILE"

reload_unbound() {
  # Preferência: unbound-control (reload sem derrubar cache)
  if command -v unbound-control >/dev/null 2>&1; then
    if unbound-control status >/dev/null 2>&1; then
      unbound-control reload >/dev/null 2>&1 && return 0
    fi
  fi

  # Fallback: systemd
  if command -v systemctl >/dev/null 2>&1; then
    systemctl reload unbound >/dev/null 2>&1 && return 0
    systemctl restart unbound >/dev/null 2>&1 && return 0
  fi

  return 1
}

log "Recarregando Unbound..."
reload_unbound || {
  log "ERRO: reload falhou. Restaurando backup..."
  if [[ -f "${ZONE_ALLOW_FILE}.bak" ]]; then
    cp -a "${ZONE_ALLOW_FILE}.bak" "$ZONE_ALLOW_FILE"
  else
    log "AVISO: backup allowlist não existe (${ZONE_ALLOW_FILE}.bak)."
  fi

  if [[ -f "${ZONE_BLOCK_FILE}.bak" ]]; then
    cp -a "${ZONE_BLOCK_FILE}.bak" "$ZONE_BLOCK_FILE"
  else
    log "AVISO: backup blocklist não existe (${ZONE_BLOCK_FILE}.bak)."
  fi

  if ! reload_unbound; then
    log "AVISO: reload falhou mesmo após rollback."
  fi
  exit 1
}

HASH_ALLOW="$(sha256_file "$ZONE_ALLOW_FILE")"
HASH_BLOCK="$(sha256_file "$ZONE_BLOCK_FILE")"
log "SHA256 allowlist zone: $HASH_ALLOW"
log "SHA256 blocklist zone: $HASH_BLOCK"
log "Atualização concluída com sucesso."
EOF
```

Torne o script executável

```bash
sudo chmod 0755 /usr/local/bin/update-unbound-rpz.sh
```

### 8.3 Nota sobre feeds (PhishTank / Spamhaus / licenças)

- **URLhaus (abuse.ch)**: excelente para malware/C2. Verifique **termos de uso** antes de uso comercial e respeite limites/fair use.
- **PhishTank**: o script pode usar o **dump público** (online-valid.csv), sem chamadas de API. A API existe (informativo), mas não é indicada para uso direto em RPZ/DNS de ISP; respeite termos/rate limits.
- **Spamhaus DROP/EDROP**: listas de **IP/prefixos** (não de domínios). Faz mais sentido em firewall/antiabuso ou em RPZ por **Response IP** (avançado e com risco de falso positivo).

> **Importante:** este guia **não fornece feed/lista pronta** de domínios. Ele mostra **como consumir** feeds *quando* você tiver fontes com **licença/termos compatíveis** com o seu uso e como transformar isso em zonefiles RPZ com governança (allowlist, auditoria e rollback).

Boas práticas de “higiene” com feeds:
- atualize com frequência razoável (ex.: **6h–24h**) e use **jitter** (ver timer abaixo);
- mantenha `User-Agent` e logs de update (auditoria);
- tenha allowlist e rollback definidos antes de ligar NXDOMAIN em produção.

Critério de escolha (por que este guia evita listas “controversas”):
- O objetivo aqui é **segurança**, não “controle de conteúdo”. Feeds de ads/tracking/pornografia/apostas etc aumentam risco jurídico e dano colateral.
- Em ISP, foque em fontes com reputação e objetivo claro de **ameaça** (phishing/malware/C2), e mantenha processo de revisão/allowlist.

### 8.4 systemd service + timer (a cada 6 horas, com jitter)

Service:

```bash
sudo tee /etc/systemd/system/unbound-rpz-update.service >/dev/null <<'EOF'
[Unit]
Description=Atualiza RPZ de segurança para Unbound
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/update-unbound-rpz.sh
Nice=10
IOSchedulingClass=idle
EOF
```

Timer:

```bash
sudo tee /etc/systemd/system/unbound-rpz-update.timer >/dev/null <<'EOF'
[Unit]
Description=Timer: atualiza RPZ (segurança) a cada 6 horas

[Timer]
OnCalendar=*-*-* 00,06,12,18:00:00
Persistent=true
AccuracySec=5m
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
EOF
```

> **Dica operacional:** evite alinhar timers de múltiplos hosts para “rodar todo mundo junto”, em um segundo servidor por exemplo pode usar `OnCalendar=*-*-* 01,07,13,19:30:00`
> Use `RandomizedDelaySec` (e, se necessário, calendários diferentes por POP) para reduzir picos simultâneos e evitar sobrecarga nos feeds.

Ative:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now unbound-rpz-update.timer
```

Execute manualmente a primeira vez:

```bash
sudo /usr/local/bin/update-unbound-rpz.sh
```

> Se o script reclamar de `unbound-control`/`reload`, confirme que você fez o passo **6.4** (chaves do `unbound-control` + serviço `unbound` rodando).

---

<a id="9"></a>
## 9. Ajustes de kernel (opcional, pós-observabilidade)

> **OPCIONAL:** se você **não** tem métricas indicando drops/erros (ex.: `UdpRcvbufErrors`, softnet drops, exaustão de portas), **não aplique ainda**.  
> Primeiro observe comportamento real em produção ou em teste controlado e volte aqui quando houver evidência.

### Contexto / por quê

Em recursor de ISP, estes ajustes fazem mais diferença quando você tem:
- **picos reais de QPS**,
- jitter/perda no caminho que aumenta retries,
- sinais de fila/buffer estourando (drops/erros UDP),
- exaustão de portas efêmeras (muitas consultas de saída simultâneas).

Se o seu recursor é pequeno e estável, trate como **opcional** e volte aqui quando tiver métricas que indiquem necessidade.

### O que fazer agora se você decidiu aplicar

Crie um arquivo dedicado:

```bash
sudo tee /etc/sysctl.d/99-unbound-isp.conf >/dev/null <<'EOF'
# Ajustes básicos para DNS recursivo em alto volume (ajuste conforme sua realidade)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 10000
net.core.somaxconn = 4096
net.ipv4.udp_rmem_min = 16384

# Ajuda em cenários de muitas conexões/consultas de saída (muito QPS)
net.ipv4.ip_local_port_range = 10240 65535
EOF
```

Aplique:

```bash
sudo sysctl --system
```

#### Explicação detalhada

O que cada parâmetro faz, quando ajuda e cuidados práticos:

- `net.core.rmem_max` / `net.core.wmem_max`  
  - **O que faz:** define o teto de buffer de recepção/transmissão dos sockets (inclui UDP).  
  - **Quando ajuda:** bursts e jitter/perda que geram retries.  
  - **Impacto/cuidado:** buffers maiores podem aumentar consumo de RAM; ajuste com métricas.

- `net.core.netdev_max_backlog`  
  - **O que faz:** tamanho máximo da fila de pacotes quando o kernel não processa imediatamente.  
  - **Quando ajuda:** rajadas curtas (bursts) e picos de softirq.  
  - **Impacto/cuidado:** backlog maior não cria CPU; se o host satura, a latência cresce.

- `net.core.somaxconn`  
  - **O que faz:** fila de conexões pendentes (mais relevante para TCP).  
  - **Quando ajuda:** picos de **TCP/53** (fallback de truncation/DNSSEC).  
  - **Impacto/cuidado:** não muda UDP diretamente; mantenha por consistência.

- `net.ipv4.udp_rmem_min`  
  - **O que faz:** mínimo de buffer de recepção por socket UDP.  
  - **Quando ajuda:** risco de `RcvbufErrors` em carga alta.  
  - **Impacto/cuidado:** o efeito aparece quando você está vendo erros/drops por buffer.

- `net.ipv4.ip_local_port_range`  
  - **O que faz:** faixa de portas efêmeras (origem) para consultas de saída.  
  - **Quando ajuda:** muita concorrência de consultas externas (muito QPS).  
  - **Impacto/cuidado:** não resolve limites do upstream; só dá mais espaço local para sockets.

Valide por métricas (antes/depois):

- **Erros/drops UDP (buffer):**  
  ```bash
  nstat -az | egrep 'Udp(InErrors|RcvbufErrors|SndbufErrors)'
  ```

- **Backlog/atraso de processamento (softnet):**  
  ```bash
  cat /proc/net/softnet_stat | head
  ```

- **Portas efêmeras:**  
  ```bash
  sudo sysctl net.ipv4.ip_local_port_range
  ```

### 9.1 (Opcional) Aumente o limite de arquivos (fd) do serviço

Em recursor de ISP, é comum precisar de mais file descriptors (sockets).

Crie um override do systemd:

```bash
sudo mkdir -p /etc/systemd/system/unbound.service.d
sudo systemctl edit unbound
```

Cole:

```ini
[Service]
LimitNOFILE=262144
```

Depois:

```bash
sudo systemctl daemon-reload
sudo systemctl restart unbound
```

> Boa prática: faça isso em janela ou com redundância, e depois valide saúde do serviço (seção 12) e o limite efetivo no processo:
> ```bash
> pid="$(pgrep -x unbound || true)"
> [ -n "$pid" ] && cat "/proc/${pid}/limits" | grep -i 'open files'
> ```

---

<a id="10"></a>
## 10. Firewall com nftables default drop, liberando só DNS para redes do ISP

**Contexto / por quê:** recursor de ISP não pode virar “DNS público”. Aqui o objetivo é manter política padrão `drop` e liberar **somente** o necessário (DNS 53/UDP+TCP) para as redes do provedor, além de integrar o banimento do Fail2Ban via `sets`.

**O que fazer agora:** aplique o arquivo abaixo, ajuste os `sets` com seus prefixos reais e valide o ruleset antes de seguir.

### 10.1 Arquivo completo `/etc/nftables.conf`

> Ajuste as redes em `set allowed_dns_v4/v6`.

```nft
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
  ##########################################################################
  # Sets
  ##########################################################################
  set allowed_dns_v4 {
    type ipv4_addr
    flags interval
    elements = {
      127.0.0.0/8,
      192.0.2.0/24,         # gerência (exemplo)
      100.64.0.0/10         # clientes (exemplo)
    }
  }

  set allowed_dns_v6 {
    type ipv6_addr
    flags interval
    elements = {
      ::1/128,
      2001:db8:200::/48,    # gerência (exemplo)
      2001:db8:100::/48     # clientes (exemplo)
    }
  }

  # IPs banidos pelo Fail2Ban (Unbound)
  set f2b-unbound4 {
    type ipv4_addr;
    flags timeout;
    timeout 1h;
  }

  set f2b-unbound6 {
    type ipv6_addr;
    flags timeout;
    timeout 1h;
  }

  ##########################################################################
  # Chains
  ##########################################################################
  chain input {
    type filter hook input priority 0; policy drop;

    # Loopback
    iif "lo" accept

    # Conexões já estabelecidas
    ct state established,related accept

    # Drop imediato para IPs banidos
    ip saddr @f2b-unbound4 drop
    ip6 saddr @f2b-unbound6 drop

    # ICMP/ICMPv6 (essencial para IPv6)
    ip saddr @allowed_dns_v4 icmp type echo-request counter accept
	ip6 saddr @allowed_dns_v6 icmpv6 type { echo-request, nd-neighbor-solicit, nd-neighbor-advert, nd-router-solicit, nd-router-advert, mld-listener-query } counter accept

    # DNS (UDP/TCP 53) apenas das redes permitidas
    ip saddr @allowed_dns_v4 udp dport 53 accept
    ip saddr @allowed_dns_v4 tcp dport 53 accept

    ip6 saddr @allowed_dns_v6 udp dport 53 accept
    ip6 saddr @allowed_dns_v6 tcp dport 53 accept

    # (Opcional) SSH, restrinja ao seu IP/VPN antes de habilitar em produção
    # tcp dport 22 accept
  }
}
```

> **Atenção (SSH):** se você está conectado via SSH, ajuste a regra de SSH **antes** de aplicar, ou use console/iDRAC/VM console para não se trancar.  
> Exemplo (substitua pela sua rede de gerência/VPN): descomente e restrinja `tcp dport 22` para `ip saddr 192.0.2.0/24`.

Valide a sintaxe sem aplicar

```bash
sudo nft -c -f /etc/nftables.conf
```

se estiver tudo correto, aplicar:

```bash
sudo systemctl enable --now nftables
sudo nft -f /etc/nftables.conf
sudo nft list ruleset
```

### 10.2 Validação dos sets do Fail2Ban (timeout)

O Fail2Ban insere IPs banidos nos sets `f2b-unbound4` e `f2b-unbound6` usando **timeout** (expiração automática).  
Para isso funcionar, os sets **precisam** ter `flags timeout;` (como no exemplo acima).

> Sem `flags timeout;`, comandos do tipo `nft add element ... timeout 15m` falham e o IP pode não entrar no set.  
> Em alguns cenários, o Fail2Ban ainda pode mostrar o IP como “banido” na jail (estado interno), mesmo que o nftables não tenha aplicado o elemento — por isso valide no set.

#### 10.2.1 Teste manual (nftables)

1) Adicione um IP de teste com timeout (IPv4):

```bash
sudo nft add element inet filter f2b-unbound4 { 203.0.113.10 timeout 10m }
sudo nft list set inet filter f2b-unbound4 | grep -F 203.0.113.10
```

> Dica: você pode testar em segundos também (mesmo formato que o Fail2Ban costuma usar internamente):  
> `sudo nft add element inet filter f2b-unbound4 { 203.0.113.10 timeout 900s }`

2) Remova o IP de teste:

```bash
sudo nft delete element inet filter f2b-unbound4 { 203.0.113.10 }
```

> Se o `add element` retornar erro, revise a definição do set (seção 10.1) e reaplique `/etc/nftables.conf`.

#### 10.2.2 Teste com Fail2Ban (banip)

1) Faça um ban de teste na jail (exemplo com RPZ):

```bash
sudo fail2ban-client set unbound-rpz-seguranca banip 203.0.113.10
```

2) Verifique se o IP entrou no set:

```bash
sudo nft list set inet filter f2b-unbound4 | grep -F 203.0.113.10
```

3) Se o IP aparecer como banido no Fail2Ban, mas não aparecer no set, procure erro na action:

```bash
sudo tail -n 200 /var/log/fail2ban.log
sudo journalctl -u fail2ban -n 200 --no-pager
```

> Se você ver erro do tipo `expecting string` apontando para `timeout 900`, significa que o timeout foi passado **sem unidade**.  
> Corrija a action para usar `timeout <bantime>s` (seção 11.3) e repita o teste.

4) Limpeza:

```bash
sudo fail2ban-client set unbound-rpz-seguranca unbanip 203.0.113.10
```

---

<a id="11"></a>
## 11. Fail2Ban + nftables set, proteção anti-abuso

### 11.1 Por que Fail2Ban em DNS recursivo?

Mesmo fechado para redes do provedor, um recursor pode sofrer abuso de:

- hosts infectados (botnets) gerando QPS alto
- scanning de NXDOMAIN (DGA / brute force de domínios)
- tentativa de uso “externo” (quando ACL/firewall está errado)

O Fail2Ban **não bloqueia conteúdo**. Ele **bane IPs abusivos** no servidor DNS.

### 11.2 Preparar logs para os filtros

> IMPORTANTE: `log-queries`/`log-replies` é **pesado** em recursor de ISP.  
> Use **apenas para calibração temporária** (ex.: **24–48h**) e depois desative.

As jails **[unbound-qps]** e **[unbound-nxdomain]** dependem desses logs por query/reply.  
Para não “poluir” a configuração principal, a recomendação é habilitar isso via um arquivo dedicado em `unbound.conf.d` fica simples ligar/desligar.

Crie `/etc/unbound/unbound.conf.d/55-fail2ban-query-logs.conf`:

```bash
sudo tee /etc/unbound/unbound.conf.d/55-fail2ban-query-logs.conf >/dev/null <<'EOF'
server:
    ########################################################################
    # Logs por query/reply (APENAS para calibração do Fail2Ban)
    #
    # Isso aumenta MUITO o volume de log e pode impactar desempenho em alto QPS.
    # Use por janela curta (ex.: 24–48h), calibre filtros e depois desative.
    ########################################################################
    verbosity: 2
    log-tag-queryreply: yes
    log-queries: yes
    log-replies: yes
EOF
```

Depois:

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
```

Em produção, após calibrar (24–48h):
- **desative** esse arquivo para reduzir volume de log, e
- volte as jails **[unbound-qps]** e **[unbound-nxdomain]** para `enabled = false` (elas não fazem sentido sem esses logs).

Para desativar rapidamente:

```bash
TS="$(date +%F_%H%M%S)"
sudo mv /etc/unbound/unbound.conf.d/55-fail2ban-query-logs.conf "/etc/unbound/unbound.conf.d/55-fail2ban-query-logs.conf.off.${TS}"
sudo unbound-control reload
```

### 11.3 Action customizada: `/etc/fail2ban/action.d/nftables-unbound-set.conf`

> Essa action **não cria** tabela/chain no nftables. Ela só **insere/remove IPs** nos sets `f2b-unbound4` e `f2b-unbound6` já definidos em `/etc/nftables.conf`.

```bash
sudo tee /etc/fail2ban/action.d/nftables-unbound-set.conf >/dev/null <<'EOF'
[Definition]
actionstart =
actionstop =
actioncheck = nft list set inet filter <addr_set> >/dev/null 2>&1
# IMPORTANTE:
# - O Fail2Ban converte `bantime` para SEGUNDOS internamente.
# - A variável `<bantime>` chega aqui como número (ex.: 900).
# - No nftables, o `timeout` precisa de UNIDADE (ex.: 900s, 15m, 1h).
actionban = nft add element inet filter <addr_set> { <ip> timeout <bantime>s }
actionunban = nft delete element inet filter <addr_set> { <ip> }

[Init]
addr_set = f2b-unbound4

[Init?family=inet6]
addr_set = f2b-unbound6
EOF
```

> **Nota importante (timeout no nftables):** o `bantime` do Fail2Ban vira `timeout` no `nft add element ... timeout ...`.  
> Na prática, o Fail2Ban passa `<bantime>` como **segundos** (inteiro). Já o nftables exige **unidade** no timeout (ex.: `900s`, `15m`, `1h`).  
> Por isso a action usa `timeout <bantime>s`. Mesmo assim, **valide** no seu ambiente (seção 10.2): faça um `banip` e confira o set com `nft list set ...`.

### 11.4 Filtros (exemplos prontos)

#### 11.4.1 QPS (exige `log-queries`)

Crie `/etc/fail2ban/filter.d/unbound-qps.conf`:

```bash
sudo tee /etc/fail2ban/filter.d/unbound-qps.conf >/dev/null <<'EOF'
[Definition]
# Nota (didático): em filtros do Fail2Ban, várias regex são declaradas na MESMA opção
# `failregex` usando múltiplas linhas indentadas.
#
# NÃO repita `failregex =` em linhas separadas; isso pode quebrar o filtro com erro do tipo:
# "option 'failregex' in section 'Definition' already exists".
#
# Formatos que este filtro cobre (varia conforme versão/opções de log do Unbound):
# - ... <IP>@<PORT> query: ...
# - ... <IP> query: ...
# - ... query: <IP>@<PORT> ...
# - ... query: <IP> ...
failregex = ^.*<HOST>(?:@\d+)?(?::)?\s+query:\s+.*$
            ^.*\bquery:\s+<HOST>(?:@\d+)?\s+.*$
ignoreregex =
EOF
```

#### 11.4.2 NXDOMAIN (exige `log-replies`)

Crie `/etc/fail2ban/filter.d/unbound-nxdomain.conf`:

```bash
sudo tee /etc/fail2ban/filter.d/unbound-nxdomain.conf >/dev/null <<'EOF'
[Definition]
# Formatos que este filtro cobre (varia conforme versão/opções de log do Unbound):
# - ... <IP>@<PORT> reply: ... NXDOMAIN ...
# - ... <IP> reply: ... NXDOMAIN ...
# - ... reply: <IP>@<PORT> ... NXDOMAIN ...
# - ... reply: <IP> ... NXDOMAIN ...
failregex = ^.*<HOST>(?:@\d+)?(?::)?\s+reply:\s+.*\bNXDOMAIN\b.*$
            ^.*\breply:\s+<HOST>(?:@\d+)?\s+.*\bNXDOMAIN\b.*$
ignoreregex =
EOF
```

Dica prática: para testar filtros sem rodar `fail2ban-regex` em logs gigantes, você pode validar contra um recorte do log:

```bash
sudo tail -n 50000 /var/log/unbound/unbound.log > /tmp/unbound.tail.log
sudo fail2ban-regex /tmp/unbound.tail.log /etc/fail2ban/filter.d/unbound-qps.conf
sudo fail2ban-regex /tmp/unbound.tail.log /etc/fail2ban/filter.d/unbound-nxdomain.conf
```

#### 11.4.3 Tentativa externa / recusas depende do log aparecer no seu nível de verbosity

Crie `/etc/fail2ban/filter.d/unbound-refused.conf`:

```bash
sudo tee /etc/fail2ban/filter.d/unbound-refused.conf >/dev/null <<'EOF'
[Definition]
# Exemplos típicos (podem variar):
# ... refused query from ip4 198.51.100.200 port 12345 ...
# ... refused query from ip6 2001:db8::200 port 12345 ...
failregex = ^.*refused query from (?:ip[46]\s+)?<HOST>\s+port\s+\d+.*$
ignoreregex =
EOF
```

#### 11.4.4 RPZ hits (segurança), recomendado (log leve)

Crie `/etc/fail2ban/filter.d/unbound-rpz-seguranca.conf`:

```bash
sudo tee /etc/fail2ban/filter.d/unbound-rpz-seguranca.conf >/dev/null <<'EOF'
[Definition]
# Exemplo real (formato típico do Unbound):
# info: rpz: applied [rpz-seguranca] www.exemplo.com. A IN CNAME . 100.64.1.10@56412 www.exemplo.com. A IN
#
# Compatibilidade extra:
# - alguns ambientes usam `rpz-log-name` diferente (ex.: o nome da zona)
# - alguns formatos podem incluir `rpz-name=...`
#
# Nota: em geral, o `rpz-log-name` é o caminho mais estável (ex.: "rpz-seguranca").
# Mesmo assim, deixamos compatibilidade com o nome da zona RPZ, caso seu log venha assim.
failregex = ^.*\brpz:\s+applied\s+(?:\[(?:rpz-seguranca|rpz\.seguranca\.(?:exemplo|bloqueio))\]|rpz-name=(?:rpz-seguranca|rpz\.seguranca\.(?:exemplo|bloqueio))).*\s<HOST>(?:[@#]\d+)?\s.*$
ignoreregex =
EOF
```

### 11.5 Jails (arquivo único)

> **Calibração obrigatória ANTES de habilitar qualquer jail:**  
> 1) garanta que o log real contém o padrão que seu filtro espera (ex.: RPZ hits com `rpz-log: yes`),  
> 2) rode `fail2ban-regex` contra o **log real**,  
> 3) só então habilite a(s) jail(s).  
>
> Isso reduz muito risco de falso positivo em produção (e também evita “jail ativa mas nunca casa regex”).

Crie `/etc/fail2ban/jail.d/unbound.local`:

```bash
sudo tee /etc/fail2ban/jail.d/unbound.local >/dev/null <<'EOF'
[DEFAULT]
backend = auto

# IMPORTANTE:
# - NÃO repita "ignoreip" em duas linhas (Falha de config).
# - Coloque aqui SOMENTE loopback + infra crítica.
# - O que entra (ou não) aqui depende de como o DNS chega no recursor (ver seção 1.2 – CGNAT e origem do IP).
ignoreip = 127.0.0.1/8 ::1 198.51.100.10/32 198.51.100.11/32 198.51.100.1/32

# Comece conservador e ajuste após observar 2–4 semanas
findtime = 1m
bantime  = 15m

# (Opcional) Escalonamento automático de bans:
# - cada reincidência aumenta o tempo de ban
# - útil para clientes infectados que insistem em abuso
bantime.increment = true
bantime.factor    = 2
bantime.maxtime   = 24h

# Recomendação operacional:
# Antes de habilitar qualquer jail em produção, valide o regex contra o seu log real:
# sudo fail2ban-regex /var/log/unbound/unbound.log /etc/fail2ban/filter.d/unbound-rpz-seguranca.conf
# Se der 0 matched, ajuste o filtro (formato de log pode variar entre versões).

[unbound-rpz-seguranca]
enabled  = false
filter   = unbound-rpz-seguranca
logpath  = /var/log/unbound/unbound.log
maxretry = 30
action   = nftables-unbound-set

[unbound-qps]
enabled  = false
filter   = unbound-qps
logpath  = /var/log/unbound/unbound.log
maxretry = 300
action   = nftables-unbound-set

[unbound-nxdomain]
enabled  = false
filter   = unbound-nxdomain
logpath  = /var/log/unbound/unbound.log
maxretry = 150
action   = nftables-unbound-set

[unbound-refused]
enabled  = false
filter   = unbound-refused
logpath  = /var/log/unbound/unbound.log
maxretry = 10
action   = nftables-unbound-set
EOF
```

### 11.6 Como habilitar as jails (passo a passo)

Este passo a passo deixa explícito **onde** habilitar e **como** validar antes, para evitar:
- jail habilitada mas “não faz nada” (regex não casa com seu log real);
- jails de QPS/NXDOMAIN habilitadas sem controle (logs pesados ligados sem necessidade).

#### 11.6.1 Onde fica o `enabled = false` (e o que trocar)

- Arquivo: `/etc/fail2ban/jail.d/unbound.local`
- As jails ficam no final do arquivo, em blocos `[unbound-...]`.
- Para habilitar uma jail, troque `enabled  = false` por `enabled  = true` **dentro do bloco da jail desejada**.

Edite o arquivo:

```bash
sudo nano /etc/fail2ban/jail.d/unbound.local
```

#### 11.6.2 Validação obrigatória antes de habilitar

Antes de habilitar qualquer jail, valide o filtro contra o **log real**:

```bash
sudo fail2ban-regex /var/log/unbound/unbound.log /etc/fail2ban/filter.d/unbound-rpz-seguranca.conf
```

Dica prática: se seu `unbound.log` é muito grande, extraia **apenas** as linhas de RPZ para testar o filtro em segundos (em vez de horas):

```bash
sudo grep -E 'rpz: applied' /var/log/unbound/unbound.log > /tmp/unbound-rpz-applied.log
sudo fail2ban-regex /tmp/unbound-rpz-applied.log /etc/fail2ban/filter.d/unbound-rpz-seguranca.conf
```

Como decidir:
- se der **0 matched**, **não habilite** a jail ainda; ajuste a regex (formato do log pode variar);
- se der `matched > 0`, o filtro está funcionando no seu ambiente.

#### 11.6.3 Produção contínua (log leve): habilitar RPZ (recomendado)

No bloco `[unbound-rpz-seguranca]`, troque:

```ini
enabled  = false
```

por:

```ini
enabled  = true
```

#### 11.6.4 Calibração temporária (log pesado): QPS e NXDOMAIN

As jails `[unbound-qps]` e `[unbound-nxdomain]` **só fazem sentido** se você também habilitou o passo **11.2** (arquivo `55-fail2ban-query-logs.conf` com `log-queries/log-replies`).

Para calibrar por 24–48h, habilite (troque `enabled  = false` por `enabled  = true`):

```ini
[unbound-qps]
enabled  = true

[unbound-nxdomain]
enabled  = true
```

Depois de calibrar:
- desative o arquivo `55-fail2ban-query-logs.conf` (passo 11.2), e
- volte `[unbound-qps]` e `[unbound-nxdomain]` para `enabled  = false`.

#### 11.6.5 Opcional: `[unbound-refused]` depende do log aparecer

Antes de habilitar `[unbound-refused]`, valide o filtro contra o log real:

```bash
sudo fail2ban-regex /var/log/unbound/unbound.log /etc/fail2ban/filter.d/unbound-refused.conf
```

Se der **0 matched**, não habilite ou revise observabilidade de forma controlada e temporária, se fizer sentido.

Se o filtro casar com seu log e você quiser habilitar:

```ini
[unbound-refused]
enabled  = true
```

#### 11.6.6 Aplicar mudanças e validar

1) Reinicie o Unbound **apenas se** você habilitou/alterou o passo **11.2**:

```bash
sudo systemctl restart unbound
```

2) Reinicie o Fail2Ban sempre, após mudar `unbound.local` e valide o status:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
```

3) Verifique as jails que você habilitou:

```bash
sudo fail2ban-client status unbound-rpz-seguranca
sudo fail2ban-client status unbound-qps
sudo fail2ban-client status unbound-nxdomain
sudo fail2ban-client status unbound-refused
```

4) Verifique os sets no nftables onde o Fail2Ban insere/remove IPs:

```bash
sudo nft list set inet filter f2b-unbound4
sudo nft list set inet filter f2b-unbound6
```

5) Teste controlado, confirma integração Fail2Ban -> nftables e valida timeout no seu ambiente:

```bash
# Banir um IP de teste (use um IP de laboratório/exemplo, não um cliente real)
sudo fail2ban-client set unbound-rpz-seguranca banip 203.0.113.10

# Verificar se entrou no set (deve aparecer o IP e um timeout/expiração)
sudo nft list set inet filter f2b-unbound4 | grep -F 203.0.113.10

# Desbanir
sudo fail2ban-client set unbound-rpz-seguranca unbanip 203.0.113.10
```

### 11.7 Logs e eventos (ban/unban)

Ver logs do Fail2Ban (ban/unban):

```bash
# Se existir log em arquivo (comum no Debian)
sudo tail -f /var/log/fail2ban.log

# Sempre funciona via systemd/journald
sudo journalctl -u fail2ban -f
```

Ver logs do Unbound (inclui RPZ hits se `rpz-log: yes`):

```bash
sudo tail -f /var/log/unbound/unbound.log
sudo journalctl -u unbound -f
```

### 11.8 Como testar sem “quebrar produção”

1) Repita a validação do filtro (sem banir) do passo **11.6.2**.

> Se aparecer **0 matched**, isso pode ser perfeitamente normal quando:
> - o servidor ainda não está recebendo consultas reais (ambiente não-prod);
> - ou `rpz-log: yes` ainda não está habilitado;
> - ou você ainda está em modo sombra e testando pouco.
>
> Para “gerar log”, faça algumas consultas de teste (seção 12) e rode o comando novamente.

2) Simule um bloqueio RPZ usando o domínio de teste:

```bash
dig @127.0.0.1 teste-malware.exemplo A +short
```

3) Desbanir manualmente:

```bash
sudo fail2ban-client set unbound-rpz-seguranca unbanip 100.64.1.10
```

---

<a id="12"></a>
## 12. Subir serviços e validação

### 12.1 Abrir interfaces e subir serviços

Até aqui, o Unbound foi mantido **restrito ao loopback** (arquivo `00-server.conf`).  
Isso reduz risco de exposição enquanto você monta firewall/Fail2Ban/RPZ.

Agora, com firewall e ACL já preparados, é o momento de **abrir as interfaces** para atender suas redes do ISP.

1) Ajuste as interfaces em `/etc/unbound/unbound.conf.d/00-server.conf` (EXEMPLO):

Backup (recomendado): use o padrão do passo **5.2.2** (arquivo único), apontando para `/etc/unbound/unbound.conf.d/00-server.conf`.

Edite:

```bash
sudo nano /etc/unbound/unbound.conf.d/00-server.conf
```

Troque:

```conf
    interface: 127.0.0.1
    interface: ::1
```

por (ou adicione) algo como:

```conf
    interface: 0.0.0.0
    interface: ::0
```

> Se você tem IPs bem definidos (anycast/VRRP/loopback), prefira bindar neles em vez de `0.0.0.0/::0`.

2) Valide e reinicie:

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
sudo ss -lntup | grep -E '(^Netid|:53\b)'
```

> Nota (Debian): se no `systemctl status unbound` aparecer um aviso sobre `DAEMON_OPTS` vazio, isso costuma ser normal.  
> Opcionalmente, crie `/etc/default/unbound` definindo `DAEMON_OPTS` como string vazia para remover o aviso:
> ```bash
> sudo tee /etc/default/unbound >/dev/null <<'EOF'
> # Opções adicionais para o serviço systemd do Unbound.
> # Deixe vazio para manter o ExecStart do pacote e apenas silenciar o warning.
> DAEMON_OPTS=""
> EOF
> sudo systemctl restart unbound
> ```

3) Garanta serviços no boot:

```bash
sudo systemctl enable --now unbound
sudo systemctl enable --now nftables
sudo systemctl enable --now fail2ban
```

### 12.2 Operação básica com `unbound-control`

```bash
sudo unbound-control status
sudo unbound-control stats_noreset | head
```

#### 12.2.1 Verificação de Cache, Performance e Saúde do Unbound

Esta seção é o “painel de instrumentos” do recursor: cache, volume, latência e sinais de abuso.

##### `stats` vs `stats_noreset` diferença prática

- `unbound-control stats` imprime estatísticas **e zera os contadores** útil para medir por janela.
- `unbound-control stats_noreset` imprime estatísticas **sem zerar** útil para acompanhar acumulado.

Em produção, a regra prática é:
- use `stats_noreset` para observabilidade contínua (“modo NOC”);
- use `stats` quando você quer medir um intervalo específico (ex.: 60s) sem precisar fazer conta manual.

Ver contadores principais (cache e volume):

```bash
sudo unbound-control stats_noreset \
  | egrep '^(total\.num\.queries|total\.num\.cachehits|total\.num\.cachemiss|total\.num\.prefetch|total\.num\.recursivereplies|total\.num\.queries_ip_ratelimited|total\.recursion\.time\.avg)='
```

Monitorar ao vivo (acumulado):

```bash
watch -n 2 -- 'sudo unbound-control stats_noreset | egrep "^(total\\.num\\.(queries|cachehits|cachemiss|prefetch|recursivereplies|queries_ip_ratelimited)|total\\.recursion\\.time\\.avg)="'
```

QPS aproximado (amostragem a cada 1s, usando `stats_noreset`):

```bash
sudo bash -c '
prev=$(unbound-control stats_noreset | sed -n "s/^total\\.num\\.queries=//p" | head -n 1)
while sleep 1; do
  now=$(unbound-control stats_noreset | sed -n "s/^total\\.num\\.queries=//p" | head -n 1)
  printf "QPS=%d total=%d\n" "$((now-prev))" "$now"
  prev=$now
done
'
```

> Se algum contador não existir na sua versão, rode `unbound-control stats_noreset | head -n 80` e ajuste os nomes do `awk/egrep`.

##### Interpretação prática (campos principais)

- `total.num.cachehits` / `total.num.cachemiss`  
  - leitura prática: hit alto = “cache trabalhando”; miss alto = mais recursão/latência/CPU.
  - dica: em recursor saudável, hits costumam superar miss em horário normal (depende do perfil).

- `total.num.prefetch`  
  - indica quantas vezes o Unbound “renovou” itens do cache antes de expirar (cache mais “quente”).

- `total.num.queries_ip_ratelimited`  
  - indica quantas queries foram limitadas por `ip-ratelimit`.
  - se subir em horário normal, `ip-ratelimit` pode estar baixo demais (ou o Unbound não enxerga IP real do cliente).

- `total.recursion.time.avg`  
  - latência média da recursão (o que o Unbound gastou “indo pra internet” e validando).
  - se aumentar muito: suspeite de upstream lento, perda/jitter, DNSSEC pesado, ou saturação de rede.

##### Teste prático de cache (com `dig +ttlunits`)

Faça duas consultas iguais e compare **Query time** e **TTL** (o TTL costuma diminuir na segunda consulta, indicando cache):

```bash
dig @127.0.0.1 cloudflare.com A +ttlunits +noall +answer +stats
dig @127.0.0.1 cloudflare.com A +ttlunits +noall +answer +stats
```

##### Inspecionar cache (com `unbound-control lookup`)

`lookup` mostra o que o Unbound tem no cache para um nome. É útil para operação (“isso está em cache?”):

```bash
sudo unbound-control lookup cloudflare.com
sudo unbound-control lookup cloudflare.com AAAA
```

> Se não retornar nada, pode ser simplesmente porque o nome ainda não foi consultado, expirou, ou está sendo respondido de outra forma (CNAME, etc).

##### Alertas operacionais

- Não deixe `log-queries`/`log-replies` ligado permanentemente (gera log gigante e pode impactar performance).  
  Use apenas para calibração temporária (seção **11.2**).

- `ip-ratelimit` muito baixo (1–10) causa falso positivo e dropa clientes normais em ISP.  
  Ponto de partida realista para ISP médio (quando o Unbound enxerga IP real do cliente): **100** (faixa 50–200).  
  Se o tráfego chega mascarado (CGNAT/BNG/NAT concentrando muitos clientes em 1 IP), você precisa subir o valor ou manter desativado (ver seção **1.2**).

- Logs do tipo `notice: ip_ratelimit exceeded` não são erro: indicam burst legítimo acima do limite naquele segundo.  
  Isso não “bane” cliente permanentemente; o excesso é dropado naquele instante.

#### 12.2.2 Ajuste de cache por métricas

- Se `total.num.cachemiss` estiver alto **e** houver RAM sobrando, aumente `msg-cache-size`/`rrset-cache-size` **aos poucos** e acompanhe `stats_noreset`.  
- Se houver pressão de memória (swap/OOM), reduza caches (comece por `msg-cache-size` e `rrset-cache-size`) e reavalie.  
- Valide sempre com `unbound-control stats_noreset` (hit/miss, volume) antes/depois da mudança.

### 12.3 Testes de resolução (IPv4/IPv6)

Resolução normal:

```bash
dig @127.0.0.1 google.com A +short
dig @127.0.0.1 google.com AAAA +short
dig @::1 google.com A +short
dig @::1 google.com AAAA +short
```

DNSSEC:

```bash
dig @127.0.0.1 sigok.verteiltesysteme.net A +dnssec +multi
dig @127.0.0.1 sigfail.verteiltesysteme.net A +dnssec +multi
```

Bloqueio RPZ:

```bash
dig @127.0.0.1 teste-malware.exemplo A
```

Interpretação:
- em **modo sombra** (`rpz-action-override: disabled`), o domínio **pode resolver** (mas o log deve registrar “rpz: applied”);
- em **modo bloqueio** (`rpz-action-override: nxdomain`), deve retornar **NXDOMAIN**.

### 12.4 “Externo não resolve” validação do fechamento

De um host fora das suas redes permitidas, a consulta deve:

- ser bloqueada pelo firewall (timeout), ou
- ser recusada (REFUSED), dependendo de onde a requisição foi bloqueada.

Exemplo de fora:

```bash
dig @IP_PUBLICO_DO_SEU_RECURSOR google.com A +time=1 +tries=1
```

Interpretação:
- **timeout** (sem resposta): firewall drop (comportamento esperado);
- **REFUSED**: a consulta chegou no Unbound, mas foi recusada pela ACL (ainda aceitável, mas preferível bloquear no firewall);
- **NOERROR**: algo está errado (resolver virou público).

Dica de diagnóstico no servidor:

```bash
sudo nft list ruleset | sed -n '1,200p'
sudo journalctl -u unbound -n 200 --no-pager
```

### 12.5 Testes básicos de performance (sem “derrubar” nada)

> Objetivo: ter um número inicial de QPS/latência e comparar “cache frio” vs “cache quente”.
>
> **Não rode isso em horário de pico** e não comece com QPS absurdo. Em ISP, faça em janela.

1) Instale ferramentas:

```bash
sudo apt -y install dnsperf
```

2) Crie um arquivo de consultas (exemplo pequeno):

```bash
cat > /tmp/dnsperf-queries.txt <<'EOF'
google.com A
google.com AAAA
cloudflare.com A
debian.org A
EOF
```

3) Rode por 20s a 2.000 QPS (ajuste):

```bash
dnsperf -s 127.0.0.1 -d /tmp/dnsperf-queries.txt -l 20 -Q 2000
```

4) Repita uma segunda vez (cache quente) e compare:
- taxa de perdas (`lost`),
- latência média/p95,
- throughput.

5) Correlacione com stats do Unbound:

```bash
sudo unbound-control stats_noreset | egrep 'total\.num\.queries|total\.num\.cachehits|total\.num\.cachemiss'
```

### 12.6 Fazer o próprio servidor usar o Unbound local sem “quebrar” o `resolv.conf`

Usar o próprio Unbound local como resolvedor do sistema ajuda a:
- validar que o serviço está funcional,
- aquecer cache,
- reduzir dependência de DNS externo em troubleshooting.

**Risco:** se o Unbound parar, o servidor pode perder resolução de nomes (SSH por IP continua).

Opção A (mais simples, quando `/etc/resolv.conf` é estático):

```bash
sudo cp -a /etc/resolv.conf /etc/resolv.conf.bak.$(date +%F)
sudo tee /etc/resolv.conf >/dev/null <<'EOF'
nameserver 127.0.0.1
nameserver ::1
options timeout:1 attempts:2
EOF
```

Opção B: use fallback para o segundo recursor do ISP

```conf
nameserver 127.0.0.1
nameserver ::1
nameserver 192.0.2.53
nameserver 2001:db8:200::53
```

> Se o seu ambiente usa `systemd-resolved`/NetworkManager, **não** edite `/etc/resolv.conf` na mão. Configure via a ferramenta do seu stack (ou mantenha a resolução do servidor apontando para outro resolver de infraestrutura).

Valide:

```bash
dig @127.0.0.1 deb.debian.org A +short
host deb.debian.org
```

### 12.7 Compliance neutralidade, segurança e auditoria, do jeito “defensável”

Boas práticas para comunicar e operar o **RPZ de segurança** sem virar “censura acidental”:

1) **Escopo estrito e público**  
   - bloquear apenas *phishing/malware/botnet/C2*;
   - evitar listas de ads/tracking/categorias.

2) **Processo de exceção (allowlist) e rollback**  
   - canal interno (NOC/SOC) para liberar falsos positivos;
   - rollback rápido (seção 13) e registro do que foi revertido.

3) **Auditoria mínima** (sem logar “tudo”)  
   - log de update (timestamp, quantidade de domínios, hash SHA256);
   - log de RPZ aplicado (quando `rpz-log: yes`);
   - retenção curta e controlada (logrotate).

4) **Transparência técnica**  
   - preferir NXDOMAIN (comportamento “natural” do DNS);
   - documentar que é uma medida de segurança (não inspeção de conteúdo).

> **Nota Brasil:** neutralidade da rede e exceções por segurança/mitigação são tema regulatório. Trate isso com seu jurídico e mantenha documentação de processo/escopo.

<a id="13"></a>
## 13. Rollback rápido, quando der errado

### 13.1 Desligar o bloqueio sem parar o DNS

Opção A (mais segura): modo sombra

1) Edite `/etc/unbound/unbound.conf.d/60-rpz-seguranca.conf` e troque:
   - `rpz-action-override: nxdomain` → `rpz-action-override: disabled`
2) Recarregue:

```bash
sudo unbound-control reload
```

Opção B: desativar a política RPZ (corta RPZ inteiro)

```bash
sudo mv /etc/unbound/unbound.conf.d/60-rpz-seguranca.conf /etc/unbound/unbound.conf.d/60-rpz-seguranca.conf.off
sudo unbound-control reload
```

### 13.2 Rollback da última lista (zonefile)

```bash
sudo cp -av /var/lib/unbound/rpz/rpz.allow.seguranca.exemplo.zone.bak /var/lib/unbound/rpz/rpz.allow.seguranca.exemplo.zone \
  || echo "INFO: backup allowlist não existe (rpz.allow.seguranca.exemplo.zone.bak)."
sudo cp -av /var/lib/unbound/rpz/rpz.seguranca.exemplo.zone.bak /var/lib/unbound/rpz/rpz.seguranca.exemplo.zone \
  || echo "INFO: backup blocklist não existe (rpz.seguranca.exemplo.zone.bak)."
sudo unbound-control reload
```

### 13.3 Desligar o Fail2Ban sem “abrir” o DNS (contingência)

Se você suspeita de falso positivo em massa (ex.: banindo infraestrutura/CGNAT), pare aqui e revise a seção **1.2** (CGNAT e origem do IP). Se precisar parar o Fail2Ban rápido:

```bash
sudo systemctl disable --now fail2ban
```

Isso **não abre** o resolver, porque:
- o fechamento principal é nftables + ACL do Unbound.

Se você quiser também limpar bans que já entraram no nftables:

```bash
sudo nft flush set inet filter f2b-unbound4
sudo nft flush set inet filter f2b-unbound6
```

---

<a id="14"></a>
## 14. Checklist pós-implantação (produção)

- [ ] Unbound ativo: `systemctl status unbound`
- [ ] Firewall ativo e default drop: `nft list ruleset`
- [ ] Resolver fechado (teste externo falha)
- [ ] DNSSEC validando (sigok ok, sigfail SERVFAIL)
- [ ] RPZ de segurança aplicando (NXDOMAIN nos testes)
- [ ] Timer de atualização RPZ funcionando: `systemctl list-timers | grep unbound-rpz`
- [ ] Fail2Ban ativo e com `unbound-rpz-seguranca` habilitada (após calibração)
- [ ] `ignoreip` revisado (sem banir BRAS/infra)

Comandos de validação (rápidos e objetivos):

```bash
# DNSSEC: deve validar (resposta normal)
dig @127.0.0.1 dnssec.works A +dnssec +noall +answer +comments

# DNSSEC: deve falhar (SERVFAIL) quando a validação está ativa
dig @127.0.0.1 dnssec-failed.org A +dnssec +noall +answer +comments

# RPZ (domínio de teste da blocklist do guia): deve resultar em NXDOMAIN quando o bloqueio está ativo
dig @127.0.0.1 teste-malware.exemplo A +noall +answer +comments
```

---

<a id="15"></a>
## 15. Referências (fontes e termos de uso)

### Unbound (documentação oficial / manpages)
- NLnet Labs, Documentação do Unbound: https://nlnetlabs.nl/documentation/unbound/
- Unbound Docs, RPZ (conceitos, triggers, ações, ordem, DISABLED/PASSTHRU): https://unbound.docs.nlnetlabs.nl/en/latest/topics/filtering/rpz.html
- Debian Manpages (testing/trixie), `unbound.conf(5)` (opções de cache, DNSSEC, RPZ, logging, ip-ratelimit): https://manpages.debian.org/testing/unbound/unbound.conf.5.en.html
- Debian (pacote `unbound`): https://packages.debian.org/trixie/unbound

### RFCs / padrões (base conceitual)
- RFC 9156, QNAME Minimization in DNS: https://www.rfc-editor.org/rfc/rfc9156
- RFC 4033, DNSSEC (introdução e requisitos): https://www.rfc-editor.org/rfc/rfc4033
- RFC 4034, DNSSEC (RRs e formatos): https://www.rfc-editor.org/rfc/rfc4034
- RFC 4035, DNSSEC (protocolo): https://www.rfc-editor.org/rfc/rfc4035

### Root hints
- Internic (IANA), root hints (`named.cache`): https://www.internic.net/domain/named.cache

### systemd timers (automação sem cron)
- systemd.timer(5), manual oficial: https://man7.org/linux/man-pages/man5/systemd.timer.5.html
- systemd.guru, exemplos de `OnCalendar` (inclui `*/6:00:00`): https://systemd.guru/systemd-timer-calendar-events/

### Fail2Ban + nftables
- Debian Sources, action `nftables.conf` (referência de tags/Init por família): https://sources.debian.org/src/fail2ban/1.0.2-2/config/action.d/nftables.conf/
- Fail2Ban, documentação oficial: https://www.fail2ban.org/

### Tutoriais deste repositório (complementares)
- NTP/Chrony: **[Guia de Produção: Servidor NTP Interno com Chrony no Debian 13](../sistema/guia_producao_ntp_chrony_debian13.md)**

### Brasil (conscientização e boas práticas)
- Internet Segura (NIC.br) — material educativo: https://internetsegura.br/
- Cartilha de Segurança para Internet (CERT.br): https://cartilha.cert.br/

> **Nota:** o portal Internet Segura é **educativo** e **não** fornece feed/lista RPZ de domínios.

### Feeds (RPZ de segurança) e termos de uso (LEIA antes de uso em ISP)
- URLhaus (abuse.ch), projeto e downloads: https://urlhaus.abuse.ch/about/
- abuse.ch / Spamhaus Technology, Terms of Use: https://abuse.ch/terms-of-use/
- PhishTank, developer info (dumps públicos e limites): https://phishtank.org/developer_info.php
- PhishTank, dump público (CSV): http://data.phishtank.com/data/online-valid.csv
- PhishTank, dump público (JSON): http://data.phishtank.com/data/online-valid.json
- PhishTank, legal / termos: https://www.phishtank.com/legal.php
- Spamhaus DROP/EDROP, página e uso/fair use: https://www.spamhaus.org/drop/drop/

### Neutralidade / compliance (Brasil), referência normativa
- Lei nº 12.965/2014 (Marco Civil da Internet): https://www.planalto.gov.br/ccivil_03/_ato2011-2014/2014/lei/l12965.htm
- Decreto nº 8.771/2016 (regulamentação, neutralidade e segurança): https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2016/decreto/d8771.htm

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
