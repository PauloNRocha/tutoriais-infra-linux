# Guia de Produção: Servidor NTP Interno com Chrony no Debian 13

*Criado em: 15 de janeiro de 2026*

Este tutorial é um guia **completo, detalhado, didático e pronto para produção** para implantar um **servidor NTP interno** usando **exclusivamente o Chrony** no **Debian 13**.

O servidor ficará com dois papéis ao mesmo tempo:
- **Cliente**: sincroniza com servidores NTP públicos confiáveis (upstreams).
- **Servidor**: distribui tempo para os dispositivos da sua rede interna (clientes).

> Aviso de segurança (importante): não exponha NTP para a internet sem necessidade. Servidores NTP abertos podem ser abusados em ataques de amplificação. Neste guia, o NTP é **interno** e o firewall restringe o acesso.

---

## Índice
1. [Introdução](#1-introdução)
2. [Arquitetura proposta](#2-arquitetura-proposta)
3. [Preparação do sistema](#3-preparação-do-sistema)
4. [Instalação do chrony](#4-instalação-do-chrony)
5. [Configuração do chrony](#5-configuração-do-chrony-chronyconf)
6. [Segurança e boas práticas](#6-segurança-e-boas-práticas)
7. [Configuração de clientes](#7-configuração-de-clientes)
8. [Testes e validação](#8-testes-e-validação)
9. [Logs e troubleshooting](#9-logs-e-troubleshooting)
10. [Manutenção e operação](#10-manutenção-e-operação)
11. [NTS](#11-nts-network-time-security-opcional)
12. [Referências](#12-referências-leitura-recomendada)

---

## 1. Introdução

**Escopo deste guia:** este tutorial trata **exclusivamente** de um **servidor NTP interno**, voltado para ambientes corporativos/datacenter, com **acesso restrito por firewall**.  
Ele **não** cobre o cenário de “NTP público exposto à internet”.

### 1.1 O que é NTP

**NTP (Network Time Protocol)** é um protocolo de rede que sincroniza data e hora entre computadores/dispositivos.  
Ele permite que várias máquinas “concordem” sobre o horário, mesmo com variações de rede e com relógios de hardware que naturalmente adiantam/atrasam (drift).

Ponto prático:
- NTP usa principalmente **UDP/123**.
- O Chrony tem o daemon **`chronyd`** e a ferramenta de administração **`chronyc`**.

### 1.2 Por que sincronização de tempo é crítica

Tempo correto é pré-requisito para:
- **Logs coerentes** (auditoria e troubleshooting).
- **TLS/SSL** (certificados dependem de data/hora correta).
- **Autenticação** (ex.: Kerberos é extremamente sensível a “relógio fora”).
- **Sistemas distribuídos** (ordem de eventos e consistência).

### 1.3 Problemas causados por data/hora incorretas

Quando o horário está errado, é comum ocorrer:
- logs “fora de ordem” (parece que um evento aconteceu antes do outro);
- falhas de login/autenticação;
- validação de certificado falhar (“ainda não é válido” / “expirado”);
- dificuldade de correlacionar incidentes entre sistemas.

### 1.4 Por que o chrony substitui o ntp tradicional

No Linux moderno, o **Chrony** é a opção recomendada porque:
- converge mais rápido após reboot;
- lida melhor com variações de rede e ambientes virtualizados;
- tem recursos atuais de segurança e controle (ex.: `ratelimit`, regras de acesso).

Neste guia:
- **não** usamos o pacote antigo `ntp`;
- usamos **exclusivamente** `chrony`.

---

## 2. Arquitetura proposta

### 2.1 Servidor NTP interno

Você terá um servidor Debian 13 rodando Chrony, servindo NTP para a rede interna.

### 2.2 Sincronização com NTP público (upstreams)

O servidor NTP interno (Chrony) consulta fontes externas para manter precisão.

Boas práticas de upstream:
- use **mais de uma fonte** (redundância);
- evite depender de um único servidor;
- prefira **pools** (que rotacionam várias fontes).

### 2.3 Clientes internos consumindo esse servidor

Clientes (servidores, desktops, VMs e outros dispositivos) devem apontar **somente** para o NTP interno, em vez de consultar NTP público diretamente.

### 2.4 Benefícios dessa arquitetura

- **Consistência**: toda a rede compartilha a mesma referência de tempo.
- **Segurança**: você não precisa abrir a rede inteira para a internet.
- **Controle (opcional)**: dá para auditar quais IPs consultam o NTP com `chronyc clients`, mas isso depende do acesso aos comandos administrativos do Chrony (ex.: `cmdallow`) e **não** é indicador obrigatório de “saúde”.
- **Resiliência**: em quedas de internet, o servidor pode continuar mantendo um relógio coerente internamente (com precisão limitada pelo hardware e configuração).

---

## 3. Preparação do sistema

### 3.1 Atualização do Debian 13

```bash
sudo apt update
sudo apt -y full-upgrade
sudo apt -y autoremove
```

Por que isso vem antes?
- reduz chance de instalar versões antigas/bugadas;
- garante correções de segurança.

### 3.2 Remoção de serviços conflitantes (ex: systemd-timesyncd, se aplicável)

O Debian pode usar `systemd-timesyncd` como sincronização básica. Para evitar conflito com o Chrony:

1) Desative a sincronização “automática” do systemd:
```bash
sudo timedatectl set-ntp false
```

2) Pare e desabilite o serviço (se existir):
```bash
sudo systemctl disable --now systemd-timesyncd || true
sudo systemctl mask systemd-timesyncd || true
```

3) Garanta que o pacote antigo `ntp` não existe:
```bash
dpkg -l | grep -E '^ii[[:space:]]+ntp([[:space:]]|$)' && sudo apt -y remove --purge ntp || true
```

### 3.3 Verificação de timezone e relógio do sistema

Em servidores, uma boa prática é usar **UTC** (facilita correlação de logs).  
Se você precisa de horário local, use o timezone local — o NTP funciona do mesmo jeito, pois sincroniza o relógio em UTC internamente e aplica timezone na apresentação.

Verificar:
```bash
timedatectl
```

Definir UTC (recomendado para servidores):
```bash
sudo timedatectl set-timezone UTC
```

Verifique o relógio de hardware (RTC):
```bash
sudo hwclock --show || true
```

> Em ambientes virtualizados, `hwclock` pode não fazer sentido (depende do hypervisor). Não force correções sem entender o ambiente.

### 3.4 Ambientes físicos vs virtualizados

Este é um ponto que confunde muita gente (e gera “ajustes” desnecessários):

- **Bare-metal (físico)**: existe um **RTC** real. Faz sentido falar de `hwclock`, `rtcsync` e “RTC alinhado”.
- **Máquina virtual (VM)**: o “relógio de hardware” pode **não existir** como um RTC real ou pode ser **controlado pelo hypervisor** (host).

Boas práticas:
- em **VMs**, evite rodar `hwclock --systohc`/`--hctosys` “no automático” sem entender como seu hypervisor entrega o tempo;
- prefira manter o tempo correto via NTP (Chrony) e deixe o host cuidar do que for “relógio de hardware” da VM.

---

## 4. Instalação do chrony

### 4.1 Instalação via apt

```bash
sudo apt install -y chrony nftables
```

### 4.2 Explicação dos serviços criados

Após instalar:
- `chronyd` é o daemon que faz a sincronização e responde NTP.
- `chronyc` é o cliente/console para consultar status e administrar.
- no Debian, o serviço systemd geralmente aparece como **`chrony`**.

### 4.3 Inicialização automática no boot

Verifique o serviço:
```bash
systemctl status chrony --no-pager
```

Garanta que inicia no boot:
```bash
sudo systemctl enable --now chrony
```

Ver versão:
```bash
chronyc --version
```

---

## 5. Configuração do chrony (chrony.conf)

### 5.1 Onde fica o arquivo e como editar com segurança

Arquivo:
- `/etc/chrony/chrony.conf`

Faça backup antes:
```bash
sudo cp /etc/chrony/chrony.conf "/etc/chrony/chrony.conf.bak.$(date +%F_%H%M%S)"
```

Edite:
```bash
sudo nano /etc/chrony/chrony.conf
```

### 5.2 Uso de servidores NTP públicos confiáveis

Exemplos comuns (use mais de um pool):
- `pool.ntp.org` (pool público global/por região)
- `*.debian.pool.ntp.org` (pool do Debian)

> Evite depender de um único hostname. Ponto de partida seguro é usar 4 pools.

#### 5.2.1 Servidores brasileiros (NTP.br) (opcional)

Se você está no Brasil e quer sincronizar com a **Hora Legal Brasileira**, o **NTP.br (NIC.br)** mantém servidores públicos e documentação bem completa.

Lista resumida (consulte a lista oficial para atualizações):

| Nome | IPv4 | IPv6 | NTS |
|---|---|---|---|
| `a.st1.ntp.br` | `200.160.7.186` | `2001:12ff:0:7::186` | sim |
| `b.st1.ntp.br` | `201.49.148.135` | `-` | sim |
| `c.st1.ntp.br` | `200.186.125.195` | `-` | sim |
| `d.st1.ntp.br` | `200.20.186.76` | `-` | sim |
| `gps.ntp.br` | `200.160.7.193` | `2001:12ff:0:7::193` | sim |
| `a.ntp.br` | `200.160.0.8` | `2001:12ff::8` | não |
| `b.ntp.br` | `200.189.40.8` | `2001:12f8:9:1::8` | não |
| `c.ntp.br` | `200.192.232.8` | `2001:12f8:b:1::8` | não |

> Dica: consulte a lista oficial e o status de NTS em `https://ntp.br/` e/ou `https://www.ntp.br/faq/`.

Exemplo (NTP clássico, sem NTS):
```conf
server a.ntp.br iburst
server b.ntp.br iburst
server c.ntp.br iburst
```

Exemplo (NTS com NTP.br, para seu servidor sincronizar com upstreams via NTS):
```conf
# NTS no Chrony é usado em associações cliente/servidor (unicast).
# Por isso usamos `server ... nts` (e não `pool ... nts`).
ntsdumpdir /var/lib/chrony

server a.st1.ntp.br iburst nts
server b.st1.ntp.br iburst nts
server c.st1.ntp.br iburst nts
server d.st1.ntp.br iburst nts
server gps.ntp.br iburst nts
```

### 5.3 Configuração exemplo (servidor NTP interno), comentada e pronta para produção

Substitua o conteúdo do seu `chrony.conf` por um modelo como este (ajuste as redes internas):

```conf
# ==========================================================
# Chrony (Debian 13) - Servidor NTP interno (produção)
# ==========================================================

# --- FONTES (UPSTREAMS) ---
# pool = usa várias fontes automaticamente (recomendado para redundância)
# iburst = acelera a sincronização inicial enviando uma "rajada" no começo
pool 0.pool.ntp.org iburst
pool 1.pool.ntp.org iburst
pool 2.pool.ntp.org iburst
pool 3.pool.ntp.org iburst

# Pool do Debian (opcional, como fonte adicional)
pool 0.debian.pool.ntp.org iburst
pool 1.debian.pool.ntp.org iburst

# --- DERIVA (DRIFT) ---
# Guarda a taxa de desvio do relógio para convergir mais rápido após reboot
driftfile /var/lib/chrony/chrony.drift

# --- CORREÇÃO INICIAL (BOOT) ---
# Corrige "na hora" se o erro for maior que 1 segundo,
# mas só nas 3 primeiras medições (depois apenas ajustes graduais)
makestep 1.0 3

# --- RTC (RELÓGIO DE HARDWARE) ---
# Mantém o RTC alinhado com o relógio do sistema
rtcsync

# --- (OPCIONAL) VMs críticas: convergência após clone/restore ---
# Em VMs que sofrem “saltos” de tempo (snapshot/clone/restore), pode ser útil limitar
# a taxa máxima de slewing para manter o ajuste mais previsível.
#
# ATENÇÃO:
# - isso não substitui `makestep` (boot) nem o procedimento de recuperação do item 9.4;
# - valores menores podem tornar correções grandes mais lentas.
# maxslewrate 1000

# --- SERVIR NTP PARA A REDE INTERNA ---
# Permita SOMENTE suas redes internas (troque pelos seus blocos reais)
allow 192.168.0.0/16
allow 10.0.0.0/8
allow fd00::/8
#
# Importante:
# - O padrão do chronyd é NÃO permitir clientes (ou seja: ele vira servidor só quando você usa `allow`).
# - Em produção controlada, normalmente basta listar os `allow` e não usar `deny all`.
# - Se você quiser bloquear algum subset dentro de um `allow`, use `deny <subnet>` só para aquele bloco.

# --- LOGS (RECOMENDADO PARA AUDITORIA/TROUBLESHOOTING) ---
logdir /var/log/chrony
log tracking measurements statistics

# --- PROTEÇÃO CONTRA ESTIMATIVAS RUINS (RECOMENDADO) ---
# Evita que medições extremamente instáveis/ruidosas gerem uma estimativa de frequência
# “não confiável” e acabem afetando o relógio do sistema.
# É uma proteção passiva: em condições normais não muda nada, mas ajuda em cenários ruidosos
# (ex.: upstreams públicos com jitter/perda).
maxupdateskew 100.0

# --- DEFESA CONTRA ABUSO ---
# Mitiga flood/abuso e reduz risco de amplificação.
# interval em potências de 2 segundos (interval=1 => 2s)
ratelimit interval 1 burst 8

# --- (OPCIONAL) “SEGURAR A REDE” QUANDO A INTERNET CAI ---
# Só use se fizer sentido para o seu ambiente.
# Veja explicação no item 5.4.
# local stratum 10
```

Depois reinicie:
```bash
sudo systemctl restart chrony
```

#### 5.3.1 IMPORTANTE: VMs, clones, snapshots e primeira sincronização

Em ambientes virtualizados (VMs), principalmente quando você:
- clona uma VM,
- restaura snapshot,
- faz restore de backup,
- ou sobe uma VM “do zero” com RTC/clock ruim,

é comum o relógio (RTC/hypervisor) vir **muito errado** e o Chrony entrar em um estado “inconsistente”, onde ele **não converge** ou demora demais para voltar a sincronizar.

Sintomas típicos:
- `chronyc tracking` mostrando:
  - `Reference ID    : 00000000`
  - `Stratum         : 0`
  - `Ref time (UTC)  : Thu Jan  1 00:00:00 1970`
  - `Leap status     : Not synchronised`
- `chronyc sources -v` mostrando somente `^?` e `Reach` zerado (`0`).

Ponto importante: se você já confirmou que:
- **há conectividade** até os upstreams (UDP/123 saindo), e
- o servidor NTP responde (há request/reply no `tcpdump`, por exemplo),

então isso geralmente **não é firewall/DNS/rede**. É o Chrony em um estado interno ruim após o evento de VM (clone/restore), e a recuperação correta é feita limpando estado e forçando nova sincronização (veja o item **9.4**).

### 5.4 Explicação detalhada de cada diretiva importante

#### server / pool
- `server` aponta para **um** servidor específico.
- `pool` aponta para **um conjunto** de servidores (mais resiliente).

Neste guia usamos `pool` por ser melhor para produção.

#### iburst
`iburst` envia várias consultas rapidamente no início, fazendo o Chrony sincronizar mais rápido após boot.

#### driftfile
`driftfile` grava o “comportamento” do relógio (quanto ele adianta/atrasa).  
Isso ajuda a manter precisão e a convergir mais rápido após reinicialização.

#### makestep
`makestep 1.0 3` significa:
- se o erro for maior que **1 segundo**, o Chrony pode “pular” o relógio (step);
- mas isso só acontece nas **3 primeiras** atualizações (normalmente no boot).

Por que isso é importante em produção?
- “step” durante a operação pode bagunçar logs e processos sensíveis;
- “step” no boot é aceitável para corrigir diferenças grandes rapidamente.

#### logdir
Define o diretório onde o Chrony grava logs (quando você habilita `log ...`).

#### maxupdateskew
`maxupdateskew` define um limite (em **ppm**) para decidir se uma estimativa de ajuste de frequência do relógio está **confiável** o suficiente para ser aplicada.

Resumidamente:
- quando as medições estão muito ruidosas (jitter/perda), o Chrony pode gerar uma estimativa de “quanto o relógio adianta/atrasa” com uma margem de erro muito grande;
- `maxupdateskew 100.0` manda o Chrony **ignorar** atualizações cuja estimativa esteja “ruim demais”, evitando bagunçar o relógio do sistema com dados instáveis.

Por que é bom em produção:
- é uma **proteção passiva**, segura e recomendada, especialmente quando você usa **upstreams públicos**;
- em condições normais não tem impacto perceptível (as estimativas são boas e passam no limite).

Nota prática:
- se a rede até os upstreams for muito instável, valores mais restritivos podem fazer o Chrony demorar mais para aceitar estimativas “boas”.

#### allow / deny
Controla quais redes podem usar seu servidor como fonte NTP.

Boas práticas:
- liste apenas redes internas realmente necessárias;
- use firewall em paralelo (defesa em profundidade).

Importante (comportamento padrão do Chrony):
- por padrão, **nenhum cliente** é permitido (chronyd opera só como cliente);
- quando você adiciona `allow`, você habilita o modo “servidor” para as redes permitidas.

**ATENÇÃO:** Evite usar `deny all` como “fechamento”:
- além de ser desnecessário (porque o padrão já é negar quem não está em `allow`),
- em algumas situações ele pode anular permissões e impedir o serviço de responder como servidor NTP.

#### local stratum (se aplicável)
`local stratum 10` faz o servidor continuar **respondendo** como fonte de tempo mesmo se perder todas as fontes externas.

Quando faz sentido:
- ambientes que precisam manter tempo coerente internamente durante falhas curtas de internet;
- redes isoladas onde a referência externa pode ficar indisponível temporariamente.

Risco:
- **não garante precisão absoluta**: ele mantém a rede sincronizada **entre si**, mas se o servidor estiver fora, ele pode **propagar erro** para todos.

Recomendação:
- **Atenção:** Use apenas se você entende o impacto e **monitore upstreams regularmente** (`chronyc sources -v`).

### 5.5 Diferença entre servidor NTP e cliente NTP

- **Cliente NTP**: sincroniza seu relógio consultando alguém (fontes).
- **Servidor NTP**: responde consultas de outros dispositivos (clientes).

No Chrony, você “vira servidor” quando:
- libera redes com `allow ...` e
- mantém UDP/123 acessível no firewall para essas redes.

---

## 6. Segurança e boas práticas

### 6.1 Restringir quem pode consultar o NTP

Camadas de proteção recomendadas:
1) `allow/deny` no `chrony.conf`
2) firewall `nftables` liberando UDP/123 **apenas** para redes internas

### 6.2 Exemplos de allow/deny no chrony

Exemplo seguro (apenas redes privadas + ULA IPv6):
```conf
allow 10.0.0.0/8
allow 172.16.0.0/12
allow 192.168.0.0/16
allow fd00::/8
```

Se você precisar bloquear um trecho específico dentro de uma rede permitida, use `deny` apenas para o trecho (exemplo):
```conf
allow 192.168.0.0/16
deny  192.168.50.0/24
```

Observação sobre IPv6 (opcional, mas importante em redes mais maduras):
- `fd00::/8` é **ULA** (IPv6 “privado”), comum em redes internas.
- Se a sua rede usa **IPv6 global roteável** (prefixos públicos do seu bloco IPv6), inclua também os **prefixos IPv6 reais** dos seus clientes/servidores nas regras `allow`/firewall (e não apenas ULA).

### 6.3 Firewall (exemplo com nftables)

Objetivo:
- permitir UDP/123 **apenas** da rede interna;
- não expor UDP/323 (chronyc) para a rede.

Edite o arquivo persistente:
```bash
sudo nano /etc/nftables.conf
```

Exemplo mínimo (ajuste redes e integre com seu ruleset):
```nft
flush ruleset

table inet filter {
  set ntp_clients_v4 { type ipv4_addr; flags interval; elements = { 10.0.0.0/8, 192.168.0.0/16 } }
  set ntp_clients_v6 { type ipv6_addr; flags interval; elements = { fd00::/8 } }

  chain input {
    type filter hook input priority 0;
    policy drop;

    iif "lo" accept
    ct state established,related accept

    # NTP (chronyd): UDP/123 apenas redes internas
    udp dport 123 ip  saddr @ntp_clients_v4 accept
    udp dport 123 ip6 saddr @ntp_clients_v6 accept
  }

  chain forward { type filter hook forward priority 0; policy drop; }
  chain output  { type filter hook output  priority 0; policy accept; }
}
```

Aplicar:
```bash
sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
```

> Se no seu ambiente a saída (`output`) for `drop`, você precisa liberar **saída UDP/123** para os upstreams, senão o servidor nunca sincroniza.

### 6.4 Considerações sobre abuso de NTP (amplification)

Servidor NTP aberto é alvo comum de abuso (amplificação).  
Para não virar parte de um ataque:
- não exponha UDP/123 para a internet;
- restrinja por rede (allow/deny + firewall);
- use `ratelimit` no Chrony.

---

## 7. Configuração de clientes

Objetivo: todos os clientes devem apontar **somente** para o NTP interno.

Conceito universal:
> Independentemente do sistema (Linux, Windows, appliance, switch), o princípio é o mesmo: **os clientes apontam para o NTP interno; apenas o servidor NTP acessa a internet**.

### 7.1 Servidores Linux usando chrony (recomendado)

1) Instale:
```bash
sudo apt update
sudo apt install -y chrony
```

2) Edite `/etc/chrony/chrony.conf` e deixe apenas seu NTP interno como fonte:
```conf
server SEU_SERVIDOR_NTP_INTERNO iburst
```

Exemplo usando IP:
```conf
server 192.168.50.10 iburst
```

3) Reinicie:
```bash
sudo systemctl restart chrony
```

### 7.2 Servidores Linux usando systemd-timesyncd, quando você NÃO quer instalar chrony no cliente

1) Garanta que o timesyncd está ativo:
```bash
sudo systemctl unmask systemd-timesyncd || true
sudo systemctl enable --now systemd-timesyncd
sudo timedatectl set-ntp true
```

2) Edite `/etc/systemd/timesyncd.conf`:
```bash
sudo nano /etc/systemd/timesyncd.conf
```

Exemplo:
```ini
[Time]
NTP=192.168.50.10
FallbackNTP=
```

3) Reinicie:
```bash
sudo systemctl restart systemd-timesyncd
```

4) Verifique:
```bash
timedatectl status
```

