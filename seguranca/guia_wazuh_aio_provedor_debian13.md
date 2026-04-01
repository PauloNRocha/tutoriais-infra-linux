# Guia de Produção: Wazuh AIO para provedores no Debian 13

*Criado em: 18 de dezembro de 2025*  
*Última atualização em: 01 de abril de 2026*

Wazuh em arquitetura All-in-One pode resolver bem quando a operação ainda está concentrada, mas ele precisa nascer com alguma folga e com o mínimo de organização para não virar dor de cabeça rápido. Este guia registra esse desenho no **Debian 13**, pensando em provedor pequeno ou médio, com múltiplos servidores Linux e integração com **MikroTik** via syslog.

O foco aqui é instalação, pós-instalação essencial, agentes, retenção, backup e um mínimo de endurecimento com `nftables`. Os exemplos abaixo seguem a linha do **Wazuh 4.14.4**.

---

## Índice rápido
1. [Dimensionamento rápido](#1)
2. [Preparação do Debian 13 (Trixie) minimal](#2)
3. [Decisões de Segurança Adotadas](#3)
4. [Instalação Wazuh 4.14.4 (All-in-One)](#4)
5. [Pós-instalação essencial](#5)
6. [Configurando Agentes](#6)
7. [Agente Debian 12/13](#6.1)
8. [Agente AlmaLinux + cPanel](#6.2)
9. [Integração MikroTik (Syslog)](#7)
10. [Retenção de logs (90 dias – ISM/ILM)](#8)
11. [Backup diário simples](#9)
12. [Saúde e troubleshooting](#10)
13. [Checklist final](#11)
14. [Notas de produção](#12)
15. [Referências](#13)

---

<a id="1"></a>
## 1. Dimensionamento rápido

Dimensionar corretamente é o primeiro passo para o sucesso. Considere estes números:

-   **Hardware mínimo AIO (1–25 agentes):** 4 vCPU, 8 GB RAM, 50 GB SSD.
-   **Para este cenário (~15 agentes):** recomendação de **6–8 vCPU, 12–16 GB RAM, 150 GB NVMe** para ter conforto e folga.
-   **Indexer (por nó):** A recomendação oficial é 16 GB RAM / 8 cores; o mínimo aceitável é 4 GB / 2 cores.
-   **Armazenamento (90 dias de retenção para o Indexer):**
    -   Servidor/agente Linux: ~3,7 GB
    -   Dispositivo de rede: ~7,4 GB
    -   Exemplo: 15 servidores → 55,5 GB; 4 MikroTik → 29,6 GB. Total ~85 GB. Adicione 20% de folga, chegando a aproximadamente **105 GB** somente para os índices.

---

<a id="2"></a>
## 2. Preparação do Debian 13 (Trixie) minimal

Vamos preparar o sistema operacional para receber o Wazuh, garantindo estabilidade e segurança.

1.  **Atualize base e instale utilitários essenciais:**
    ```bash
    sudo apt update && sudo apt full-upgrade -y
    # No Debian 13, o APT já suporta HTTPS nativamente (não precisa instalar apt-transport-https).
    sudo apt install -y curl wget gnupg ca-certificates lsb-release nftables sudo vim htop net-tools rsync
    ```
2.  **Defina Hostname e edite hosts:**
    ```bash
    sudo hostnamectl set-hostname wazuh-isp
    echo "127.0.1.1 wazuh-isp" | sudo tee -a /etc/hosts
    ```
3.  **Configure o firewall (nftables) — simples, stateful e “server-first”:**

    Este modelo segue o padrão de hardening adotado nos outros guias do repositório:
    - política padrão **DROP** no `input`;
    - firewall **stateful** (aceita conexões estabelecidas/relacionadas);
    - libera apenas o necessário por **rede de origem** (melhor para ISP do que “abrir portas para o mundo”).
    
    Veja também: [Guia de Produção: Acesso SSH por chave pública (porta customizada) em servidores Linux](../acesso-remoto/guia_producao_ssh_chave_publica_linux.md)

    3.1) Habilite o serviço:
    ```bash
    sudo systemctl enable --now nftables
    ```

    3.2) Faça backup do ruleset atual:
    ```bash
    sudo cp /etc/nftables.conf "/etc/nftables.conf.bak.$(date +%F_%H%M%S)"
    ```

    3.3) Aplique um ruleset base (AJUSTE as redes/IPs):
    ```bash
    sudo tee /etc/nftables.conf >/dev/null <<'NFT'
    #!/usr/sbin/nft -f

    flush ruleset

    table inet filter {
      # =========================
      # AJUSTE AQUI (OBRIGATÓRIO)
      # =========================
      #
      # - mgmt_*: redes que podem acessar SSH e o Dashboard
      # - agents_*: redes onde estão os servidores que rodam wazuh-agent (endpoints)
      # - mikrotik_*: IPs dos roteadores que enviarão syslog
      #
      # Boas práticas:
      # - use ranges o mais restritos possível
      # - prefira /32 para hosts críticos (ex.: MikroTik)

      set mgmt_v4 {
        type ipv4_addr; flags interval;
        elements = { 10.0.0.0/8, 192.168.0.0/16 }
      }

      set agents_v4 {
        type ipv4_addr; flags interval;
        elements = { 10.0.0.0/8, 192.168.0.0/16 }
      }

      set mikrotik_v4 {
        type ipv4_addr; flags interval;
        elements = { 192.168.0.1/32 }
      }

      # Se você usa IPv6 na rede interna, ajuste e habilite também
      set mgmt_v6 {
        type ipv6_addr; flags interval;
        elements = { fd00::/8 }
      }

      set agents_v6 {
        type ipv6_addr; flags interval;
        elements = { fd00::/8 }
      }

      set mikrotik_v6 {
        type ipv6_addr; flags interval;
        elements = { fd00::1/128 }
      }

      chain input {
        type filter hook input priority 0; policy drop;

        # Básico
        iif "lo" accept
        ct state invalid drop
        ct state established,related accept

        # ICMP (diagnóstico de rede)
        ip protocol icmp accept
        ip6 nexthdr ipv6-icmp accept

        # =========================
        # ACESSO ADMINISTRATIVO
        # =========================

        # SSH (recomendado: restrito à rede de gestão)
        tcp dport 22 ip saddr @mgmt_v4 accept
        tcp dport 22 ip6 saddr @mgmt_v6 accept

        # Dashboard (HTTPS)
        # Observação: este guia usa 443/tcp. Se no seu ambiente o Dashboard estiver em outra porta, ajuste aqui.
        tcp dport 443 ip saddr @mgmt_v4 accept
        tcp dport 443 ip6 saddr @mgmt_v6 accept

        # =========================
        # AGENTES (ENDPOINTS)
        # =========================

        # Comunicação do agente com o Manager (eventos)
        tcp dport 1514 ip saddr @agents_v4 accept
        tcp dport 1514 ip6 saddr @agents_v6 accept

        # Registro de agentes (recomendado: abrir somente durante onboarding e/ou restringir ainda mais)
        tcp dport 1515 ip saddr @agents_v4 accept
        tcp dport 1515 ip6 saddr @agents_v6 accept
        # Boas práticas (produção ISP): após concluir o onboarding inicial, remova/comente as regras de 1515/tcp
        # e recarregue o ruleset no passo 3.4.

        # =========================
        # SYSLOG (MIKROTIK)
        # =========================
        udp dport 514 ip saddr @mikrotik_v4 accept
        udp dport 514 ip6 saddr @mikrotik_v6 accept
      }

      chain output {
        type filter hook output priority 0; policy accept;
      }
    }
    NFT
    ```

    3.4) Carregue e valide:
    ```bash
    sudo nft -f /etc/nftables.conf
    sudo nft list ruleset
    ```
4.  **Ajuste Sysctl e limites:**
    ```bash
    sudo cat >/etc/sysctl.d/90-wazuh.conf <<'EOF'
    vm.max_map_count=262144
    vm.swappiness=10
    fs.file-max=2097152
    net.core.somaxconn=1024
    net.ipv4.tcp_max_syn_backlog=2048
    EOF
    sudo sysctl -p /etc/sysctl.d/90-wazuh.conf
    ```
    Adicione ao final de `/etc/security/limits.conf`:
    ```
    * soft nofile 65536
    * hard nofile 65536
    * soft nproc 65536
    * hard nproc 65536
    ```
5.  **Configure Swap (se sua RAM for < 16 GB):**
    ```bash
    sudo fallocate -l 4G /swapfile
    sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
    ```

---

<a id="3"></a>
## 3. Decisões de Segurança Adotadas

Este guia não foca apenas na instalação, mas em uma implementação segura desde o início. As principais decisões de hardening aplicadas foram:

-   **Firewall por padrão (DROP) e stateful:** O `nftables` aplica política padrão de negar tráfego de entrada, aceitando `established,related` e liberando apenas portas estritamente necessárias.
-   **Restrição de portas de gerenciamento:** SSH e Dashboard ficam restritos às redes de gestão (`mgmt_v4/mgmt_v6` no `nftables.conf`).
-   **Restrição do onboarding de agentes:** a porta `1515/tcp` deve ser aberta apenas durante o onboarding e/ou restrita a redes específicas (`agents_v4/agents_v6` no `nftables.conf`).
-   **Comunicação Interna Segura:** A porta `55000/tcp` (Filebeat para Indexer) não foi aberta no firewall, pois em uma arquitetura AIO a comunicação é local e não deve ser exposta externamente.
-   **Restrição de Syslog por IP:** A recepção de logs syslog na porta `514/udp` deve ser restrita aos IPs dos roteadores na configuração do Wazuh (`ossec.conf`).

---

<a id="4"></a>
## 4. Instalação Wazuh 4.14.4 (All-in-One)

No Debian 13, o instalador oficial ainda pode mostrar aviso de que o sistema não está na lista de plataformas recomendadas. Mesmo assim, com os recursos mínimos atendidos, o fluxo AIO pode seguir normalmente.

```bash
cd /root
# Baixe o script de instalação oficial
sudo curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo sed -i 's/software-properties-common//g' wazuh-install.sh
sudo chmod +x wazuh-install.sh
# Inicie a instalação All-in-One (Manager + Indexer + Dashboard + Filebeat)
sudo ./wazuh-install.sh -a
```

Observações práticas sobre esse instalador no Debian 13:

- ele atualmente baixa a série `4.14.4`
- ainda mostra aviso de sistema não recomendado
- exige pelo menos `4 GB RAM` e `2 vCPU`; abaixo disso ele aborta antes de instalar

Se o host estiver abaixo disso e você quiser continuar mesmo assim, o próprio script sugere `-i`. Para produção, o melhor é respeitar os requisitos.
Ao final, **salve as senhas!** Elas são cruciais.
```bash
sudo tar -O -xf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```
Anote as senhas do `admin` do Indexer e do `wazuh-wui` (Dashboard).

Verifique se todos os serviços estão ativos:
```bash
systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

---

<a id="5"></a>
## 5. Pós-instalação essencial

-   **Heap do Indexer:** A regra de ouro é definir aproximadamente **50% da RAM** do seu servidor para o Indexer, sem nunca ultrapassar 32GB. Por exemplo, para um servidor com 12GB de RAM, usaríamos 6GB (`6g`):
    ```bash
    sudo sed -i 's/^-Xms.*$/-Xms6g/; s/^-Xmx.*$/-Xmx6g/' /etc/wazuh-indexer/jvm.options
    sudo systemctl restart wazuh-indexer
    ```
-   **Proteção da configuração:** Faça um backup inicial das configurações.
    ```bash
    sudo mkdir -p /backup/wazuh-conf
    sudo tar -czf /backup/wazuh-conf/instalacao_inicial_$(date +%F).tgz /var/ossec/etc /etc/wazuh-indexer /etc/wazuh-dashboard /etc/filebeat
    ```

---

<a id="6"></a>
## 6. Configurando Agentes

**IMPORTANTE (arquitetura Wazuh 4.x):**

- O **`wazuh-agent` é SOMENTE para servidores/hosts remotos** que serão monitorados (endpoints).
- O host do **Wazuh All-in-One / Wazuh Manager NÃO deve instalar `wazuh-agent`**.
- Se você tentar instalar `wazuh-agent` no mesmo host do Manager, o conflito de pacotes é **comportamento esperado** (os pacotes entram em conflito porque compartilham componentes/caminhos).  
  Exemplo de erro típico (Debian/Ubuntu): o `apt`/`dpkg` acusa que `wazuh-agent` **conflita** com `wazuh-manager` e a instalação não prossegue.
- O Wazuh Manager já consegue coletar logs **do próprio host** (coleta local), então instalar um agent ali não é necessário.

Por quê isso importa em produção ISP?
- Evita “quebrar” um AIO funcionando por tentar instalar um componente que **não é para esse papel**.
- Mantém a arquitetura alinhada com o desenho oficial (agentes ↔ servidor Wazuh).

<a id="6.1"></a>
### 6.1 Debian 12/13

Execute no **servidor a ser monitorado (servidor remoto)**.  
**Não execute** no host do Wazuh All-in-One / Wazuh Manager.

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

Boas práticas para nomes de grupo (exemplo):
- sem espaços
- sem acentos
- lowercase
- exemplo: `servers-linux`

RECOMENDADO (primeiro deploy): use o mesmo padrão do Dashboard em **“Deploy new agent”**  
(download do `.deb` + `dpkg -i`). Isso evita cenários onde o método via APT não aplica corretamente
as variáveis no `postinst` e deixa o placeholder `MANAGER_IP` no `ossec.conf` (o agente não inicia).

IMPORTANTE: ajuste a URL/arquivo conforme seu ambiente (versão e arquitetura).
```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.4-1_amd64.deb && \
sudo env WAZUH_MANAGER="<IP_DO_MANAGER>" WAZUH_AGENT_GROUP="<NOME_DO_GRUPO>" WAZUH_AGENT_NAME="<NOME_DO_HOST>" \
dpkg -i ./wazuh-agent_4.14.4-1_amd64.deb
```

Se o `dpkg` reclamar de dependências, corrija e finalize:
```bash
sudo apt -y -f install
sudo dpkg -i ./wazuh-agent_4.14.4-1_amd64.deb
```

Atualizações futuras:
- manter o repositório APT acima faz sentido para atualizar o agente depois;
- se você preferir instalar via APT desde o início, use este formato para garantir que
  as variáveis cheguem ao `postinst` (em vez de `VAR=... sudo apt install ...`):
  ```bash
  sudo env WAZUH_MANAGER="<IP_DO_MANAGER>" WAZUH_AGENT_GROUP="<NOME_DO_GRUPO>" WAZUH_AGENT_NAME="<NOME_DO_HOST>" \
  apt install -y wazuh-agent
  ```

Validação rápida (evita o problema do placeholder):
```bash
sudo grep -nE 'MANAGER_IP|address' /var/ossec/etc/ossec.conf
```

Por fim, habilite/inicie o agente:
```bash
sudo systemctl enable --now wazuh-agent
```

<a id="6.2"></a>
### 6.2 AlmaLinux + cPanel (ignorar virtfs)

Execute no **servidor AlmaLinux/cPanel a ser monitorado (servidor remoto)**.  
**Não execute** no host do Wazuh All-in-One / Wazuh Manager.

```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo cat >/etc/yum.repos.d/wazuh.repo <<'EOF'
[wazuh]
name=Wazuh repository
baseurl=https://packages.wazuh.com/4.x/yum/
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
EOF

# RECOMENDADO (primeiro deploy): padrão do Dashboard em "Deploy new agent"
# (instalação via RPM direto). Use `sudo env ...` para garantir que as variáveis cheguem ao postinst.
#
# IMPORTANTE: ajuste versão e arquitetura conforme seu ambiente.
sudo env WAZUH_MANAGER="<IP_DO_MANAGER>" WAZUH_AGENT_GROUP="<NOME_DO_GRUPO>" WAZUH_AGENT_NAME="<NOME_DO_HOST>" \
dnf install -y https://packages.wazuh.com/4.x/yum/wazuh-agent-4.14.2-1.x86_64.rpm

# (Opcional) Se preferir instalar do repositório (sem URL do RPM), use este formato (evita perder variáveis):
# sudo env WAZUH_MANAGER="<IP_DO_MANAGER>" WAZUH_AGENT_GROUP="<NOME_DO_GRUPO>" WAZUH_AGENT_NAME="<NOME_DO_HOST>" \
# dnf install -y wazuh-agent

# Validação rápida (evita o problema do placeholder):
# Se aparecer "MANAGER_IP" aqui, o agente pode não iniciar até você corrigir.
# sudo grep -nE 'MANAGER_IP|address' /var/ossec/etc/ossec.conf
```
Edite `/var/ossec/etc/ossec.conf` no agente para ignorar `virtfs`:
```xml
<syscheck>
  <ignore>/home/virtfs</ignore>
  <ignore>/proc</ignore>
  <ignore>/sys</ignore>
</syscheck>
```
Depois habilite/inicie o agente:
```bash
sudo systemctl enable --now wazuh-agent
```
Reinicie o agente: `sudo systemctl restart wazuh-agent`.

---

<a id="7"></a>
## 7. Integração MikroTik (Syslog)

<a id="7.1"></a>
### 7.1 Configuração no MikroTik

Os comandos abaixo foram validados em **RouterOS v7.20.8**.  
Se o seu MikroTik tiver mais de um IP/interface e você quer garantir que o syslog sempre “saia” por um IP fixo, use `src-address`.

```
/system logging action add name=wazuh target=remote remote=<IP_WAZUH> remote-port=514 remote-protocol=udp syslog-time-format=bsd-syslog remote-log-format=syslog
/system logging add topics=info,warning,error,critical,firewall,account action=wazuh
```

Aqui, `syslog-time-format=bsd-syslog` só define o formato tradicional do timestamp enviado no syslog, o que costuma facilitar a leitura e o parser do lado do Wazuh. Já `remote-log-format=syslog` força o envio em syslog completo, o que simplifica a recepção no manager.

Validação rápida no RouterOS (recomendado antes de ir para produção):
```
/system logging action print detail where name="wazuh"
/system logging print
```

Exemplo com `src-address` (opcional):
```
/system logging action set [find where name="wazuh"] src-address=<IP_DO_MIKROTIK>
```
> **Dica:** Para uma visibilidade ainda maior, você pode adicionar os tópicos `interface`, `route` e `bgp` à lista.

<a id="7.2"></a>
### 7.2 Configuração no Wazuh Manager
Edite `/var/ossec/etc/ossec.conf` no seu **servidor Wazuh**:
```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips><IP_MIKROTIK>/32</allowed-ips>
</remote>
```
Para esta integração, não é obrigatório criar decoder customizado. Com o MikroTik enviando syslog completo, o Wazuh já consegue pré-decodificar o cabeçalho e você pode ficar só com uma regra local simples.

Adicione em `/var/ossec/etc/rules/local_rules.xml`:
```xml
<group name="mikrotik,">
  <rule id="110100" level="5">
    <hostname type="pcre2">^MikroTik$</hostname>
    <match type="pcre2">^[A-Za-z0-9_.-]+,[A-Za-z0-9_.-]+ .+</match>
    <description>MikroTik evento geral</description>
  </rule>
</group>
```
Reinicie o manager: `sudo systemctl restart wazuh-manager`.

Valide se o manager ficou ouvindo em `514/udp`:
```bash
sudo ss -lunp | grep ':514 '
```

Se você quiser testar a regra antes de depender do roteador em produção, use o `wazuh-logtest` com um exemplo no formato que o RouterOS envia:
```bash
sudo /var/ossec/bin/wazuh-logtest
```

Cole uma linha como esta:
```text
Apr  1 17:55:29 MikroTik script,info teste de syslog
```

Para mensagens de autenticação, o próprio Wazuh já costuma reconhecer padrões como `login failure` e `authentication failed` pela regra `2501` (`syslog: User authentication failure`). Exemplo para teste:
```text
Apr  1 17:55:29 MikroTik account,error login failure for admin from 192.168.1.2 via ssh
```

---

<a id="8"></a>
## 8. ⏳ Retenção de logs (90 dias – ISM/ILM)

```bash
curl -k -u admin:<SENHA_INDEXER_ADMIN> -XPUT "https://localhost:9200/_plugins/_ism/policies/wazuh-90d" \
  -H 'Content-Type: application/json' -d @- <<'JSON'
{
  "policy": {
    "description": "Retencao 90 dias Wazuh",
    "default_state": "hot",
    "states": [
      { "name": "hot", "actions": [], "transitions": [
          { "state_name": "delete", "conditions": { "min_index_age": "90d" } } 
      ]},
      { "name": "delete", "actions": [ { "delete": {} } ], "transitions": []}
    ]
  }
}
JSON
```
Associe esta política aos índices `wazuh-alerts-*` via Dashboard (Management → Index Management).

---

<a id="9"></a>
## 9. Backup diário simples

Crie o script `/usr/local/bin/wazuh-backup.sh`:
```bash
#!/bin/bash
DEST=/backup/wazuh; mkdir -p "$DEST"
DATE=$(date +%Y%m%d_%H%M%S)
tar -czf "$DEST/wazuh_cfg_$DATE.tgz" /var/ossec/etc /etc/wazuh-indexer /etc/wazuh-dashboard /etc/filebeat
find "$DEST" -name "wazuh_cfg_*.tgz" -mtime +30 -delete
echo "Backup concluído em $DEST/wazuh_cfg_$DATE.tgz"
```
Deixe o script executável:
```bash
sudo chmod +x /usr/local/bin/wazuh-backup.sh
```

Crie um service `oneshot`:
```bash
sudo nano /etc/systemd/system/wazuh-backup.service
```

Conteúdo:
```ini
[Unit]
Description=Backup diário simples do Wazuh
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/wazuh-backup.sh
```

Crie o timer:
```bash
sudo nano /etc/systemd/system/wazuh-backup.timer
```

Conteúdo:
```ini
[Unit]
Description=Timer diário do backup do Wazuh

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=30m

[Install]
WantedBy=timers.target
```

Ative:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-backup.timer
systemctl list-timers | grep wazuh-backup
```

---

<a id="10"></a>
## 10. Saúde e troubleshooting

Comandos rápidos de diagnóstico:
```bash
# Status dos serviços
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard filebeat

# Saúde do cluster Indexer
curl -k -u admin:<SENHA_INDEXER_ADMIN> https://localhost:9200/_cluster/health?pretty

# Agentes conectados e ativos
/var/ossec/bin/agent_control -l

# Teste de regras de alerta
/var/ossec/bin/wazuh-logtest

# Uso de disco pelo Indexer
df -h /var/lib/wazuh-indexer
```

---

<a id="11"></a>
## 11. Checklist final

-   [ ] Debian 13 minimal hardenizado
-   [ ] Wazuh AIO instalado e serviços ativos
-   [ ] Senhas do Indexer e Dashboard guardadas com segurança
-   [ ] Heap do Indexer ajustado
-   [ ] Agentes Debian e AlmaLinux registrados e reportando
-   [ ] Logs MikroTik chegando e gerando alertas
-   [ ] Política de retenção de 90 dias aplicada
-   [ ] Backup diário configurado e testado

---

<a id="12"></a>
## 12. Notas de produção

-   Promova atualizações de versão do Wazuh (ex: 4.14.x) somente após realizar snapshots/backups completos do servidor.
-   Se a base de agentes crescer (>80–100 agentes), planeje a migração para uma arquitetura de cluster distribuído.
-   Priorize versões **estáveis** (stable) e evite RC/beta em produção.

---

<a id="13"></a>
## 13) Referências (fontes para consulta)

### Wazuh (oficial)

- Quickstart: https://documentation.wazuh.com/current/quickstart.html
- Portal de instalação: https://documentation.wazuh.com/current/installation-guide/index.html
- Assistente de instalação (`wazuh-install.sh`): https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/installation-assistant.html
- Arquitetura e portas padrão (4.x): https://documentation.wazuh.com/current/getting-started/components/wazuh-server.html
- Requisitos e portas (firewall): https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/step-by-step.html
- Repositório no GitHub: https://github.com/wazuh/wazuh

### Debian (referências rápidas)

- `apt-transport-https` (quando é necessário): https://manpages.debian.org/apt-transport-https

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
