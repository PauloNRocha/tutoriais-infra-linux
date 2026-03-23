# Guia de Produção: BIND9 no Debian 13 com Primary/Secondary, TSIG, DNSSEC e Fail2Ban

*Criado em: 08 de janeiro de 2026*  
*Última atualização em: 23 de março de 2026*

Montei este guia para deixar registrado um padrão de implantação de **BIND9 autoritativo** no **Debian 13 (Trixie)**, com dois servidores, transferência de zona autenticada por **TSIG**, assinatura automática com **DNSSEC** e proteção básica com **nftables** e **Fail2Ban**.

Veja também: [guia de recuperação de zona BIND9 com DNSSEC após erro de sintaxe](./guia_producao_bind9_recuperacao_zona_dnssec.md)

Ele combina as seguintes características e boas práticas:

-   **Segurança de protocolo**: Transferência de zona com **TSIG** (mais robusto que confiar apenas em IPs).
-   **Segurança ativa**: Logs detalhados e **Fail2Ban** para bloqueio automático de abusos.
-   **DNSSEC moderno**: Configuração de `dnssec-policy` (KASP) com rotação automática de chaves.
-   **Boas práticas Debian**: Organização de arquivos, validação e técnicas avançadas de troubleshooting.

> **Convenção de nomes:** a documentação atual do BIND usa `Primary` e `Secondary`. Se você está acostumado com `Master` e `Slave`, a equivalência aqui é direta.

> **Objetivo:** ao final deste guia, a ideia é ter dois servidores autoritativos funcionando de forma previsível, com zonas forward e reversa, transferência Primary/Secondary autenticada por TSIG, DNSSEC com KASP e um mínimo de proteção para exposição pública em produção.


---