### 7.3 Explicação conceitual válida para qualquer dispositivo

Qualquer dispositivo que suporte NTP geralmente pede:
- IP/hostname do servidor NTP
- (às vezes) um “servidor secundário”

Boa prática:
- use o NTP interno como primário;
- só use secundário externo se realmente necessário e com política de rede bem definida.

---

## 8. Testes e validação

### 8.1 Comandos chronyc (servidor)

#### tracking
Mostra o estado geral do relógio:
```bash
chronyc tracking
```

Como interpretar:
- **System time**: offset atual (quanto está “adiantado/atrasado”).
- **Last offset**: último offset medido.
- offsets pequenos e estáveis normalmente indicam bom funcionamento.

#### sources
Lista as fontes (upstreams) e qual está sendo usada:
```bash
chronyc sources -v
```

Como interpretar (pontos principais):
- um `*` indica a fonte selecionada;
- `^?` indica fonte **inacessível**;
- `Reach` deve ir “preenchendo” (quando há conectividade).

### 8.2 O que é “normal” vs “problema” (interpretação prática)

#### Sinais de servidor saudável

- `chronyc sources -v` mostrando pelo menos uma fonte com `^*` (fonte atual) e/ou `^+` (fontes combinadas).
- `Reach` preenchido e estável (ex.: `377`).
- `chronyc tracking` com offset pequeno/estável (em geral, convergindo ao longo do tempo).
- `Stratum` baixo (ex.: 2, 3, 4) e `Leap status: Normal` no `chronyc tracking`.
- Se você já tem clientes apontados para ele, o teste **mais confiável** é do lado do cliente (ver abaixo).

