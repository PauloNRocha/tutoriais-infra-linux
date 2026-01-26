# Configuração de Rede – Debian 12 (Ethernet)

*Criado em 29 de setembro de 2025 e atualizado em 08 de dezembro de 2025*

Neste guia, vamos configurar interfaces de rede com fio no **Debian 12 (Bookworm)** de forma estável e previsível, usando `ifupdown` e nomes de interface consistentes. Ele funciona bem para servidores e desktops. Se você usa Desktop com NetworkManager, veja a seção 4.

Veja também: [Configuração de Wi‑Fi – Debian 12](../rede/guia_configurar_wifi_debian12.md).

---

## Índice rápido
1. [Pré-requisitos e Escopo](#1)
2. [Começo Rápido (DHCP)](#2)
3. [Instalar ferramentas necessárias](#3)
4. [Identificar a interface e MAC](#4)
5. [Configurar Ethernet para subir no boot (ifupdown)](#5)
6. [NetworkManager (nmcli) — alternativa ao ifupdown](#6)
7. [Desativar Wake-on-LAN (WOL)](#7)
8. [(Opcional) Renomear a interface para `eth0`](#8)
9. [Checklist pós-reboot](#9)
10. [Resumo](#10)
11. [Avançado (opcional)](#11)
12. [Solução de Problemas](#12)

---

<a id="1"></a>
## Pré-requisitos e Escopo

- Acesso com privilégios de administrador (`sudo`).  
- Ambiente alvo: servidores/notebooks com interface Ethernet.  
- Escolha um gerenciador de rede por host (não misture na mesma interface):
  - `ifupdown` (padrão deste guia) → simples, estável, arquivo `/etc/network/interfaces`.
  - `NetworkManager` (NM) → preferível em desktops; há seção alternativa com `nmcli`.
  - `systemd-networkd` → opção moderna para servidores; seção opcional ao final.

Recomendação: para servidor, usar `ifupdown` ou `systemd-networkd`. Para desktop, `NetworkManager`.

---

<a id="2"></a>
## Começo Rápido (DHCP com `ifupdown`)

Para quem precisa apenas conectar a rede com fio via DHCP rapidamente.

1.  **Identifique a interface:**
    ```bash
    ip -br link
    # Anote o nome, ex: enp2s0
    ```

2.  **Edite o arquivo de interfaces:**
    ```bash
    sudo nano /etc/network/interfaces
    ```

3.  **Adicione o bloco abaixo (substitua `enp2s0` se necessário):**
    ```ini
    auto enp2s0
    iface enp2s0 inet dhcp
    ```

4.  **Suba a interface:**
    ```bash
    sudo ifup enp2s0
    ```

5.  **Checklist de verificação rápida:**
    - **Ver se pegou IP:** `ip -br a`
    - **Testar conectividade com a internet:** `ping -c 3 8.8.8.8`
    - **Se algo falhou, veja os logs:** `journalctl -u networking.service -n 50 --no-pager`

Se a conexão via DHCP funcionar, prossiga para o guia completo abaixo (IP fixo, WOL, etc.).

---

<a id="3"></a>
## 1. Instalar ferramentas necessárias

```bash
sudo apt update
sudo apt install -y ifupdown net-tools ethtool
```

- `ifupdown` → gerencia interfaces via `/etc/network/interfaces`.  
- `net-tools` → utilitários clássicos (`ifconfig`, `netstat`).  
- `ethtool` → opções da placa (WOL, velocidade/duplex).

---

<a id="4"></a>
## 2. Identificar a interface e MAC

```bash
ip -br link
```

Exemplo de saída:

```
enp2s0           DOWN   00:1a:2b:3c:4d:5e
```

Interface: `enp2s0`  
MAC: `00:1a:2b:3c:4d:5e`

Para detalhes:
```bash
ip link show enp2s0
```

---

<a id="5"></a>
## 3. Configurar Ethernet para subir no boot (ifupdown)

Este guia assume que a interface será gerenciada pelo `ifupdown` (não pelo NetworkManager).

1) Faça backup e edite o arquivo:
```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
sudo nano /etc/network/interfaces
```

2) Escolha UMA das opções abaixo (ajuste `enp2s0` e o IP conforme necessário):

### Opção A — DHCP (IP via servidor)
```ini
auto enp2s0
iface enp2s0 inet dhcp
```

### Opção B — IP fixo (recomendado para servidor)
```ini
auto enp2s0
iface enp2s0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 1.1.1.1 8.8.8.8
```
```

3) Aplicar sem reboot:
```bash
sudo ifdown enp2s0 || true
sudo ifup enp2s0
```

💡 Usa Desktop com NetworkManager? Ou desative o gerenciamento dessa interface pelo NM, ou configure tudo via `nmcli` (ex.: `nmcli con mod ...`). Misturar NM e ifupdown na mesma interface causa conflito.

### IPv6 com ifupdown

- SLAAC (autoconfiguração via RA):
```ini
iface enp2s0 inet6 auto
```

- DHCPv6:
```ini
iface enp2s0 inet6 dhcp
```

- IPv6 estático:
```ini
iface enp2s0 inet6 static
    address 2001:db8:1234::10
    netmask 64
    gateway 2001:db8:1234::1
    dns-nameservers 2606:4700:4700::1111 2606:4700:4700::1001
```

### Rotas estáticas (IPv4/IPv6)

Exemplos com `post-up`/`pre-down`:
```ini
iface enp2s0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    # rota estática extra
    post-up ip route add 10.10.0.0/16 via 192.168.1.254 dev enp2s0
    pre-down ip route del 10.10.0.0/16 via 192.168.1.254 dev enp2s0 || true

iface enp2s0 inet6 static
    address 2001:db8:1234::10
    netmask 64
    gateway 2001:db8:1234::1
    # rota IPv6 extra
    post-up ip -6 route add 2001:db8:abcd::/48 via 2001:db8:1234::fe dev enp2s0
    pre-down ip -6 route del 2001:db8:abcd::/48 via 2001:db8:1234::fe dev enp2s0 || true
```

### Reiniciar serviço (cautela)

```bash
sudo systemctl restart networking
```
Preferir `ifdown/ifup` da interface específica quando possível, para evitar queda geral de rede.

---

<a id="6"></a>
## 4. 🧭 NetworkManager (nmcli) — alternativa ao ifupdown

Use esta opção em ambientes Desktop ou quando já utiliza NM. Não misture NM e ifupdown na mesma interface.

### DHCP (IPv4) + SLAAC (IPv6)
```bash
nmcli con add type ethernet ifname enp2s0 con-name lan-dhcp \
  ipv4.method auto ipv6.method auto
nmcli con up lan-dhcp
```

### IP estático (IPv4) e DNS
```bash
nmcli con add type ethernet ifname enp2s0 con-name lan-static \
  ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1 8.8.8.8" ipv4.method manual ipv6.method ignore
nmcli con up lan-static
```
 
### Ajustes comuns
```bash
# conectar automaticamente e ajustar métrica da rota
nmcli con mod lan-static connection.autoconnect yes ipv4.route-metric 100

# MTU (ex.: jumbo frames)
nmcli con mod lan-static 802-3-ethernet.mtu 9000

# Evitar que o NM altere WOL (combine com regra udev de WOL desligado)
nmcli con mod lan-static 802-3-ethernet.wake-on-lan ignore
```

### Tornar interface “unmanaged” pelo NM (persistente)
Edite o arquivo de configuração:
```ini
sudo nano /etc/NetworkManager/NetworkManager.conf
```
Conteúdo (exemplo):
```
[keyfile]
unmanaged-devices=interface-name:enp2s0
```
Depois reinicie o NM: `sudo systemctl restart NetworkManager`.

---

<a id="7"></a>
## 5. 🛑 Desativar Wake-on-LAN (WOL)

Verificar WOL atual:
```bash
sudo ethtool enp2s0 | grep Wake-on
```

Desativar (sessão atual):
```bash
sudo ethtool -s enp2s0 wol d
```

Tornar persistente com udev (recomendado):
```bash
sudo nano /etc/udev/rules.d/70-disable-wol.rules
```
Conteúdo (substitua o MAC):
```
ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="00:1a:2b:3c:4d:5e", RUN+="/usr/sbin/ethtool -s %k wol d"
```

Aplicar:
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Alternativa: inserir no `interfaces` (precisa ajustar o nome da interface se renomear depois):
```ini
pre-up /usr/sbin/ethtool -s enp2s0 wol d
```

---

<a id="8"></a>
## 6. 🏷️ (Opcional) Renomear a interface para `eth0`

O Debian 12 usa nomes previsíveis (ex.: `enp2s0`). Se preferir `eth0`, use um arquivo `.link` do systemd (método suportado e atual):

1) Anote o MAC da interface (ex.: `00:1a:2b:3c:4d:5e`).

2) Crie o link file:
```bash
sudo nano /etc/systemd/network/10-eth0.link
```
Conteúdo:
```
[Match]
MACAddress=00:1a:2b:3c:4d:5e

[Link]
Name=eth0
```

 

3) Reboot para aplicar a renomeação:
```bash
sudo reboot
```

4) Após o reboot, ajuste `/etc/network/interfaces` para usar `eth0` no lugar de `enp2s0` e reative a interface, se necessário:
```bash
sudo ifdown eth0 || true
sudo ifup eth0
```

⚠️ Se você usa NetworkManager, a renomeação pode exigir que a conexão do NM seja recriada ou renomeada.

---

<a id="9"></a>
## 7. ✅ Checklist pós-reboot

- `ip -br a` → deve listar `eth0` (se renomeado) ou `enp2s0`.  
- Interface em estado **UP** com o IP esperado.  
- Teste de conectividade:
  ```bash
  ping -c 4 8.8.8.8
  ping -c 4 google.com
  ```
- Verifique WOL:
  ```bash
  sudo ethtool eth0 | grep Wake-on   # ou enp2s0
  ```

---

<a id="10"></a>
## 8. 🚀 Resumo

- Interface configurada para subir no boot via `ifupdown`.  
- Ferramentas instaladas (`ifupdown`, `net-tools`, `ethtool`).  
- WOL desativado de forma persistente.  
- (Opcional) Nome padronizado `eth0` via `.link`.  
- IP fixo quando necessário.

---

<a id="11"></a>
## 9. 🧠 Avançado (opcional)

As seções abaixo cobrem cenários mais avançados. Use apenas se fizer sentido no seu ambiente.

### DNS e `/etc/resolv.conf`

- A diretiva `dns-nameservers` no `/etc/network/interfaces` funciona quando o pacote `resolvconf` está presente.  
- Se não surtir efeito, edite manualmente `/etc/resolv.conf` ou configure o `systemd-resolved`.

Exemplo de `/etc/resolv.conf` simples:
Comando:
```bash
sudo nano /etc/resolv.conf
```
```text
nameserver 1.1.1.1
nameserver 1.0.0.1
```

### Endereço em loopback (anycast/serviços)

Útil para amarrar IPs de serviços à interface `lo`.
```ini
auto lo
iface lo inet loopback

iface lo inet static
    address 200.200.200.200/32
```

### Ponto a ponto (pointopoint)

Quando quer gateway sem usar /30 público no roteador.
```ini
allow-hotplug enp0s3
iface enp0s3 inet static
    address 200.200.200.0
    pointopoint 10.50.50.1
    netmask 255.255.255.255
    gateway 10.50.50.1
```

### IP privado com origem pública (rota com `src`)

Padronize tráfego saindo por IP público mesmo com vizinhança privada.
```ini
allow-hotplug enp0s3
iface enp0s3 inet static
    address 200.200.200.200/32

iface enp0s3 inet static
    address 10.33.33.2/30
    post-up /usr/sbin/ip route add default via 10.33.33.1 src 200.200.200.200
```

IPv6 equivalente:
```ini
# IPv6 Público
iface enp0s3 inet6 static
    pre-up modprobe ipv6
    address 2804:bebe:cafe::cafe
    netmask 128

# IPv6 Privado
iface enp0s3 inet6 static
    pre-up modprobe ipv6
    address fd00:a::2
    netmask 64
    post-up /usr/sbin/ip -6 route add default via fd00:a::1 src 2804:bebe:cafe::cafe
```

### VLAN em `ifupdown`

Carregue o módulo e ative no boot:
```bash
sudo modprobe 8021q
echo 8021q | sudo tee -a /etc/modules
```

Defina a subinterface VLAN (ex.: 171):
```ini
allow-hotplug enp0s3.171
iface enp0s3.171 inet static
    address 10.88.88.2/24
```

### PBR – Roteamento baseado em políticas

Crie uma tabela dedicada e regras de roteamento:
```bash
echo "100 cgnat" | sudo tee -a /etc/iproute2/rt_tables
```

Exemplo enviando 100.64.0.0/10 por outro next-hop:
```ini
allow-hotplug enp0s3
iface enp0s3 inet static
    address 10.200.200.2/30
    post-up ip route add default via 10.200.200.1 dev enp0s3 table cgnat
    post-up ip rule add from 100.64.0.0/10 lookup cgnat
```

### Rotas blackhole

Para descartar tráfego de um prefixo específico:
```ini
allow-hotplug enp0s3
iface enp0s3 inet static
    address 10.33.33.2/30
    post-up /usr/sbin/ip route add blackhole 200.200.200.128/26 metric 250 || true
```

### Comandos úteis (sem reiniciar serviço)

Adicionar/visualizar/remover IPs de forma volátil:
```bash
ip addr add 192.168.7.7/32 dev enp0s3
ip -6 addr add 2001:db8:1::1/128 dev enp0s3
ip address
ip addr del 192.168.7.7/32 dev enp0s3
ip -6 addr del 2001:db8:1::1/128 dev enp0s3
```

---

### Múltiplos IPs na mesma interface

O `ifupdown` não suporta múltiplos `address` no mesmo bloco. Use comandos `up`/`down`:
```ini
iface enp2s0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    # IPs adicionais
    up ip addr add 192.168.1.101/24 dev enp2s0
    down ip addr del 192.168.1.101/24 dev enp2s0 || true
    up ip addr add 192.168.1.102/24 dev enp2s0
    down ip addr del 192.168.1.102/24 dev enp2s0 || true
```

### Rotas com métrica (multi-homing)

Defina a prioridade das rotas padrão:
```ini
post-up ip route replace default via 192.168.1.1 dev enp2s0 metric 100
post-up ip route add default via 10.0.0.1 dev enp3s0 metric 200
```

### MTU, offload, velocidade/duplex e EEE

Verifique capacidades:
```bash
sudo ethtool enp2s0
sudo ethtool -k enp2s0   # offloads
sudo ethtool --show-eee enp2s0
```

Ajustes comuns (sessão atual):
```bash
# desativar EEE (pode corrigir instabilidade com alguns switches)
sudo ethtool --set-eee enp2s0 eee off

# forçar 1000/Full (cautela: deve combinar com o switch)
sudo ethtool -s enp2s0 autoneg off speed 1000 duplex full

# offloads (apenas para troubleshooting)
sudo ethtool -K enp2s0 gro off gso off tso off lro off rx off tx off
```

### Persistir ajustes de `ethtool` com systemd

Crie um serviço para aplicar no boot (ajuste interface e comandos conforme seu caso):
```bash
sudo nano /etc/systemd/system/ethtool-enp2s0.service
```
Conteúdo:
```
[Unit]
Description=Apply ethtool settings for enp2s0
Wants=network-pre.target
Before=network-pre.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -s enp2s0 wol d
ExecStart=/usr/sbin/ethtool --set-eee enp2s0 eee off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Ative:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ethtool-enp2s0.service
```

### Bridge (br0) para virtualização/containers

Instale utilitários e crie a bridge:
```bash
sudo apt install -y bridge-utils
```
Exemplo com IP na bridge e a porta física como membro:
```ini
auto br0
iface br0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    bridge_ports enp2s0
    bridge_stp off
    bridge_fd 0
```

### Bonding (agregação) — 802.3ad ou active-backup

Instale e carregue o módulo:
```bash
sudo apt install -y ifenslave
echo bonding | sudo tee -a /etc/modules
```

Exemplo LACP (requer switch configurado para 802.3ad):
```ini
auto bond0
iface bond0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    slaves enp2s0 enp3s0
    bond-mode 802.3ad
    bond-miimon 100
    bond-lacp-rate 1
    bond-xmit-hash-policy layer3+4
```

Exemplo active-backup (sem necessidade de LACP):
```ini
auto bond0
iface bond0 inet dhcp
    slaves enp2s0 enp3s0
    bond-mode active-backup
    bond-miimon 100
```

### Alternativa: `systemd-networkd`
Para servidores modernos, `systemd-networkd` é uma alternativa poderosa ao `ifupdown`. A configuração é feita em arquivos `.network` no diretório `/etc/systemd/network/`.

Arquivo: `/etc/systemd/network/10-wired.network`
Comando:
```bash
sudo nano /etc/systemd/network/10-wired.network
```
Conteúdo (estático IPv4 + SLAAC IPv6):
```
[Match]
Name=enp2s0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=1.1.1.1
IPv6AcceptRA=yes
```
Ative o serviço e desative o que não usa:
```bash
sudo systemctl disable --now networking NetworkManager
sudo systemctl enable --now systemd-networkd systemd-resolved
```
Sincronize o `resolv.conf`:
```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

---

<a id="12"></a>
## 10. 🧰 Solução de Problemas

- Link sobe/desce (flapping): verifique `dmesg -T | grep -i link`, negocie velocidade/duplex com `ethtool`, desative EEE.
- Sem IP no DHCP: `journalctl -b -u networking | tail -n 100`, `sudo dhclient -v enp2s0` e analise respostas.
- DNS não resolve: confira `/etc/resolv.conf`, `systemd-resolved` (`resolvectl status`).
- Gateway inalcançável: `ip route`, `ping -c 2 gateway`, verifique VLAN/MTU.
- Sem conectividade externa: teste `ping 8.8.8.8`; se ok e nomes falham, é DNS. Use `traceroute`/`mtr` para rotas.
- Inspecione tráfego: `sudo tcpdump -i enp2s0 arp or icmp`.
 

---

## Referências (fontes para consulta)

### Debian Wiki

- NetworkConfiguration: https://wiki.debian.org/NetworkConfiguration

### Manpages (Debian)

- `interfaces(5)` (ifupdown): https://manpages.debian.org/bookworm/ifupdown/interfaces.5.en.html
- `nmcli(1)` (NetworkManager): https://manpages.debian.org/bookworm/network-manager/nmcli.1.en.html
- `ip(8)` (iproute2): https://manpages.debian.org/bookworm/iproute2/ip.8.en.html
- `systemd-networkd.service(8)`: https://manpages.debian.org/bookworm/systemd/systemd-networkd.service.8.en.html
- `resolvectl(1)` (systemd-resolved): https://manpages.debian.org/bookworm/systemd/resolvectl.1.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