## Índice rápido
0. [O que você vai construir (visão rápida)](#0)
1. [Antes de começar (checklist obrigatório)](#1)
2. [Variáveis do exemplo (troque pelos seus dados)](#2)
3. [Preparação do sistema (ns1 e ns2)](#3)
4. [Firewall com nftables (ns1 e ns2)](#4)
5. [Organização de diretórios](#5)
6. [Logging do BIND (para auditoria + Fail2Ban)](#6)
7. [Configuração base do BIND (`named.conf.options`)](#7)
8. [TSIG: transferência segura Primary ↔ Secondary](#8)
9. [Primary (ns1): declarar zonas + criar arquivos de zona](#9)
10. [Secondary (ns2): declarar zonas e transferir do Primary](#10)
11. [Registro.br: publicar delegação (NS/glue) e DNSSEC (DS)](#11)
12. [Fail2Ban: bloquear abuso automaticamente (ns1 e ns2)](#12)
13. [(Opcional) DoH (DNS over HTTPS) — use com MUITA clareza de objetivo](#13)
14. [Checklist de validação final (antes de publicar)](#14)
15. [Troubleshooting (curto e prático)](#15)
16. [Cenário de desastre (DR): reconstruindo um NS perdido sem quebrar DNSSEC](#16)
17. [Referências (fontes para consulta)](#17)

---

<a id="0"></a>
## 0) O que você vai construir (visão rápida)

Dois servidores DNS autoritativos:

- **ns1** = Primary — edita zonas e notifica o Secondary
- **ns2** = Secondary — puxa zonas via transferência:
  - `AXFR` = transferência completa da zona
  - `IXFR` = transferência incremental, só com as mudanças

Com:

- Zona forward: `exemplo.com.br`
- Zona reverse IPv4: `113.0.203.in-addr.arpa` (exemplo `203.0.113.0/24`)
- Zona reverse IPv6: `d.c.b.a.8.b.d.0.1.0.0.2.ip6.arpa` (exemplo `2001:db8:abcd::/48`)
- **DNSSEC** com `dnssec-policy`
- **TSIG** para transferências e NOTIFY
- **nftables** (firewall)
- **Fail2Ban** (banimento automático por log)

---

<a id="1"></a>
## 1) Antes de começar (checklist obrigatório)

1. **IPs fixos** (IPv4 e, se possível, IPv6) em ambos os servidores.
2. Porta **53/UDP e 53/TCP** liberadas **até seus servidores** (DNS usa UDP, mas TCP é essencial para respostas grandes e transferências).
3. Hostnames/FQDN definidos corretamente:
   - `ns1.exemplo.com.br`
   - `ns2.exemplo.com.br`
4. Relógio sincronizado (DNSSEC depende disso): `chrony`/NTP.
5. Você tem acesso ao **Registro.br** do domínio e (se aplicável) ao painel de **Numeração** para reverso.

> Regra de ouro: servidor autoritativo público **não deve virar “open resolver”**. Por padrão, este guia deixa `recursion no;` (autoritativo puro). Se você realmente precisa recursão para rede interna, há uma seção opcional mais abaixo.

---

<a id="2"></a>
## 2) Variáveis do exemplo (troque pelos seus dados)

Nos exemplos abaixo, vou usar IPs de documentação (RFC 5737) para evitar colisão com endereços reais.

| Item | Exemplo |
|---|---|
| Domínio | `exemplo.com.br` |
| Primary | `ns1.exemplo.com.br` |
| Secondary | `ns2.exemplo.com.br` |
| IPv4 ns1 | `203.0.113.10` |
| IPv4 ns2 | `203.0.113.20` |
| IPv6 ns1 | `2001:db8:abcd::10` |
| IPv6 ns2 | `2001:db8:abcd::20` |
| Reverse IPv4 | `203.0.113.0/24` → `113.0.203.in-addr.arpa` |
| Reverse IPv6 | `2001:db8:abcd::/48` → `d.c.b.a.8.b.d.0.1.0.0.2.ip6.arpa` |
| Rede interna (opcional) | `192.168.1.0/24` e `2001:db8:1::/64` |

---

<a id="3"></a>
## 3) Preparação do sistema (ns1 e ns2)

### 3.1) Ajuste hostname (recomendado)

No **ns1**:

```bash
sudo hostnamectl set-hostname ns1.exemplo.com.br
```

No **ns2**:

```bash
sudo hostnamectl set-hostname ns2.exemplo.com.br
```

Em instalação limpa, também vale ajustar `/etc/hosts` para o hostname novo resolver localmente. Isso evita aviso de `sudo` do tipo `unable to resolve host ...`.

No **ns1**:

```bash
echo '127.0.1.1 ns1.exemplo.com.br ns1' | sudo tee -a /etc/hosts
```

No **ns2**:

```bash
echo '127.0.1.1 ns2.exemplo.com.br ns2' | sudo tee -a /etc/hosts
```

### 3.2) Instale pacotes

Em **ambos**:

```bash
sudo apt update
sudo apt install -y bind9 bind9-utils dnsutils nftables fail2ban
```

Se a máquina estiver muito enxuta e você estiver usando um usuário comum, pode ser necessário instalar `sudo` antes.  
Se você já entrou como `root`, basta executar os mesmos comandos sem o `sudo`.

Verifique a versão:

```bash
sudo named -v
```

Suba o serviço:

```bash
sudo systemctl enable --now named
```

Cheque:

```bash
systemctl status named --no-pager
```

### 3.3) Garanta horário sincronizado

```bash
timedatectl
```

Se o NTP não estiver ativo:

```bash
sudo timedatectl set-ntp true
```

Se isso retornar algo como `Failed to set ntp: NTP not supported`, instale e habilite `chrony`:

```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc tracking
```

Se você quiser um passo a passo completo só para NTP, veja também: [guia de produção de servidor NTP interno com Chrony no Debian 13](../sistema/guia_producao_ntp_chrony_debian13.md)

---

<a id="4"></a>
## 4) Firewall com nftables (ns1 e ns2)

> Se você já tem firewall, adapte. Não copie “no escuro”.
>
> **Produção:** restrinja o SSH ao seu IP/VPN/bastion sempre que possível.

Veja também: [Guia de Produção: Acesso SSH por chave pública (porta customizada) em servidores Linux](../acesso-remoto/guia_producao_ssh_chave_publica_linux.md)

Edite `/etc/nftables.conf`:

```nft
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;

    iif "lo" accept
    ct state established,related accept
    ct state invalid drop

    # ICMP/ICMPv6 (essencial, principalmente no IPv6)
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept

    # SSH (ajuste a porta se necessário)
    # Produção (recomendado): restrinja ao seu IP/VPN/bastion:
    # ip  saddr 198.51.100.10/32 tcp dport 22 ct state new accept
    # ip6 saddr 2001:db8:1::/64 tcp dport 22 ct state new accept
    tcp dport 22 ct state new accept

    # DNS autoritativo: UDP e TCP
    udp dport 53 accept
    tcp dport 53 ct state new accept

    # (Opcional) DoH para rede interna
    # tcp dport 443 ct state new accept

    reject with icmpx type port-unreachable
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

Aplique:

```bash
sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
```

---

<a id="5"></a>
## 5) Organização de diretórios

### 5.1) Primary (ns1)

```bash
sudo mkdir -p /var/lib/bind/master-aut/exemplo.com.br/keys
sudo mkdir -p /var/lib/bind/master-rev/IPv4/203.0.113.x/keys
sudo mkdir -p /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/keys
sudo chown -R bind:bind /var/lib/bind/master-aut /var/lib/bind/master-rev
sudo chmod -R 750 /var/lib/bind/master-aut /var/lib/bind/master-rev
```

### 5.2) Secondary (ns2)

```bash
sudo mkdir -p /var/cache/bind/slave-aut/exemplo.com.br
sudo mkdir -p /var/cache/bind/slave-rev/IPv4/203.0.113.x
sudo mkdir -p /var/cache/bind/slave-rev/IPv6/2001.db8.abcd.x
sudo chown -R bind:bind /var/cache/bind/slave-aut /var/cache/bind/slave-rev
```

> Observação: no Secondary, esse layout é só uma forma organizada de separar forward e reverso. Em produção, você pode usar um cache mais “flat”, como `/var/cache/bind/slaves/`, desde que os caminhos declarados nas zonas batam com o que o BIND consegue gravar.

> Por que assim?
> - Config fica em `/etc/bind/`
> - Dados do Primary ficam em `/var/lib/bind/` (e as chaves DNSSEC ficam em `.../keys` **dentro de cada zona**)
> - Zonas transferidas (cache do Secondary) ficam em `/var/cache/bind/`
>
> Dica: em `/var/cache/bind/` você pode ver arquivos gerados pelo próprio BIND (ex.: `managed-keys.bind`, `*.jnl`, `named_dump.db`). Isso é normal.

---

<a id="6"></a>
## 6) Logging do BIND (para auditoria + Fail2Ban)

Vamos criar logs em arquivo dentro de `/var/log/named/` para:

- Auditoria (eventos “query denied”, “refused notify”, etc.)
- Fail2Ban (banir abusos automaticamente)
- (Opcional) observar o **RRL** (`rate-limit`) quando você habilitar no passo 7

### 6.1) Criar diretório de logs (ns1 e ns2)

```bash
sudo mkdir -m 775 -p /var/log/named
sudo chown root:bind /var/log/named
```

### 6.2) Ativar bloco `logging {}` no BIND (ns1 e ns2)

Edite `/etc/bind/named.conf` e adicione **antes** dos `include`:

> Dica: aqui é só o **logging**. Não coloque `include "/etc/bind/keys.conf";` no `named.conf` (isso costuma duplicar include no Debian). A inclusão de TSIG fica no `named.conf.local` no passo 8.4.

```conf
logging {
  channel default_file {
    file "/var/log/named/named.log" versions 3 size 30m;
    severity info;
    print-time yes;
    print-severity yes;
    print-category yes;
  };

  channel security_file {
    file "/var/log/named/security.log" versions 3 size 30m;
    severity dynamic;
    print-time yes;
    print-severity yes;
    print-category yes;
  };

  channel ratelimit_file {
    file "/var/log/named/rate-limit.log" versions 3 size 30m;
    severity info;
    print-time yes;
    print-severity yes;
    print-category yes;
  };

  category default { default_file; };
  category general { default_file; };
  category security { security_file; };
  # No BIND, eventos de RRL aparecem em mais de uma categoria:
  # - `rate-limit`: avisos/resumos (início/fim, tabela, etc.)
  # - `query-errors`: pode incluir eventos "slip/drop" por RRL (depende versão/config)
  category rate-limit { ratelimit_file; };
  category query-errors { ratelimit_file; default_file; };
};
```

> Opcional (debug): se você quiser logar **queries** por um período (cuidado: isso pode crescer **muito** rápido), você pode adicionar também:
>
> ```conf
> channel queries_file {
>   file "/var/log/named/queries.log" versions 3 size 200m;
>   severity info;
>   print-time yes;
>   print-severity yes;
>   print-category yes;
> };
>
> category queries { queries_file; };
> ```
>
> Recomendo usar isso só para diagnóstico (e depois desligar), mantendo o `security.log` como log “enxuto” para auditoria/Fail2Ban.

Valide:

```bash
sudo named-checkconf
```

Reinicie:

```bash
sudo systemctl restart named
```

Verifique se os arquivos estão sendo escritos:

```bash
sudo ls -la /var/log/named
sudo tail -n 50 /var/log/named/named.log
```

> Nota: `security.log` tende a crescer quando houver eventos do tipo `denied/refused`. Já `rate-limit.log` aparece principalmente quando você habilita RRL (passo 7) e/ou em alguns tipos de `query-errors`.

> Se aparecer erro de permissão (AppArmor), verifique os logs e ajuste o caminho para um diretório permitido, ou ajuste o perfil. (Isso depende do ambiente.)

---

<a id="7"></a>
## 7) Configuração base do BIND (`named.conf.options`) — ns1 e ns2

Faça backup:

```bash
sudo cp -v /etc/bind/named.conf.options /etc/bind/named.conf.options.bkp.$(date +%F)
```

Edite `/etc/bind/named.conf.options`:

```conf
# (Opcional) ACL usada apenas se você habilitar recursão interna no passo 7.1.
acl "trusted" {
  127.0.0.1;
  ::1;
  192.168.1.0/24;        // ajuste (rede interna)
  2001:db8:1::/64;       // ajuste (rede interna)
};

options {
  directory "/var/cache/bind";

  listen-on port 53 { any; };
  listen-on-v6 port 53 { any; };

  # Autoritativo público: qualquer um consulta suas zonas
  allow-query { any; };

  # Autoritativo "padrão ouro": NÃO faça recursão/caching (evita open resolver)
  recursion no;
  allow-recursion { none; };
  allow-query-cache { none; };

  # Transferência: negada por padrão (liberamos por zona com TSIG)
  allow-transfer { none; };

  # Hardening leve
  minimal-responses yes;

  # (Opcional) Response Rate Limiting (RRL): comece em "log-only" e ajuste antes de dropar respostas.
  # Logs ficam em /var/log/named/rate-limit.log (passo 6).
  # rate-limit {
  #   responses-per-second 50;
  #   window 5;
  #   log-only yes;
  # };
  version "not currently available";

  # Validação DNSSEC é para resolvedor (recursão). Em autoritativo puro, deixe desligado.
  dnssec-validation no;
};
```

> Resumo: com `recursion no;` você tem um **autoritativo puro** (recomendado para servidor público).

### 7.1) (Opcional) Resolver interno: habilitar recursão **somente** para rede interna

Se (e somente se) você precisa que este mesmo servidor resolva nomes para sua rede interna:

1. Garanta que a ACL `trusted` contenha **apenas** suas redes internas (e nada público).
2. No bloco `options {}`, troque estas linhas:

   ```conf
   recursion yes;
   allow-recursion { trusted; };
   allow-query-cache { trusted; };
   dnssec-validation auto;
   ```

3. Faça dois testes:
   - De **dentro** da rede interna: deve resolver (ex.: `dig @SEU_DNS google.com A`)
   - De **fora**: deve dar **REFUSED**

> Dica de arquitetura (padrão ouro): autoritativo e resolvedor separados (ex.: BIND autoritativo + Unbound para recursão), em hosts/VMs/containers diferentes.

Valide e reinicie:

```bash
sudo named-checkconf
sudo systemctl restart named
```

### 7.2) (Opcional) Fazer o próprio servidor usar o resolvedor local (COM CUIDADO)

Isso só faz sentido se você **habilitou recursão** no passo 7.1 (ou se você tem um resolvedor local separado).

> Se seu servidor é **autoritativo puro** (`recursion no;`), **não** aponte o `/etc/resolv.conf` para `127.0.0.1/::1` — senão o próprio servidor não vai conseguir resolver nomes externos (apt, curl, etc.).

#### 7.2.1) Descubra quem “manda” no `/etc/resolv.conf`

```bash
ls -la /etc/resolv.conf
readlink -f /etc/resolv.conf
systemctl is-active systemd-resolved || echo "systemd-resolved inativo ou ausente"
```

#### 7.2.2) Caso A — `/etc/resolv.conf` é arquivo normal (estático)

Faça backup e aponte para o loopback:

```bash
sudo cp -v /etc/resolv.conf /etc/resolv.conf.bkp.$(date +%F)
printf 'nameserver 127.0.0.1\nnameserver ::1\n' | sudo tee /etc/resolv.conf >/dev/null
```

> Dica: se você quer evitar ficar “sem DNS” quando o BIND está parado, adicione um **fallback** (ex.: o DNS do seu roteador/ISP) como terceira linha.

#### 7.2.3) Caso B — `systemd-resolved` está ativo (comum em algumas instalações)

Nesse caso, muitas vezes o `/etc/resolv.conf` é um link para arquivos em `/run/systemd/resolve/` e aponta para o stub `127.0.0.53`.

Você pode manter o stub e fazer o `systemd-resolved` usar o BIND como upstream:

1. Edite `/etc/systemd/resolved.conf` e ajuste:
   ```ini
   [Resolve]
   DNS=127.0.0.1 ::1
   FallbackDNS=
   ```
2. Reinicie e verifique:
   ```bash
   sudo systemctl restart systemd-resolved
   resolvectl status || systemd-resolve --status
   ```

#### 7.2.4) Testes (para saber se o servidor “resolve por ele mesmo”)

1. Teste usando explicitamente o BIND:
   ```bash
   dig @127.0.0.1 google.com A +short
   dig @::1 google.com AAAA +short
   ```
2. Teste usando o resolvedor do sistema (usa `/etc/resolv.conf`):
   ```bash
   dig google.com A +short
   host google.com
   ```
3. Diagnóstico avançado: `+trace` (mostra o caminho de delegação)
   ```bash
   dig google.com A +trace
   ```
   > Observação: `+trace` faz consultas iterativas diretamente (não é um “teste do seu resolvedor”), mas é excelente para entender **onde** uma resolução falha (raiz → TLD → autoritativos).

---

<a id="8"></a>
## 8) TSIG: transferência segura Primary ↔ Secondary (recomendado)

> TSIG autentica transferência/NOTIFY (`AXFR`/`IXFR`). **Não tem relação com DNSSEC.**
>
> Resumo rápido:
> - `AXFR` = cópia completa da zona
> - `IXFR` = só as diferenças desde o último serial
>
> Quando tudo está saudável, o Secondary tende a preferir `IXFR`. Se não der, ele pode cair para `AXFR`.

### 8.1) Criar o arquivo de chaves (NS1)

No **ns1 (Primary)**:

```bash
sudo install -d -m 0750 -o root -g bind /etc/bind
sudo install -m 0640 -o root -g bind /dev/null /etc/bind/keys.conf
```

### 8.2) Gerar a chave TSIG no Primary (ns1)

Verifique qual utilitário está disponível:

```bash
command -v tsig-keygen || command -v ddns-confgen
```

Opção A — usando `tsig-keygen` (se disponível):

```bash
sudo tsig-keygen -a hmac-sha256 xfr-exemplo | sudo tee /etc/bind/keys.conf >/dev/null
```

Opção B — alternativa compatível, mas em modo silencioso:

```bash
sudo ddns-confgen -q -a hmac-sha256 -k xfr-exemplo | sudo tee /etc/bind/keys.conf >/dev/null
```

> No Debian 13 atual, `ddns-confgen` **sem `-q`** inclui blocos de exemplo como `update-policy` e `nsupdate`, e isso quebra o `named-checkconf` quando você grava a saída direto em `/etc/bind/keys.conf`.  
> Por isso, aqui o caminho mais seguro é:
> - preferir `tsig-keygen`, ou
> - usar `ddns-confgen` com `-q`.

Ajuste permissões (se necessário):

```bash
sudo chown root:bind /etc/bind/keys.conf
sudo chmod 0640 /etc/bind/keys.conf
```

> **ATENÇÃO:** Trate este arquivo como uma senha. Ele contém o segredo criptográfico usado para autenticar transferências de zona.

### 8.3) Copiar a chave para o Secondary (ns2)

No **ns1**:

```bash
sudo scp /etc/bind/keys.conf root@203.0.113.20:/etc/bind/keys.conf
```

No **ns2**:

```bash
sudo chown root:bind /etc/bind/keys.conf
sudo chmod 0640 /etc/bind/keys.conf
```

### 8.4) Incluir o arquivo de chaves no BIND (**apenas uma vez**)

No Debian, o `/etc/bind/named.conf` normalmente **inclui** o `named.conf.local`.  
Se você colocar `include "/etc/bind/keys.conf";` nos dois lugares, o BIND vai “ler” o mesmo arquivo duas vezes e você verá erro do tipo:

`key 'xfr-exemplo': already exists previous definition`

**Recomendação (mais simples):** inclua o `keys.conf` **somente** no topo do `/etc/bind/named.conf.local` (ns1 e ns2) e **não** inclua no `named.conf`.

```conf
include "/etc/bind/keys.conf";
```

Como conferir se não ficou duplicado:

```bash
sudo grep -RIn --color=always 'include "/etc/bind/keys.conf";' /etc/bind
```

O esperado é aparecer **uma única** ocorrência.

Valide a configuração:

```bash
sudo named-checkconf
```

### 8.5) Observações importantes

- A chave TSIG deve ser a mesma no Primary e no Secondary.
- Gere a chave apenas no Primary.
- Não publique nem versiona o arquivo `keys.conf`.
- Se o arquivo for perdido, gere uma nova chave e atualize ambos os lados.

### 8.6) (Recomendado) “Amarrar” TSIG para o outro servidor com `server { keys { ... } }`

Além de usar TSIG em `allow-transfer`/`masters`, vale a pena declarar a chave no bloco `server {}`. Isso deixa o NOTIFY/transferência mais consistente e facilita troubleshooting de “bad key” / “notify refused”.

Nós vamos usar isso nos exemplos do `named.conf.local` do ns1 e do ns2.

---

<a id="9"></a>
## 9) Primary (ns1): declarar zonas + criar arquivos de zona

### 9.1) `named.conf.local` (ns1)

Backup:

```bash
sudo cp -v /etc/bind/named.conf.local /etc/bind/named.conf.local.bkp.$(date +%F)
```

Edite `/etc/bind/named.conf.local`:

```conf
// IMPORTANTE: TSIG autentica transferência/NOTIFY (AXFR/IXFR). Não tem relação com DNSSEC.
// Se você perder /etc/bind/keys.conf, gere uma nova chave TSIG e atualize em ns1 e ns2.
include "/etc/bind/keys.conf";

server 203.0.113.20 { keys { "xfr-exemplo"; }; };
server 2001:db8:abcd::20 { keys { "xfr-exemplo"; }; };

zone "exemplo.com.br" {
  type master;
  file "/var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts";

  # Transferência/NOTIFY com TSIG
  allow-transfer { key "xfr-exemplo"; };
  also-notify { 203.0.113.20; 2001:db8:abcd::20; };
  notify yes;

  # DNSSEC (KASP): recomendado começar com o default
  key-directory "/var/lib/bind/master-aut/exemplo.com.br/keys";
  dnssec-policy default;

  # Em versões atuais do BIND, `dnssec-policy` em zona estática depende de inline-signing.
  # Na prática, o policy default já tende a usar isso, mas deixamos explícito para não ficar ambíguo no arquivo.
  inline-signing yes;
};

zone "113.0.203.in-addr.arpa" {
  type master;
  file "/var/lib/bind/master-rev/IPv4/203.0.113.x/203.0.113.rev";

  allow-transfer { key "xfr-exemplo"; };
  also-notify { 203.0.113.20; 2001:db8:abcd::20; };
  notify yes;

  # DNSSEC (KASP): se você também assina este reverso
  # Observação: para ter cadeia de confiança, o DS precisa existir no pai (painel de quem delega o reverso —
  # por exemplo: Registro.br → Numeração, ou seu provedor/ISP/RIR).
  key-directory "/var/lib/bind/master-rev/IPv4/203.0.113.x/keys";
  dnssec-policy default;
  inline-signing yes;
};

zone "d.c.b.a.8.b.d.0.1.0.0.2.ip6.arpa" {
  type master;
  file "/var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/2001.db8.abcd.rev";

  allow-transfer { key "xfr-exemplo"; };
  also-notify { 203.0.113.20; 2001:db8:abcd::20; };
  notify yes;

  # DNSSEC (KASP): se você também assina este reverso
  # Observação: para ter cadeia de confiança, o DS precisa existir no pai (painel de quem delega o reverso —
  # por exemplo: Registro.br → Numeração, ou seu provedor/ISP/RIR).
  key-directory "/var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/keys";
  dnssec-policy default;
  inline-signing yes;
};
```

### 9.2) Criar a zona forward (ns1)

> Importante (DNSSEC com `inline-signing`): **edite sempre o arquivo “fonte”** (no exemplo, `*.hosts` e `*.rev`).  
> O BIND vai gerar automaticamente arquivos como `*.signed` e `*.signed.jnl` no mesmo diretório — **não edite esses**.

```bash
sudo -u bind nano /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts
```

Conteúdo (exemplo):

```zone
$TTL 86400
@   IN SOA  ns1.exemplo.com.br. hostmaster.exemplo.com.br. (
        2026010701 ; serial (AAAAMMDDNN)
        3600       ; refresh
        900        ; retry
        1209600    ; expire
        3600       ; negative cache TTL
)

; Nameservers
@   IN NS   ns1.exemplo.com.br.
@   IN NS   ns2.exemplo.com.br.

; Glue/hosts dos NS (obrigatório existir)
ns1 IN A     203.0.113.10
ns1 IN AAAA  2001:db8:abcd::10
ns2 IN A     203.0.113.20
ns2 IN AAAA  2001:db8:abcd::20

; Site
@   IN A     203.0.113.50
@   IN AAAA  2001:db8:abcd::50
www IN CNAME @

; E-mail (opcional)
mail IN A     203.0.113.60
mail IN AAAA  2001:db8:abcd::60
@    IN MX 10 mail.exemplo.com.br.
@    IN TXT "v=spf1 a mx ~all"
```

### 9.3) Reverso IPv4 (/24) (ns1)

```bash
sudo -u bind nano /var/lib/bind/master-rev/IPv4/203.0.113.x/203.0.113.rev
```

```zone
$TTL 86400
@   IN SOA  ns1.exemplo.com.br. hostmaster.exemplo.com.br. (
        2026010701
        3600
        900
        1209600
        3600
)

@   IN NS   ns1.exemplo.com.br.
@   IN NS   ns2.exemplo.com.br.

10  IN PTR  ns1.exemplo.com.br.
20  IN PTR  ns2.exemplo.com.br.
50  IN PTR  exemplo.com.br.
60  IN PTR  mail.exemplo.com.br.
```

#### 9.3.1) (Opcional) Uso de `$GENERATE` em zonas reversas (IPv4)

##### O que é `$GENERATE`

O `$GENERATE` é uma diretiva do BIND que permite **gerar vários registros automaticamente** na hora de carregar a zona. Ela evita centenas (ou milhares) de linhas repetidas em blocos grandes.

##### Quando faz sentido usar

Use `$GENERATE` quando:

- O PTR segue um padrão previsível (ex.: `203-0-113-<IP>.clientes.exemplo.com.br.`)
- Você tem um bloco grande (/24, /23, etc.)
- Você não precisa de nomes “custom” por IP (ou só precisa em poucos casos)

Casos comuns:

- Pool de clientes (dinâmico)
- CGNAT / grandes pools
- “Faixas” onde o rDNS é só informativo/organizacional

##### Quando NÃO usar

Evite `$GENERATE` para:

- Servidores (mail, web, NS, etc.)
- Infraestrutura crítica (roteadores, firewalls, links, etc.)
- Clientes corporativos com PTR específico

##### Exemplo prático: PTR “genérico” para um pool dentro do /24

Vamos supor que no bloco `203.0.113.0/24` você separou:

- Infra/serviços (PTR explícito): `10`, `20`, `50`, `60`
- Clientes (PTR genérico): `100–254`

Você mantém os PTRs importantes explícitos e gera o resto:

```zone
; =========================================================
; Infra / serviços (PTR explícito)
; =========================================================
10  IN PTR  ns1.exemplo.com.br.
20  IN PTR  ns2.exemplo.com.br.
50  IN PTR  exemplo.com.br.
60  IN PTR  mail.exemplo.com.br.

; =========================================================
; Pool de clientes (PTR genérico)
; =========================================================
$GENERATE 100-254 $ IN PTR 203-0-113-$.clientes.exemplo.com.br.
```

O caractere `$` é substituído pelo número do intervalo. Nesse exemplo, o BIND “expande” como se você tivesse escrito:

```text
100 IN PTR 203-0-113-100.clientes.exemplo.com.br.
101 IN PTR 203-0-113-101.clientes.exemplo.com.br.
...
254 IN PTR 203-0-113-254.clientes.exemplo.com.br.
```

##### Misturar PTR explícito + `$GENERATE` (boa prática)

**Atenção:** o BIND **não “dá prioridade”** para o PTR explícito. O `$GENERATE` é expandido na carga da zona; se você gerar um número que já tem PTR explícito, você pode acabar com **dois PTRs** para o mesmo IP (o que normalmente é indesejado).

Por isso, evite sobreposição de intervalos.

Se você quiser gerar “quase tudo” (`1–254`), mas sem pisar nos IPs da infra, faça em pedaços:

```zone
; Infra explícita
10  IN PTR  ns1.exemplo.com.br.
20  IN PTR  ns2.exemplo.com.br.
50  IN PTR  exemplo.com.br.
60  IN PTR  mail.exemplo.com.br.

; Clientes genéricos (sem sobrepor 10/20/50/60)
$GENERATE 1-9   $ IN PTR 203-0-113-$.clientes.exemplo.com.br.
$GENERATE 11-19 $ IN PTR 203-0-113-$.clientes.exemplo.com.br.
$GENERATE 21-49 $ IN PTR 203-0-113-$.clientes.exemplo.com.br.
$GENERATE 51-59 $ IN PTR 203-0-113-$.clientes.exemplo.com.br.
$GENERATE 61-254 $ IN PTR 203-0-113-$.clientes.exemplo.com.br.
```

##### Boas práticas rápidas

- Evite gerar `.0` e `.255` (em /24, normalmente são rede/broadcast; em ambientes modernos pode variar, mas continua sendo uma convenção útil).
- Use nomes neutros e claros: `clientes`, `dyn`, `pool`.
- Documente no arquivo o que está sendo gerado automaticamente.
- Se você precisa de **FCrDNS** (reverse bate com forward), garanta também os registros forward correspondentes (geralmente só é obrigatório para serviços específicos, como e-mail).

##### Nota sobre IPv6

O BIND também suporta `$GENERATE` em zonas `ip6.arpa`, mas **não é recomendado** usar isso em produção para “clientes em massa”. O espaço de endereçamento do IPv6 é gigantesco; então um `$GENERATE` tende a dar uma **sensação falsa de cobertura**, sem refletir o que está realmente alocado/ativo.

Em IPv6, a prática mais segura e comum é:

- Manter PTRs **explícitos** apenas para infraestrutura e serviços.
- Criar PTRs de clientes **sob demanda** (quando houver necessidade real), em vez de “pré-gerar” faixas inteiras.

### 9.4) Reverso IPv6 (/48) (ns1)

```bash
sudo -u bind nano /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/2001.db8.abcd.rev
```

> Regras rápidas (IPv6 reverse):
> - Expanda o IPv6 até 32 nibbles (hex).
> - Inverta os nibbles.
> - Para /48, o “nome da zona” já consome 12 nibbles (48 bits). Você escreve os 20 nibbles restantes no PTR.

```zone
$TTL 86400
@   IN SOA  ns1.exemplo.com.br. hostmaster.exemplo.com.br. (
        2026010701
        3600
        900
        1209600
        3600
)

@   IN NS   ns1.exemplo.com.br.
@   IN NS   ns2.exemplo.com.br.

; 2001:db8:abcd::10 -> ...:0010
0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR ns1.exemplo.com.br.

; 2001:db8:abcd::20 -> ...:0020
0.2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR ns2.exemplo.com.br.

; 2001:db8:abcd::50 -> ...:0050
0.5.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR exemplo.com.br.
```

### 9.5) Validar e reiniciar (ns1)

```bash
sudo named-checkconf
sudo named-checkzone exemplo.com.br /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts
sudo named-checkzone 113.0.203.in-addr.arpa /var/lib/bind/master-rev/IPv4/203.0.113.x/203.0.113.rev
sudo named-checkzone d.c.b.a.8.b.d.0.1.0.0.2.ip6.arpa /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/2001.db8.abcd.rev
sudo systemctl restart named
```

Opcional: confira os arquivos gerados (inclui `*.signed` e `*.jnl`):

```bash
sudo -u bind ls -la /var/lib/bind/master-aut/exemplo.com.br/
sudo -u bind ls -la /var/lib/bind/master-rev/IPv4/203.0.113.x/
sudo -u bind ls -la /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/
```

---

<a id="10"></a>
## 10) Secondary (ns2): declarar zonas e transferir do Primary

### 10.1) `named.conf.local` (ns2)

Backup:

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bkp.$(date +%F)
```

Edite `/etc/bind/named.conf.local`:

```conf
// IMPORTANTE: TSIG autentica transferência/NOTIFY (AXFR/IXFR). Não tem relação com DNSSEC.
// Se você perder /etc/bind/keys.conf, gere uma nova chave TSIG e atualize em ns1 e ns2.
include "/etc/bind/keys.conf";

server 203.0.113.10 { keys { "xfr-exemplo"; }; };
server 2001:db8:abcd::10 { keys { "xfr-exemplo"; }; };

zone "exemplo.com.br" {
  type slave;
  file "/var/cache/bind/slave-aut/exemplo.com.br/exemplo.com.br.hosts.signed";
  masters { 203.0.113.10 key "xfr-exemplo"; 2001:db8:abcd::10 key "xfr-exemplo"; };
  allow-notify { 203.0.113.10; 2001:db8:abcd::10; };
};

zone "113.0.203.in-addr.arpa" {
  type slave;
  file "/var/cache/bind/slave-rev/IPv4/203.0.113.x/203.0.113.rev.signed";
  masters { 203.0.113.10 key "xfr-exemplo"; 2001:db8:abcd::10 key "xfr-exemplo"; };
  allow-notify { 203.0.113.10; 2001:db8:abcd::10; };
};

zone "d.c.b.a.8.b.d.0.1.0.0.2.ip6.arpa" {
  type slave;
  file "/var/cache/bind/slave-rev/IPv6/2001.db8.abcd.x/2001.db8.abcd.rev.signed";
  masters { 203.0.113.10 key "xfr-exemplo"; 2001:db8:abcd::10 key "xfr-exemplo"; };
  allow-notify { 203.0.113.10; 2001:db8:abcd::10; };
};
```

Valide e reinicie:

```bash
sudo named-checkconf
sudo systemctl restart named
```

### 10.2) Verificar se a transferência aconteceu

No ns2:

```bash
sudo -u bind ls -la /var/cache/bind/slave-aut/exemplo.com.br/
sudo -u bind ls -la /var/cache/bind/slave-rev/IPv4/203.0.113.x/
sudo -u bind ls -la /var/cache/bind/slave-rev/IPv6/2001.db8.abcd.x/
```

> É normal ver arquivos `*.jnl` (journal) e, em zonas com DNSSEC/inline-signing, arquivos `*.signed` e `*.signed.jnl`.

Opcional (se você quer ver a árvore de diretórios no estilo do seu `tree`):

```bash
sudo apt install -y tree
sudo tree /var/cache/bind -L 4
```

Teste com dig local:

```bash
dig @127.0.0.1 exemplo.com.br SOA +noall +answer
dig @127.0.0.1 exemplo.com.br NS  +noall +answer
```

Se não transferir, acompanhe logs:

```bash
sudo journalctl -u named -n 200 --no-pager
```

---

<a id="11"></a>
## 11) Registro.br: publicar delegação (NS/glue) e DNSSEC (DS)

Agora que o ns1 e ns2 já estão configurados e respondendo, chegou a hora de **publicar a delegação**:

- **Domínio (forward)**: você publica os **NS** (e o **glue**, se necessário) no **Registro.br**.
- **Reverso (numeração)**: você publica os **NS** (e, se usar DNSSEC, o **DS**) no painel de **quem delega o reverso** — por exemplo: **Registro.br → Numeração** (quando o bloco está sob seus recursos) ou o painel do seu provedor/ISP/RIR.

> Importante: você pode publicar a delegação (NS/glue) **antes** de publicar DNSSEC (DS).  
> DNSSEC só entra em jogo quando existe **DS publicado no pai**.

### 11.1) Antes de mexer no Registro.br: checklist mínimo (evita dor de cabeça)

Antes de apontar o domínio, confirme:

1. O ns1 e ns2 respondem autoritativamente pelo domínio.
2. UDP/53 **e** TCP/53 estão acessíveis externamente.
3. Recursão está desligada (se este é um autoritativo público).

Testes (faça a partir de uma rede “de fora”):

```bash
# Forward (SOA) em cada servidor
dig @203.0.113.10 exemplo.com.br SOA +noall +answer
dig @203.0.113.20 exemplo.com.br SOA +noall +answer

# IPv6 (se aplicável)
dig @2001:db8:abcd::10 exemplo.com.br SOA +noall +answer
dig @2001:db8:abcd::20 exemplo.com.br SOA +noall +answer

# Teste negativo (autoritativo puro): deve dar REFUSED
dig @203.0.113.10 google.com A +noall +comments +answer
```

### 11.2) Publicar NS (e glue, se necessário) do domínio no Registro.br

1. Acesse o **painel do Registro.br** e selecione o domínio (ex.: `exemplo.com.br`).
2. Vá na parte de **DNS / Servidores DNS** e defina seus servidores autoritativos:
   - `ns1.exemplo.com.br`
   - `ns2.exemplo.com.br`
3. Se o Registro.br pedir “IP do host ns1/ns2”, isso é o **glue**. Preencha:
   - IPv4 do ns1/ns2
   - IPv6 do ns1/ns2 (se você usa IPv6)

> Regra prática sobre glue:
> - Se seus nameservers são `ns1.exemplo.com.br` e `ns2.exemplo.com.br` (dentro do próprio domínio), glue é obrigatório.
> - Se seus nameservers estiverem em outro domínio (ex.: `ns1.provedor.net`), normalmente não precisa glue.

### 11.3) Validar a delegação do domínio (antes de pensar em DNSSEC)

1. Use a ferramenta do Registro.br: **Verificação de DNS**.
2. Use `dig +trace` para ver se a delegação está indo para ns1/ns2:

```bash
dig exemplo.com.br NS +trace
```

> `+trace` é ótimo para diagnosticar “onde parou” (raiz → TLD → autoritativos). Ele não depende do seu resolvedor local.

### 11.4) (Se aplicável) Publicar reverso no Registro.br → Numeração (DNS e DNSSEC)

Se os seus blocos (IPv4/IPv6) estão sob seus recursos no Registro.br, você consegue delegar o reverso no painel de **Numeração**.

1. No painel, abra **Numeração** e selecione o prefixo (ex.: `203.0.113.0/24`).
2. Em **DNS**, configure os NS do reverso:
   - `ns1.exemplo.com.br`
   - `ns2.exemplo.com.br`
3. Se você assina o reverso com DNSSEC e o painel oferecer **DNSSEC/DS**, publique o DS do reverso ali (passo 11.7).

Teste depois de publicar:

```bash
dig -x 203.0.113.10 @1.1.1.1 +noall +answer
dig -x 2001:db8:abcd::10 @1.1.1.1 +noall +answer
```

### 11.5) DNSSEC: o que o `dnssec-policy default` faz

- Gera chaves automaticamente
- Assina a zona automaticamente
- Mantém assinaturas/rollover conforme política

> Isso exige que o BIND tenha **permissão de escrita** na zona e no `key-directory`.

### 11.6) DNSSEC: aguardar geração de chaves (ns1)

Após reiniciar, aguarde 1–3 minutos e liste:

```bash
sudo -u bind sh -c 'cd / && ls -1 /var/lib/bind/master-aut/exemplo.com.br/keys/'
```

Identifique qual delas é **KSK** (a própria `.key` costuma ter comentário indicando):

```bash
sudo -u bind sh -c 'cd / && find /var/lib/bind/master-aut/exemplo.com.br/keys -name "*.key" -exec grep -Hi "key-signing key" {} +'
```

> Dica: o `cd /` evita erro quando você roda o comando a partir de um diretório que o usuário `bind` não consegue acessar, como `/root` ou `/home/seu_usuario`.

### 11.7) DNSSEC: gerar o DS (SHA-256) a partir da KSK

Use o arquivo KSK encontrado e gere DS com SHA-256:

```bash
# Exemplo: se sua KSK for "Kexemplo.com.br.+013+12345.key",
# passe o caminho SEM a extensão ".key":
sudo -u bind dnssec-dsfromkey -2 /var/lib/bind/master-aut/exemplo.com.br/keys/Kexemplo.com.br.+013+12345
```

> Se aparecer erro procurando `...key.key`, passe o caminho **sem** `.key` (como no exemplo acima).

Opcional (mais seguro, sem “adivinhar” o ID da KSK):

```bash
sudo -u bind bash -c '
cd /
set -e
KSK_KEY=$(grep -l "key-signing key" /var/lib/bind/master-aut/exemplo.com.br/keys/*.key | head -n 1)
echo "KSK encontrada: $KSK_KEY"
'
```

Depois, use o caminho retornado acima no `dnssec-dsfromkey`:

```bash
sudo -u bind sh -c 'cd / && dnssec-dsfromkey -2 /var/lib/bind/master-aut/exemplo.com.br/keys/Kexemplo.com.br.+013+12345'
```

Se você também assina **reverso** com DNSSEC, gere o DS do reverso do mesmo jeito (um DS por zona):

```bash
# Reverso IPv4 (/24) — exemplo
sudo -u bind bash -c '
cd /
set -e
KSK_KEY=$(grep -l "key-signing key" /var/lib/bind/master-rev/IPv4/203.0.113.x/keys/*.key | head -n 1)
echo "KSK reverso IPv4: $KSK_KEY"
'

# Reverso IPv6 (/48) — exemplo
sudo -u bind bash -c '
cd /
set -e
KSK_KEY=$(grep -l "key-signing key" /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/keys/*.key | head -n 1)
echo "KSK reverso IPv6: $KSK_KEY"
'
```

Depois, repita o mesmo padrão usando o caminho da KSK de cada reverso.

Saída (exemplo):

```text
exemplo.com.br. IN DS 12345 13 2 ABCDEF...
```

Campos:

- `12345` = Key Tag
- `13` = Algoritmo da chave (pode variar)
- `2` = Digest Type (SHA-256)
- `ABCDEF...` = Digest

### 11.8) DNSSEC: publicar o DS (domínio e, se aplicável, reverso)

1. **Domínio (forward)**: no Registro.br, abra a área de **DNSSEC** do domínio e cole os campos do DS:
   - Key Tag
   - Algoritmo
   - Digest Type
   - Digest
2. **Reverso (se aplicável)**:
   - se o reverso é delegado pelo Registro.br (Numeração), publique o DS do reverso na área de **DNSSEC** do prefixo;
   - se o reverso é delegado pelo seu provedor/ISP/RIR, publique o DS no painel deles (se suportarem).
3. Valide com a ferramenta de **Verificação de DS** do Registro.br (domínio) e/ou com validadores como DNSViz.

---

<a id="12"></a>
## 12) Fail2Ban: bloquear abuso automaticamente (ns1 e ns2)

### 12.0) Debian 13: preciso instalar `rsyslog` por causa do Fail2Ban?

**Na maioria dos casos, não.**

- No Debian 13, é comum o sistema usar **systemd-journald** como fonte principal de logs (e o `rsyslog` pode nem vir instalado por padrão).
- O Fail2Ban consegue ler logs de duas formas:
  1. **Arquivos de log** (`logpath = /var/log/...`) — que é o que usamos aqui com o BIND (`/var/log/named/security.log`).
  2. **Journald** (`backend = systemd`) — útil para jails como `sshd` em sistemas sem `auth.log`.

Como aqui o BIND grava o log **direto em arquivo** (via `logging { file ... }`), **você não precisa instalar `rsyslog`** para a jail do BIND funcionar.

> Se você quiser usar Fail2Ban também para **SSH** no Debian 13 sem `rsyslog`, a abordagem recomendada é configurar o jail do `sshd` com `backend = systemd` (jornald), em vez de instalar rsyslog só por causa disso.

### 12.1) Garantir que o BIND está gerando eventos “denied”

Um teste simples: faça uma consulta **recursiva** (para um nome que seu servidor não é autoritativo), por exemplo:

```bash
dig @SEU_IP_PUBLICO google.com A
```

O esperado é **REFUSED** (ou timeout se o firewall bloquear).

- Se você seguiu o perfil padrão (autoritativo puro), **qualquer origem** deve receber `REFUSED`.
- Se você habilitou a recursão interna (passo 7.1), teste também a partir de fora da ACL `trusted` (deve ser `REFUSED`) e de dentro (deve resolver).

Se aparecer evento no log:

```bash
sudo tail -n 50 /var/log/named/security.log
```

> Observação importante: em Debian 13 limpo, um `dig @SEU_IP google.com A` retornando `REFUSED` **não garante**, por si só, que o `security.log` já terá linha útil para o Fail2Ban.  
> Antes de habilitar a jail `named-refused`, confirme com `fail2ban-regex` que o log real do seu ambiente está casando com o filtro.

### 12.2) Usar filtro do Fail2Ban (preferência: `named-refused`)

No Debian, geralmente existe o filtro:

```bash
ls -1 /etc/fail2ban/filter.d/named-refused.conf
```

Se existir, use ele.
Crie `/etc/fail2ban/jail.d/named-refused.local`:

```bash
sudo nano /etc/fail2ban/jail.d/named-refused.local
```

> Atenção: **não** é para substituir `/etc/fail2ban/filter.d/named-refused.conf`.  
> Esse arquivo é o **filtro** (regex). O bloco abaixo é a **jail** (configuração), e deve ficar em `jail.d/`.

```ini
[named-refused]
enabled = true
port = 53
logpath = /var/log/named/security.log
maxretry = 3
findtime = 10m
bantime = 12h

# Se você usa nftables (recomendado)
banaction = nftables-multiport
```

> Se no seu sistema não existir a ação `nftables-multiport`, use a variação genérica:
> `banaction = nftables[type=multiport]`

Reinicie:

```bash
sudo systemctl restart fail2ban
sudo systemctl status fail2ban --no-pager
sudo fail2ban-client status named-refused
```

> Se o `fail2ban-client` reclamar de socket (`/var/run/fail2ban/fail2ban.sock`), é porque o serviço não subiu.  
> Veja o erro completo com: `sudo journalctl -u fail2ban -n 200 --no-pager`.

### 12.3) Validar regex do filtro (boa prática)

```bash
sudo fail2ban-regex /var/log/named/security.log /etc/fail2ban/filter.d/named-refused.conf
```

> Importante: filtros mal escritos podem ser explorados (DoS de regex). Evite criar regex “genérica demais”.

### 12.4) `ignoreip` / whitelist: boas práticas para não se auto-banir (FAÇA ANTES DE ALLPORTS)

O Fail2Ban tem um “whitelist” simples chamado `ignoreip`. Tudo que estiver nele **nunca** será banido.

Recomendação: configurar isso em um arquivo global, para valer para todas as jails:

Crie `/etc/fail2ban/jail.d/00-local-defaults.local`:

```ini
[DEFAULT]
# Lista de IPs/redes que nunca serão banidos.
# Para adicionar seus IPs/redes de administração, edite APENAS a linha `ignoreip` abaixo e acrescente no final (separando por espaço).
ignoreip = 127.0.0.1/8 ::1 198.51.100.10/32 192.168.1.0/24 2001:db8:1::/64
```

Checklist para não quebrar o Fail2Ban:

- Não copie as “crases” de Markdown (linhas começando com ```).
- Garanta que `[DEFAULT]` começa na **coluna 1** (sem espaços/tabs antes).
- Use IPs separados por **espaço** (evite vírgulas).
- Use **apenas uma** linha `ignoreip = ...` (não repita `ignoreip` no mesmo arquivo/section). Para adicionar IPs, continue na **mesma** linha.

Depois:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client get named-refused ignoreip
```

Se o Fail2Ban não subir após editar esse arquivo, veja o erro completo e corrija a linha indicada:

```bash
sudo journalctl -u fail2ban -n 200 --no-pager
sudo nl -ba /etc/fail2ban/jail.d/00-local-defaults.local | sed -n '1,80p'
```

**Prática recomendada (bem real):**

- Coloque o IP da sua VPN/bastion/adm no `ignoreip` **antes** de usar allports.
- Se você perceber que algum IP legítimo (ex.: ferramentas de validação do registro, monitoramento do provedor, etc.) está sendo banido, adicione ao `ignoreip` com base no que aparece em:
  ```bash
  sudo tail -n 200 /var/log/fail2ban.log
  sudo fail2ban-client status named-refused
  ```

**E sobre colocar IPs/prefixos do Registro.br no `ignoreip`?**

- Não é recomendado sair colocando “prefixos grandes” por padrão (isso vira um “super-whitelist” e reduz sua proteção).
- Se você notar que as ferramentas do Registro.br (Verificação de DNS/DS) estão sendo banidas pelo `named-refused`, o caminho mais seguro é: **whitelist mínima**, só com os IPs que você viu sendo banidos.

Fluxo prático:

1. Rode a verificação no Registro.br.
2. Veja se teve ban naquele horário:
   ```bash
   sudo grep -i "Ban " /var/log/fail2ban.log | tail -n 20
   sudo fail2ban-client status named-refused
   ```
3. Se for um IP legítimo e você quiser “fixar” como permitido, acrescente esse IP no `ignoreip` (passo 12.4) e reinicie o Fail2Ban.

Se você realmente quiser “pré-liberar” as redes do NIC.br/Registro.br (opcional e mais permissivo), você pode acrescentar na **mesma linha** do `ignoreip`:

```ini
# Exemplo (adicione no final da linha ignoreip, sem duplicar a opção):
# ignoreip = 127.0.0.1/8 ::1 ... 200.160.0.0/20 200.219.148.0/24 2001:12f8:6::/47 2001:12ff::/32
```

> Observação: esses prefixos são grandes (principalmente IPv6). Use somente se você aceitar o trade-off de reduzir a área protegida pelo Fail2Ban.

Para desbanir manualmente:

```bash
sudo fail2ban-client set named-refused unbanip SEU_IP_AQUI
```

### 12.5) Modo “mais bruto”: banir **allports** e usar **DROP**

Por padrão, a configuração acima bloqueia o IP atacante **somente na porta 53** (DNS). Isso já reduz bastante varreduras.

Se você quer ser mais agressivo (e banir o IP no servidor inteiro), você pode usar **allports**, que bloqueia **todas as portas** (incluindo SSH, HTTP, etc.) para aquele IP.

> Atenção: se você ativar allports sem configurar o `ignoreip` (passo 12.4), você pode se trancar para fora do servidor.

**Quando isso é uma boa ideia:**

- Seu servidor sofre varredura pesada e você quer cortar ruído rapidamente.
- Você tem acesso out-of-band (console/IPMI) ou está seguro de que seu IP de administração está em `ignoreip` (passo 12.4).

**Como ativar :**

1. Edite `/etc/fail2ban/jail.d/named-refused.local` e troque a linha do `banaction` por:
   ```ini
   banaction = nftables[type=allports, blocktype=drop]
   ```

2. Reinicie e valide:
   ```bash
   sudo systemctl restart fail2ban
   sudo systemctl status fail2ban --no-pager
   sudo fail2ban-client status named-refused
   sudo nft list ruleset | grep -i f2b || echo "nenhuma regra f2b apareceu no ruleset"
   ```

**Por que usar `nftables[type=allports]` e não `nftables-allports`?**

- Em muitos sistemas existe o arquivo `/etc/fail2ban/action.d/nftables-allports.conf`, mas ele costuma ser só um “atalho” (wrapper) e pode estar marcado como **obsoleto**.
- A forma mais clara (e que evita confusão) é usar a ação genérica `nftables[...]` passando o parâmetro `type=allports`.

> Alternativa (se você preferir usar o atalho): `banaction = nftables-allports[blocktype=drop]`.

> Dica: antes de mudar para allports em produção, recomenda-se reduzir temporariamente o `bantime` (ex.: `10m`) e testar. Assim, se ocorrer banimento por engano, a recuperação é mais simples.


### 12.6) Recidive: **allports** só para reincidentes (Opcional, bem útil)

Se você quer evitar falso-positivo “caro” (allports), uma estratégia bem prática é:

- `named-refused`: bloqueia **só DNS (53)** por um tempo razoável.
- `recidive`: se o mesmo IP for banido várias vezes em um período, aí sim ele entra em banimento **longo** e/ou **allports**.

Crie `/etc/fail2ban/jail.d/recidive.local`:

```ini
[recidive]
enabled = true
logpath = /var/log/fail2ban.log
findtime = 1d
maxretry = 3
bantime = 7d

# Ação mais agressiva só aqui (reincidentes)
banaction = nftables[type=allports, blocktype=drop]
```

Reinicie:

```bash
sudo systemctl restart fail2ban
sudo systemctl status fail2ban --no-pager
sudo fail2ban-client status recidive
```

---

<a id="13"></a>
## 13) (Opcional) DoH (DNS over HTTPS) — use com MUITA clareza de objetivo

**Quando faz sentido**: oferecer recursão **para rede interna** via HTTPS.

**Quando NÃO faz sentido**: “colocar DoH” em servidor autoritativo público “só por colocar”.

### 13.1) Pré-requisitos

- Certificado TLS válido para um hostname (ex.: `doh.exemplo.com.br`).
- Você vai escutar na porta 443 **apenas no IP interno**, ou atrás de um LB interno.

### 13.2) Configuração (no servidor que fará recursão interna)

No `/etc/bind/named.conf.options`, adicione (fora do `options {}`):

```conf
tls local-tls {
  cert-file "/etc/letsencrypt/live/doh.exemplo.com.br/fullchain.pem";
  key-file  "/etc/letsencrypt/live/doh.exemplo.com.br/privkey.pem";
};

http local-http {
  endpoints { "/dns-query"; };
};
```

E dentro de `options {}`:

```conf
listen-on port 443 tls local-tls http local-http { 127.0.0.1; 192.168.1.10; };
listen-on-v6 port 443 tls local-tls http local-http { ::1; 2001:db8:1::10; };
```

Abra 443 no `nftables` **só para a rede interna**.

### 13.3) Teste DoH

Se seu `dig` suportar:

```bash
dig +https @192.168.1.10 exemplo.com.br A
```

Alternativa robusta: `kdig` (Knot DNS utils):

```bash
sudo apt install -y knot-dnsutils
kdig +https @192.168.1.10 exemplo.com.br A
```

---

<a id="14"></a>
## 14) Checklist de validação final (antes de publicar)

### 14.1) Autoridade e consistência

No ns1 e ns2:

```bash
dig @127.0.0.1 exemplo.com.br SOA +noall +answer
dig @127.0.0.1 exemplo.com.br NS  +noall +answer
```

De fora (apontando para cada NS):

```bash
dig @203.0.113.10 exemplo.com.br SOA +noall +answer
dig @203.0.113.20 exemplo.com.br SOA +noall +answer
```

### 14.2) Reverso IPv4/IPv6

```bash
dig @203.0.113.10 -x 203.0.113.10 +noall +answer
dig @203.0.113.10 -x 2001:db8:abcd::10 +noall +answer
```

### 14.3) DNSSEC

No ns1:

```bash
dig @127.0.0.1 exemplo.com.br DNSKEY +noall +answer
dig @127.0.0.1 exemplo.com.br A +dnssec +noall +answer
```

Depois de publicar DS no Registro.br (e aguardar propagação), teste com um validador:

```bash
dig @1.1.1.1 exemplo.com.br A +dnssec +multi
```

Use DNSViz para diagnóstico visual.

### 14.4) Fail2Ban

```bash
sudo fail2ban-client status named-refused
sudo tail -n 50 /var/log/fail2ban.log
```

### 14.5) Teste negativo: garantir que sua zona NÃO vaza via AXFR

O objetivo aqui é confirmar que **só o Secondary autorizado** consegue transferir zona — e que qualquer outro host recebe “transfer failed”.

1. De uma máquina que **não** seja o ns2 (e que **não** tenha a TSIG), rode:
   ```bash
   dig @203.0.113.10 exemplo.com.br AXFR
   dig @203.0.113.20 exemplo.com.br AXFR
   ```
   Resultado esperado: falhar (ex.: `Transfer failed.`).

2. (Opcional) Do ns2 (que tem a TSIG), valide que a transferência é possível **com assinatura TSIG**:
   ```bash
   dig -k /etc/bind/keys.conf @203.0.113.10 exemplo.com.br AXFR | head
   ```

Se o passo (1) der certo e imprimir a zona inteira, você está com **vazamento de zona** e precisa revisar `allow-transfer` (deve ser TSIG, não `any;`).

---

<a id="15"></a>
## 15) Troubleshooting (curto e prático)

### 15.1) Serviço não sobe

```bash
sudo named-checkconf
sudo journalctl -u named -n 200 --no-pager
```

### 15.2) Secondary não transfere

- Confira TCP/53 liberado entre ns1↔ns2.
- Confirme a mesma TSIG em `/etc/bind/keys.conf` nos dois lados.
- Veja logs:
  ```bash
  sudo journalctl -u named -n 200 --no-pager
  ```

### 15.3) DNSSEC dá SERVFAIL após publicar DS

- DS no Registro.br não bate com a KSK atual → gere DS novamente com `dnssec-dsfromkey -2`.
- Relógio errado → `timedatectl`.

### 15.4) Aviso sobre `ixfr-from-differences` em zona assinada

Se no `journalctl -u named` aparecer algo como:

```text
'ixfr-from-differences' is ignored for inline-signed zones
```

isso costuma ser só um aviso informativo quando a zona usa `dnssec-policy` com `inline-signing yes`.

Na prática:

- o BIND vai ignorar `ixfr-from-differences` nessa zona;
- isso não significa que a zona está quebrada;
- se você quiser evitar o aviso, remova `ixfr-from-differences yes` da zona afetada.

### 15.5) AppArmor bloqueando o log `/var/log/named/security.log`

Sintoma típico: o `named` não sobe (ou sobe sem log) e o journal mostra algo como `apparmor="DENIED"` ao tentar escrever em `/var/log/named/security.log`.

1. Verifique se o AppArmor está ativo e se o perfil do `named` está carregado:
   ```bash
   sudo aa-status | grep -i named || sudo apparmor_status | grep -i named
   ```
   Se `aa-status` não existir:
   ```bash
   sudo apt install -y apparmor-utils
   ```

2. Veja as negações:
   ```bash
   sudo journalctl -k | grep -i apparmor | tail -n 50
   ```

3. Garanta que o diretório exista com dono/permissões adequados (isso resolve a maioria dos casos):
   ```bash
   sudo mkdir -m 775 -p /var/log/named
   sudo chown root:bind /var/log/named
   sudo systemctl restart named
   ```

4. Se ainda negar, use o “override local” do perfil (forma mais limpa de ajustar sem editar o profile principal):
   ```bash
   sudo install -o root -g root -m 0644 /dev/null /etc/apparmor.d/local/usr.sbin.named
   sudo nano /etc/apparmor.d/local/usr.sbin.named
   ```
   Adicione:
   ```text
   /var/log/named/ rw,
   /var/log/named/** rw,
   ```

5. Recarregue o perfil e reinicie o serviço:
   ```bash
   sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named
   sudo systemctl restart named
   ```

---

<a id="16"></a>
## 16) Cenário de desastre (DR): reconstruindo um NS perdido sem quebrar DNSSEC

Se você perder o **ns1 (Primary)** mas o **ns2 (Secondary)** ainda estiver respondendo, você tem uma “janela” até o campo **`expire`** do SOA estourar. A prioridade é:

1. **Manter o domínio resolvendo** (mesmo que sem DNSSEC por um período).
2. Recuperar a capacidade de **editar zona** com segurança (voltar a ter um Primary).
3. Recolocar **DNSSEC corretamente** (DS compatível com as chaves em produção).

### 16.1) Checklist rápido (o que você ainda tem?)

Antes de mexer em qualquer coisa, responda:

- Você tem backup dos **arquivos de zona “fonte”** (sem DNSSEC), ex.: `/var/lib/bind/master-aut/*/*.hosts` e `/var/lib/bind/master-rev/**`?
- Você tem backup da chave **TSIG** (`/etc/bind/keys.conf`)?
- Você tem backup das chaves DNSSEC (**KASP**) por zona, ex.: `/var/lib/bind/master-aut/*/keys` e `/var/lib/bind/master-rev/**/keys`?
- Existe **DS publicado** no Registro.br para o domínio? (use a ferramenta de Verificação de DS)

### 16.2) Caso A — você tem backup das chaves DNSSEC (melhor cenário)

Aqui a regra é simples: **restaure as chaves e mantenha o DS** (não precisa trocar DS se as chaves são as mesmas).

1. Reinstale/suba o ns1 seguindo o guia normalmente (passos 3 a 10).
2. Restaure os `key-directory` no ns1 (exemplo; adapte ao seu backup). Aqui nós guardamos chaves **dentro de cada zona**, então você restaura “por pasta”:
   ```bash
   # Forward
   sudo install -d -o bind -g bind -m 0750 /var/lib/bind/master-aut/exemplo.com.br/keys
   sudo cp -a /CAMINHO_DO_BACKUP/master-aut/exemplo.com.br/keys/. /var/lib/bind/master-aut/exemplo.com.br/keys/

   # Reverse IPv4
   sudo install -d -o bind -g bind -m 0750 /var/lib/bind/master-rev/IPv4/203.0.113.x/keys
   sudo cp -a /CAMINHO_DO_BACKUP/master-rev/IPv4/203.0.113.x/keys/. /var/lib/bind/master-rev/IPv4/203.0.113.x/keys/

   # Reverse IPv6
   sudo install -d -o bind -g bind -m 0750 /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/keys
   sudo cp -a /CAMINHO_DO_BACKUP/master-rev/IPv6/2001.db8.abcd.x/keys/. /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/keys/

   sudo chown -R bind:bind /var/lib/bind/master-aut /var/lib/bind/master-rev
   sudo chmod -R 750 /var/lib/bind/master-aut /var/lib/bind/master-rev
   ```
   > Se você preferir `rsync` (mais robusto para backups), aplique a mesma lógica por pasta `.../keys`.
3. Confirme que as zonas no `named.conf.local` ainda têm:
   - `key-directory "/var/lib/bind/master-aut/exemplo.com.br/keys";`
   - `key-directory "/var/lib/bind/master-rev/IPv4/203.0.113.x/keys";`
   - `key-directory "/var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/keys";`
   - `dnssec-policy default;`
   - `inline-signing yes;`
4. Reinicie e valide DNSSEC (passo 14.3 e DNSViz).

### 16.3) Caso B — você NÃO tem as chaves DNSSEC (cenário comum)

Se você perdeu as chaves, o **DS antigo vira uma armadilha**: validadores vão considerar a zona **BOGUS** até você remover/atualizar o DS.

#### 16.3.1) Primeiro, desative DNSSEC no Registro.br (remova o DS)

1. No Registro.br, remova o **DS** (desabilite DNSSEC) para o domínio.
2. Valide com “Verificação de DS” do Registro.br (passo 11.8).

> Por alguns minutos/horas, alguns resolvedores ainda podem ter o DS antigo em cache. É normal a propagação levar um tempo.

#### 16.3.2) Suba o DNS SEM DNSSEC primeiro (zona “limpa”)

No ns1 (e, se necessário, no ns2 enquanto você normaliza o ambiente):

1. No `named.conf.local`, **comente temporariamente** em cada zona:
   - `key-directory ...`
   - `dnssec-policy ...`
   - `inline-signing ...`
2. Reinicie o BIND e valide autoridade (passo 14.1).

#### 16.3.3) Recupere os arquivos de zona

**Caminho recomendado (melhor):** restaurar do seu backup/controle de versão os arquivos de zona “fonte”.

**Se você só tem o ns2 como fonte:** copie as zonas do Secondary e use como base (cuidado com zonas assinadas).

No ns2, copie as zonas para um lugar seguro:

```bash
sudo install -d -m 0700 /root/dr-zones
sudo cp -a /var/cache/bind/slave-aut/exemplo.com.br/exemplo.com.br.hosts.signed /root/dr-zones/
sudo cp -a /var/cache/bind/slave-rev/IPv4/203.0.113.x/203.0.113.rev.signed /root/dr-zones/
sudo cp -a /var/cache/bind/slave-rev/IPv6/2001.db8.abcd.x/2001.db8.abcd.rev.signed /root/dr-zones/
ls -la /root/dr-zones
```

Copie para o ns1 (exemplo com `scp`):

```bash
scp /root/dr-zones/* root@203.0.113.10:/root/dr-zones/
```

No ns1, use isso como **referência** e reconstrua os arquivos “fonte”, ou restaure do seu backup.

```bash
sudo install -d -o bind -g bind -m 0750 /var/lib/bind/master-aut/exemplo.com.br/keys
sudo install -d -o bind -g bind -m 0750 /var/lib/bind/master-rev/IPv4/203.0.113.x/keys
sudo install -d -o bind -g bind -m 0750 /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/keys

# Exemplo (se você tiver os arquivos "fonte" do backup)
# sudo cp -a /CAMINHO_DO_BACKUP/exemplo.com.br.hosts /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts
# sudo cp -a /CAMINHO_DO_BACKUP/203.0.113.rev       /var/lib/bind/master-rev/IPv4/203.0.113.x/203.0.113.rev
# sudo cp -a /CAMINHO_DO_BACKUP/2001.db8.abcd.rev   /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/2001.db8.abcd.rev

sudo chown -R bind:bind /var/lib/bind/master-aut /var/lib/bind/master-rev
```

Antes de subir o serviço no ns1, é recomendável validar **em lote** (isso reduz o risco de “subir quebrado”):

```bash
# 1) Sintaxe do BIND (sempre)
sudo named-checkconf

# 2) (Recomendado) Checa as zonas que estão declaradas na configuração
# (carrega os arquivos apontados no named.conf.local)
sudo named-checkconf -z

# 3) (Opcional) Checagem explícita dos arquivos "fonte" (ajuda em troubleshooting)
while read -r ZONE FILE; do
  echo "== $ZONE ($FILE) =="
  sudo named-checkzone "$ZONE" "$FILE" || exit 1
done <<'EOF'
exemplo.com.br /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts
113.0.203.in-addr.arpa /var/lib/bind/master-rev/IPv4/203.0.113.x/203.0.113.rev
d.c.b.a.8.b.d.0.1.0.0.2.ip6.arpa /var/lib/bind/master-rev/IPv6/2001.db8.abcd.x/2001.db8.abcd.rev
EOF
```

Se tudo passar, aí sim reinicie:

```bash
sudo systemctl restart named
```

> Atenção: se o seu Primary usava `dnssec-policy`/`inline-signing`, é comum o Secondary ter uma **zona já assinada** (arquivo `*.signed`, cheia de `RRSIG`, `DNSKEY`, etc.).  
> Nesse caso, **trate como referência**: não use como “fonte” se você perdeu as chaves. Reconstrua a zona “limpa” (somente os registros que você administra) e volte com DNSSEC depois.

#### 16.3.4) Volte com DNSSEC (gerar novo DS)

Depois que tudo estiver estável (autoridade + transferência + serial correto):

1. Reative `dnssec-policy`/`inline-signing` nas zonas do ns1.
2. Reinicie e gere o **novo DS** (passos 11.6 e 11.7).
3. Publique o DS novo no Registro.br (passo 11.8).

---

<a id="17"></a>
## 17) Referências (fontes para consulta)

### RFCs (IETF)

- DNS (RFC 1035): https://www.ietf.org/rfc/rfc1035.txt
- TSIG (RFC 2845): https://datatracker.ietf.org/doc/html/rfc2845
- NOTIFY (RFC 1996): https://datatracker.ietf.org/doc/html/rfc1996
- IXFR (RFC 1995): https://datatracker.ietf.org/doc/html/rfc1995
- AXFR (RFC 5936): https://datatracker.ietf.org/doc/html/rfc5936
- DNSSEC (RFC 4033/4034/4035): https://datatracker.ietf.org/doc/html/rfc4033 • https://datatracker.ietf.org/doc/html/rfc4034 • https://datatracker.ietf.org/doc/html/rfc4035
- DoH (RFC 8484): https://datatracker.ietf.org/doc/html/rfc8484
- Reverso classless IPv4 (RFC 2317): https://datatracker.ietf.org/doc/html/rfc2317

### BIND / ISC (oficial)

- BIND 9 Reference: https://bind9.readthedocs.io/en/latest/reference.html
- Categorias de log (`rate-limit`, `query-errors`, etc.): https://bind9.readthedocs.io/en/v9.20.6/reference/logging-categories.html
- Nameserver Basics (autoritativo vs resolvedor recursivo): https://kb.isc.org/docs/aa-00817
- TSIG — `server { keys { ... } }` (Security Configurations): https://bind9.readthedocs.io/en/v9.18.30/chapter7.html
- `$GENERATE` (BIND primary file extension): https://bind9.readthedocs.io/en/v9.18.14/chapter3.html#bind-primary-file-extension-the-generate-directive
- DNSSEC Guide (dnssec-policy): https://bind9.readthedocs.io/en/stable/dnssec-guide.html
- DNSSEC policy (ISC KB): https://kb.isc.org/docs/dnssec-key-and-signing-policy
- Response Rate Limiting (RRL): https://kb.isc.org/docs/aa-01148
- Usando RRL (inclui modo “log-only”): https://kb.isc.org/docs/aa-00994
- DoH no BIND (ISC blog): https://www.isc.org/blogs/doh-talkdns/
- `named-checkconf` (manpage Debian): https://manpages.debian.org/trixie/bind9-utils/named-checkconf.1.en.html
- `named-checkzone` (manpage Debian): https://manpages.debian.org/trixie/bind9-utils/named-checkzone.1.en.html

### Registro.br

- Portal: https://registro.br/
- Whois (IP/ASN): https://registro.br/tecnologia/ferramentas/whois/
- Verificação de DNS: https://registro.br/tecnologia/ferramentas/verificacao-de-dns/
- Verificação de DS: https://registro.br/tecnologia/ferramentas/verificacao-de-ds/
- Numeração (reverso): https://registro.br/tecnologia/numeracao/suporte/

### Ferramentas

- DNSViz (DNSSEC): https://dnsviz.net/
- Root hints (InterNIC): https://www.internic.net/domain/named.root
- Gerador de PTR IPv6: http://rdns6.com/hostRecord

### Fail2Ban (referência)

- Manual do `jail.conf` (backends, `systemd`/journald, etc.): https://manpages.debian.org/bookworm/fail2ban/jail.conf.5.en.html
- (Debian 13/Trixie) Discussão sobre `rsyslog` não vir instalado por padrão: https://lists.debian.org/debian-user/2025/02/msg00456.html

### AppArmor (referência)

- Debian Wiki (BIND9) — recarregar profile com `apparmor_parser`: https://wiki.debian.org/BIND9
- Perfil `usr.sbin.named` (comentários sobre `/var/lib/bind`, `/var/cache/bind` e `/var/log/named`): https://bugs-devel.debian.org/904983
- Caso prático: logs em `/var/log/named` e permissões: https://serverfault.com/questions/989969/debian-after-update-bind9-cannot-create-log-in-var-log-bind

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