#### Sinais de problema

- `chronyc sources -v` com tudo `^?` (todas as fontes inacessíveis).
- `Reach` zerado (ex.: `0`) por muito tempo.
- `Stratum` muito alto (ex.: `16`) e/ou indicação de “não sincronizado”.
- offset que só cresce e não converge.

Se o servidor parece saudável, mas algum cliente não sincroniza, valide **no cliente**:
- em clientes com Chrony: `chronyc tracking` (deve apontar para o seu servidor interno como referência);
- em clientes com systemd-timesyncd: `timedatectl status` (deve indicar “System clock synchronized: yes”).

Nota prática (produção): em alguns cenários, o `timedatectl` pode demorar a refletir o “estado real” quando você usa Chrony (principalmente logo após mudanças/restarts).  
Para validar o servidor, use como base principal:
- `chronyc tracking` (procure `Leap status: Normal` e `Stratum` > 0)
- `chronyc sources -v` (procure ao menos uma fonte `^*` e `Reach` > 0, com offsets baixos)

#### Dica importante sobre `Reach`

O `Reach` é exibido em **octal** (base 8).  
O valor `377` geralmente indica que as **últimas 8 tentativas** de comunicação com a fonte deram certo.

#### (Opcional) `chronyc clients` (visão administrativa)
Lista clientes que estão consultando este servidor (visão administrativa):
```bash
chronyc clients
```

Observação importante (produção): **não use `chronyc clients` como indicador obrigatório de saúde**.

Motivos:
- `chronyc clients` é uma interface **administrativa** (não é o “mecanismo NTP” em si).
- A disponibilidade/saída depende de permissões na interface administrativa (ex.: `cmdallow`/`cmdport`) e de hardening/políticas do ambiente.
- Mesmo com NTP funcionando perfeitamente, o comando pode não listar o que você espera (ou pode retornar erros como **`501 Not authorised`**).

Verificação correta de “cliente sincronizando”:
- faça a validação **no cliente** (Chrony: `chronyc tracking`; systemd-timesyncd: `timedatectl status`).

Dica prática (produção):
- se você precisa rodar `chronyc` **de outra máquina**, a abordagem mais segura é executar via SSH no servidor NTP (evita expor/abrir a interface administrativa na rede).

### 8.3 Testes adicionais (porta/serviço)

Confirme que o servidor está escutando em UDP/123:
```bash
sudo ss -u -lnp | grep -E '(:123|:ntp)([[:space:]]|$)' || true
```

Confirme o status do serviço:
```bash
systemctl status chrony --no-pager
```

### 8.4 Como saber se o servidor está saudável

Checklist de saúde:
- `chronyc sources -v` mostra fontes alcançáveis (não apenas `^?`);
- `chronyc tracking` com offsets razoáveis e não “explodindo”;
- validação no cliente confirma sincronização (`chronyc tracking` ou `timedatectl status`).

---

## 9. Logs e troubleshooting

### 9.1 Onde ficam os logs

Opção A (sempre disponível): journald
```bash
sudo journalctl -u chrony -n 200 --no-pager
sudo journalctl -u chrony -f
```

Opção B (se você habilitou `logdir` + `log ...` no `chrony.conf`):
```bash
sudo ls -la /var/log/chrony || true
sudo tail -f /var/log/chrony/tracking.log || true
```

### 9.2 Diagnóstico rápido (runbook de produção)

Esta seção é pensada para **incidentes reais** (NOC/produção): curta, direta e “executável”.

#### Sintoma A: o servidor não sincroniza com upstream (“hora não ajusta” / tudo `^?` / `Reach` zerado)

- **Comando**
  ```bash
  chronyc sources -v
  ```
- **O que esperar**
  - pelo menos uma fonte alcançável (não apenas `^?`);
  - `Reach` preenchendo (não ficando em `0`).
- **Próximo passo**
  - ver logs do serviço e procure erros de rede/DNS:
    ```bash
    sudo journalctl -u chrony -n 200 --no-pager
    ```
  - se ainda estiver inconclusivo, confirme tráfego NTP:
    ```bash
    sudo tcpdump -ni any udp port 123
    ```
    > Use `tcpdump` apenas para diagnóstico pontual. Não deixe rodando permanentemente em produção.

#### Sintoma B: `Offset` alto e não converge

- **Comando**
  ```bash
  chronyc tracking
  ```
- **O que esperar**
  - offset reduzindo/convergindo ao longo do tempo.
- **Próximo passo**
  - veja estatísticas das fontes e valide o driftfile:
    ```bash
    chronyc sourcestats -v
    sudo ls -la /var/lib/chrony/chrony.drift || true
    ```

#### Sintoma C: o servidor parece saudável, mas um cliente não sincroniza (MikroTik, `ntpd`, outro Linux)

- **Comando (no servidor)**
  ```bash
  chronyc accheck <IP_DO_CLIENTE>
  ```
- **O que esperar**
  - “allowed” (ou equivalente). Se der “denied”, a rede/IP do cliente não está coberta pelos `allow`.
- **Próximo passo**
  - confirme se o servidor está **escutando** em UDP/123:
    ```bash
    sudo ss -u -lnp | grep -E '(:123|:ntp)([[:space:]]|$)' || true
    ```
  - confirme a regra do firewall para UDP/123 (nftables):
    ```bash
    sudo nft list ruleset | grep -nE 'udp dport 123|ntp_clients' || true
    ```
  - prove request/reply “no fio” (diagnóstico pontual; use a interface correta):
    ```bash
    sudo tcpdump -ni <IFACE> udp port 123
    ```
	  - valide **no cliente** (o teste mais confiável):
	    ```bash
	    # Se o cliente usa Chrony:
	    chronyc sources -v

	    # Se o cliente usa systemd-timesyncd:
	    timedatectl status
	    ```
  - (opcional) ver “drops” e zerar contadores:
    ```bash
    chronyc clients -r
    ```

### 9.3 Troubleshooting aprofundado (explicações e casos)

Esta seção explica **o porquê** dos sintomas do runbook (seção 9.2) e como evitar armadilhas comuns em produção ISP.

### 9.4 Recuperação do Chrony em estado inconsistente (Stratum 0 / 1970 / Not synchronised)

Use este procedimento principalmente quando:
- VM foi clonada/restaurada (snapshot/backup) e voltou com hora “absurda”;
- `chronyc tracking` mostra `Stratum 0`, `Ref time 1970` e `Leap status Not synchronised`;
- `chronyc sources -v` fica em `^?` com `Reach 0`, mesmo com rede ok.

Passo a passo (execute exatamente nesta ordem):

```bash
sudo systemctl stop chrony
sudo rm -f /var/lib/chrony/chrony.drift
sudo rm -f /var/lib/chrony/*.dat
sudo systemctl start chrony
sleep 10
chronyc sources -v
```

Se necessário (quando as fontes estão alcançáveis, mas ainda demora a “pegar”):

```bash
chronyc burst 4/4
chronyc sources -v
```

Se ainda houver uma diferença grande (muitos segundos/minutos) — especialmente pós-restore/clone — force step:

```bash
chronyc makestep
```

Depois (recomendado para persistir em bare-metal; em VM pode não fazer sentido dependendo do hypervisor):

```bash
sudo hwclock --systohc
```

Observação importante sobre `makestep`:
- use quando a diferença é grande e você precisa corrigir rápido (boot/restore/clone);
- evite “step” durante operação normal, porque pode bagunçar logs e processos sensíveis.

#### 9.3.1 “Hora não ajusta”: o que isso costuma significar

Quando o servidor não sincroniza com upstream, os sintomas mais comuns são:
- `chronyc sources -v` com tudo `^?` (fontes inacessíveis);
- `Reach` zerado por muito tempo;
- `chronyc tracking` não converge.

O que normalmente está por trás disso (do mais comum para o menos comum):
- **bloqueio de UDP/123** (bordas/firewalls/políticas corporativas);
- **DNS quebrado** no servidor (não resolve pools/hosts);
- **upstreams indisponíveis** (fonte fora/rota ruim).

Se você já executou o **Sintoma A** do runbook (9.2) e ainda não ficou claro, use o `tcpdump` (9.2) para separar “não sai pacote”, “sai e não volta” e “sai e volta mas está instável”. Isso direciona a correção sem tentativa-e-erro.

#### 9.3.2 “Offset alto”: por que acontece e como pensar

Offset alto que não converge costuma ser consequência de:
- rede ruim até as fontes (perda/jitter), gerando medições inconsistentes;
- host com carga alta (CPU/memória), atrasando respostas e amostras;
- driftfile ausente/sem gravação, prejudicando a compensação de drift entre boots.

No runbook (9.2), você já valida `tracking`, `sourcestats` e `driftfile`. Se os dados apontarem para instabilidade nas fontes, o foco deixa de ser “Chrony” e passa a ser **qualidade de rede**/seleção de upstream.

#### 9.3.3 “Cliente não sincroniza”: causas típicas (e por que o problema pode ser “silencioso”)

Em produção, o caso mais comum não é “Chrony não funciona”, e sim “política não está coerente”:
- o cliente aponta para o servidor correto, mas a rede/IP não está coberta pelos `allow`;
- o `allow` cobre, mas o firewall (nftables) não cobre (ou cobre outro range);
- o servidor está sincronizado como **cliente**, mas não está servindo NTP (não está escutando em UDP/123).

Por isso o runbook (9.2) primeiro checa:
1) `accheck` (política do Chrony),
2) escuta em UDP/123,
3) firewall,
4) request/reply no fio.

Se no `tcpdump` (9.2) você vê apenas **Client** chegando e não vê **Server** saindo, o problema deixa de ser “o cliente” e passa a ser “o servidor não está respondendo”. Isso normalmente é coerente com falha de política, serviço que não está servindo, ou bloqueio local.

#### 9.3.4 Aviso MUITO importante: `deny all` pode quebrar “silenciosamente”

Evite usar `deny all` como “fechamento” do arquivo.

Padrão mais “à prova de susto”:
- use apenas `allow` para as redes necessárias;
- restrinja no firewall (nftables) por rede;
- se precisar negar um trecho específico, use `deny <subnet>` apenas para aquele trecho.

#### 9.3.5 Alinhe os ranges (chrony.conf ↔ firewall ↔ IPs reais)

Um erro comum em redes grandes é ter “ranges parecidos” mas diferentes:
- no `chrony.conf`: `allow 192.168.10.0/24`
- no firewall: set com `192.168.100.0/24`

Resultado: chega request, mas a política não está coerente e a validação fica confusa.

Boa prática:
1) descubra o IP real do cliente (pelo `tcpdump`, conforme runbook 9.2);
2) garanta que ele está dentro de um `allow` **e** dentro do conjunto liberado no firewall.

---

## 10. Manutenção e operação

### 10.1 Atualizações

Manter o sistema e o Chrony atualizados é parte de operação de produção:
```bash
sudo apt update
sudo apt -y full-upgrade
```

### 10.2 Monitoramento básico

Rotina recomendada (diária/semanal):
- `chronyc tracking`
- `chronyc sources -v`
- `journalctl -u chrony` (erros)

Exemplo de “checagem rápida”:
```bash
chronyc tracking
chronyc sources -v | head -n 30
```

### 10.3 Boas práticas de longo prazo

- Use **múltiplas fontes** e revise periodicamente.
- (Opcional) Em ambientes críticos, é comum manter **dois servidores NTP internos** (primário e secundário), ambos sincronizando com múltiplos upstreams. Assim, os clientes podem apontar para duas fontes internas e você reduz ponto único de falha.
- Evite mexer manualmente na hora em produção.
- Mantenha firewall restritivo (UDP/123 apenas interno).
- Se habilitar logs em arquivo, configure rotação (`logrotate`) para não lotar disco (veja 10.5).
- Depois de grandes mudanças (troca de hardware, VM, clocksource), valide novamente (`tracking/sources`).

### 10.4 Rotina simples (checklist)

- **Semanal:** `chronyc tracking` e `chronyc sources -v`
- **Mensal (opcional):** revisar consumo/clients (`chronyc clients`) e confirmar se só há clientes esperados (se a interface administrativa estiver liberada/local)
- **Após reboot/manutenção:** validar sincronização (tracking + sources) antes de declarar “OK”

---

### 10.5 Rotação de logs do Chrony (logrotate) (se você usa `logdir`)

Se você ativou logs em arquivo no `chrony.conf` (por exemplo):
```conf
logdir /var/log/chrony
log tracking measurements statistics
```

…então, em produção, você precisa garantir 3 pontos:
1) o diretório existe e o Chrony consegue gravar nele;
2) os logs não crescem infinitamente;
3) o Chrony consegue “reabrir” os arquivos após rotação.

#### Passo 1) Garanta que o diretório de logs existe e tem permissões corretas

No Debian, o usuário/grupo do Chrony costuma ser **`_chrony`** (não `chrony`). Isso é uma **decisão do pacote do Debian**: ele cria o usuário de sistema `_chrony` e o `chronyd` derruba privilégios para ele na inicialização.  
Para não errar, descubra primeiro qual usuário existe no seu sistema:
```bash
getent passwd _chrony || getent passwd chrony
```

```bash
sudo mkdir -p /var/log/chrony
sudo chown _chrony:_chrony /var/log/chrony
sudo chmod 750 /var/log/chrony
```

> Se o seu sistema usar `chrony:chrony` (mais comum em outras distros), ajuste o `chown` conforme o usuário/grupo retornado pelo `getent`.

#### Passo 2) Configure (ou ajuste) o logrotate do Chrony

No Debian, o pacote `chrony` normalmente já instala um arquivo de logrotate. Mesmo assim, em produção vale conferir.

1) Verifique se existe:
```bash
sudo ls -la /etc/logrotate.d/chrony
```

2) Se não existir (ou se você quiser padronizar), crie/ajuste com este conteúdo:
```bash
sudo nano /etc/logrotate.d/chrony
```

```conf
/var/log/chrony/*.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    nocreate
    postrotate
        /usr/bin/chronyc cyclelogs > /dev/null 2>&1 || true
    endscript
}
```

O que este arquivo faz (resumo):
- `daily` + `rotate 14`: mantém 14 dias de histórico.
- `compress` + `delaycompress`: reduz uso de disco sem “atrapalhar” diagnósticos recentes.
- `nocreate`: deixa o próprio `chronyd` recriar os arquivos ao reabrir.
- `chronyc cyclelogs`: manda o `chronyd` fechar e reabrir os logs (fundamental para rotação correta).

#### Passo 3) Valide a configuração

1) Simule sem rotacionar (modo debug):
```bash
sudo logrotate -d /etc/logrotate.d/chrony
```

2) Se estiver OK, você pode forçar uma rotação (em janela de manutenção):
```bash
sudo logrotate -f /etc/logrotate.d/chrony
```

3) Confirme que os arquivos continuam sendo escritos:
```bash
sudo ls -la /var/log/chrony
sudo tail -n 20 /var/log/chrony/tracking.log || true
```

---

## 11. NTS (Network Time Security) (opcional)

Esta seção é opcional. O guia principal continua usando NTP “clássico” por padrão porque:
- é mais simples de operar;
- é compatível com praticamente todos os dispositivos;
- em redes internas protegidas por firewall, costuma ser suficiente.

### 11.1 O que é NTS

**NTS** é uma extensão moderna do NTP que adiciona **autenticação criptográfica** ao processo de sincronização.

Em alto nível, ele funciona em 2 etapas:
1) **NTS-KE (Key Establishment)**: o cliente faz uma negociação via **TLS** com o servidor (por padrão em **TCP/4460**) e recebe material criptográfico (cookies/chaves).
2) **NTP autenticado**: o NTP continua usando UDP, mas com autenticação baseada nas chaves negociadas.

Importante (produção): o NTS adiciona a fase **NTS-KE** via TCP (normalmente **4460**). Em seguida, o servidor informa ao cliente qual IP/porta usar para a sessão NTP. Na prática, muitas implementações usam **UDP/123**, mas valide sempre na documentação do provedor antes de “fechar” firewall.

O que muda na prática:
- com NTS, o cliente tem mais garantias contra **spoofing/MITM** na relação cliente ↔ upstream;
- sem NTS, NTP clássico é simples e funciona muito bem em redes controladas.

### 11.2 Quando usar NTS (e quando não usar)

Use NTS quando:
- o seu servidor NTP precisa sincronizar através de redes **menos confiáveis**;
- você quer aumentar garantias criptográficas da sincronização com upstream;
- seus clientes (se for usar NTS internamente) suportam NTS.

Não use NTS (cenário comum) quando:
- o serviço é estritamente **interno** e protegido por firewall;
- simplicidade operacional e compatibilidade com equipamentos legados são prioridade.

### 11.3 Opção A (recomendada): NTS apenas entre seu servidor e os upstreams

Aqui, seu servidor interno busca tempo “autenticado” dos upstreams via NTS, mas continua servindo NTP clássico para a LAN.  
Benefício: melhora o elo “mais exposto” (internet) sem exigir NTS nos clientes.

#### Passo 1) Use upstreams compatíveis com NTS

Você precisa escolher servidores upstream que suportem NTS. Nem todo servidor público suporta.

#### Passo 2) Ative persistência de cookies (evita NTS-KE em massa após reboot)

No `/etc/chrony/chrony.conf`, adicione:
```conf
ntsdumpdir /var/lib/chrony
```

#### Passo 3) Configure os upstreams com `nts` (exemplos reais)

Exemplos de upstreams públicos com NTS:
```conf
server time.cloudflare.com iburst nts
server sth1.nts.netnod.se iburst nts
server sth2.nts.netnod.se iburst nts
server ptbtime1.ptb.de iburst nts
server nts.netnod.se iburst nts
```

Observações:
- Use 2 a 4 fontes, equilibrando redundância vs. simplicidade.
- Evite “supor” que `pool.ntp.org`/`pool.ntp.br` suportam NTS: o pool não garante que todos os servidores participantes ofereçam NTS. Prefira hostnames explicitamente publicados como NTS.
- Se você está no Brasil, o NTP.br também oferece servidores com NTS (veja a seção **5.2.1**).

Se você quer um ambiente realmente previsível, evite fontes automáticas via DHCP:
```conf
# sourcedir /run/chrony-dhcp
```

Reinicie:
```bash
sudo systemctl restart chrony
```

#### Passo 4) Firewall: atenção ao TCP/4460 (NTS-KE)

Se o seu firewall tem `output drop`, você precisa liberar:
- **saída UDP/123** (NTP) e
- **saída TCP/4460** (NTS-KE).

> Se o seu provedor usar portas diferentes, ajuste conforme a documentação do serviço.

### 11.4 Opção B (avançada): habilitar NTS no seu servidor para clientes internos

Use esta opção quando você tem clientes Linux modernos com Chrony e quer que os clientes internos também validem criptograficamente o servidor NTP.

O que você vai precisar:
- certificado e chave para o servidor NTS (PKI/CA interna ou outro método);
- abrir **TCP/4460** apenas para a LAN (além de UDP/123).

Exemplo de configuração (no `chrony.conf`):
```conf
ntsport 4460
ntsservercert /etc/chrony/nts-server.crt
ntsserverkey  /etc/chrony/nts-server.key
ntsdumpdir /var/lib/chrony
```

Validação (no servidor):
```bash
sudo chronyc -N serverstats
sudo chronyc -N clients -k
```

### 11.5 Observação importante: “leap smear”

Alguns serviços públicos de NTP utilizam “leap smear” (tratamento especial de leap second) e recomendam **não misturar** com servidores que não fazem smear, para evitar inconsistências durante o período do leap second.

---

## 12. Referências (leitura recomendada)


Chrony (documentação e manpages):
https://chrony-project.org/documentation.html
https://chrony-project.org/doc/4.1/chrony.conf.html
https://chrony-project.org/doc/4.1/chronyc.html

Manpages do Debian (chrony.conf / chronyc):
https://manpages.debian.org/chrony.conf
https://manpages.debian.org/chronyc

nftables:
https://wiki.nftables.org/wiki-nftables/index.php/Main_Page

NTS (padrão e conceito):
https://www.rfc-editor.org/rfc/rfc8915

NTS no Chrony (documentação do chrony.conf):
https://chrony-project.org/doc/4.6/chrony.conf.html

Exemplos e validação de NTS (chronyc -N):
https://chrony-project.org/doc/4.6.1/chronyc.html

NTS (visão geral e fases):
https://developers.cloudflare.com/time-services/nts/

Serviços públicos com NTS (exemplos e portas):
https://www.netnod.se/time-and-frequency-services/nts

NTP.br (Hora Legal Brasileira / servidores / NTS):
https://ntp.br/
https://www.ntp.br/faq/
https://ntp.br/guia/linux/#chrony
https://ntp.br/conteudo/ntp/

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
