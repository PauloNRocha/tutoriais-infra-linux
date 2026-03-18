# Guia de Produção: Configuração de Rede no Debian 12/13 (Ethernet)

*Criado em: 29 de setembro de 2025*  
*Última atualização em: 17 de março de 2026*

Montei este guia para deixar registrado um jeito estável e previsível de configurar rede cabeada no **Debian 12/13**. O foco principal aqui é `ifupdown`, mas o guia também traz alternativa com `NetworkManager` e alguns cenários mais avançados para quando o básico já estiver funcionando.

Observação prática:

- em teste rápido em uma VM Debian 13 mínima, `ifupdown` continuou funcionando como esperado;
- já o `NetworkManager` precisou ser instalado manualmente;
- e a interface só passou para o `NM` quando deixou de ser declarada em `/etc/network/interfaces`.

Veja também: [guia de configuração de Wi-Fi no Debian 12](./guia_configurar_wifi_debian12.md)

---

## Índice rápido
1. [Pré-requisitos e Escopo](#1)
2. [Começo Rápido (DHCP)](#2)
3. [Instalar ferramentas necessárias](#3)
4. [Identificar a interface, IP e MAC](#4)
5. [Configurar Ethernet para subir no boot (ifupdown)](#5)
6. [NetworkManager (nmcli) — alternativa ao ifupdown](#6)
7. [Desativar Wake-on-LAN (WOL)](#7)
8. [(Opcional) Renomear a interface para `eth0`](#8)
9. [Checklist pós-reboot](#9)
10. [Avançado (opcional)](#10)
11. [Solução de Problemas](#11)

---

<a id="1"></a>
## 1. Pré-requisitos e escopo

- Acesso com privilégios de administrador (`sudo`).  
- Ambiente alvo: servidores/notebooks com interface Ethernet.  
- Escolha um gerenciador de rede por host (não misture na mesma interface):
  - `ifupdown` (padrão deste guia) → simples, estável, arquivo `/etc/network/interfaces`.
  - `NetworkManager` (NM) → preferível em desktops; há seção alternativa com `nmcli`.
  - `systemd-networkd` → opção moderna para servidores; seção opcional ao final.

Na prática:

- para servidor, eu começaria com `ifupdown` ou `systemd-networkd`;
- para desktop, faz mais sentido usar `NetworkManager`.

---

<a id="2"></a>
## 2. Começo rápido (DHCP com `ifupdown`)

Para quem precisa apenas conectar a rede com fio via DHCP rapidamente.

1.  **Identifique a interface:**
    ```bash
    ip -br a
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



---

<a id="3"></a>
## 3. Instalar ferramentas necessárias

```bash
sudo apt update
sudo apt install -y ifupdown net-tools ethtool
```

- `ifupdown` → gerencia interfaces via `/etc/network/interfaces`.  
- `net-tools` → utilitários clássicos (`ifconfig`, `netstat`); hoje é opcional e fica mais por compatibilidade com hábito antigo.
- `ethtool` → opções da placa (WOL, velocidade/duplex).

---

<a id="4"></a>
## 4. Identificar a interface, IP e MAC

```bash
ip -br a
```

Exemplo de saída:

```
enp2s0           UP     192.168.1.20/24
```

Esse comando é o melhor para bater o olho e descobrir:

- nome da interface;
- estado (`UP`/`DOWN`);
- IPs atribuídos.

Para ver o MAC de forma resumida:

```bash
ip -br link
```

Exemplo:

```
enp2s0           UP             00:1a:2b:3c:4d:5e
```

Interface: `enp2s0`  
MAC: `00:1a:2b:3c:4d:5e`

Se quiser ver os detalhes completos de uma interface específica:
```bash
ip link show enp2s0
```

Se quiser ver a listagem completa de todas as interfaces do host:

```bash
ip a
```

---

<a id="5"></a>
## 5. Configurar Ethernet para subir no boot (ifupdown)

Aqui a ideia é deixar essa interface sob controle do `ifupdown`, não do `NetworkManager`.

1) Faça backup do arquivo:
```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```

2) Edite o arquivo:

```bash
sudo nano /etc/network/interfaces
```

3) Escolha UMA das opções abaixo (ajuste `enp2s0` e o IP conforme necessário):

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

Observação importante sobre DNS:

- em `ifupdown`, a diretiva `dns-nameservers` só tem efeito quando existe algo no sistema aplicando isso ao `resolv.conf`, como `resolvconf`;
- em host que usa `systemd-resolved` ou `NetworkManager`, o comportamento do DNS pode ser outro;
- se o DNS não subir como esperado, valide primeiro quem controla `/etc/resolv.conf` antes de insistir em editar o arquivo manualmente.

4) Aplicar sem reboot:
```bash
sudo ifdown enp2s0
sudo ifup enp2s0
```

Se estiver remoto por SSH, essa etapa merece cautela. Se possível, tenha console, snapshot ou outra forma de voltar atrás antes de derrubar a interface.

Se você usa Desktop com `NetworkManager`, faça uma escolha:

- ou deixa essa interface fora do `NM`;
- ou configura tudo via `nmcli`.

Misturar `NetworkManager` e `ifupdown` na mesma interface costuma virar conflito.

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
    pre-down ip route del 10.10.0.0/16 via 192.168.1.254 dev enp2s0

iface enp2s0 inet6 static
    address 2001:db8:1234::10
    netmask 64
    gateway 2001:db8:1234::1
    # rota IPv6 extra
    post-up ip -6 route add 2001:db8:abcd::/48 via 2001:db8:1234::fe dev enp2s0
    pre-down ip -6 route del 2001:db8:abcd::/48 via 2001:db8:1234::fe dev enp2s0
```

### Reiniciar serviço (cautela)

```bash
sudo systemctl restart networking
```
Preferir `ifdown/ifup` da interface específica quando possível, para evitar queda geral de rede.

---

<a id="6"></a>
## 6. NetworkManager (nmcli) — alternativa ao ifupdown

Use esta opção em ambientes Desktop ou quando já utiliza NM. Não misture NM e ifupdown na mesma interface.

### Instalar o NetworkManager, se necessário

Em instalação mínima do Debian, o `nmcli` pode não existir ainda.

```bash
sudo apt install network-manager -y
```

Antes de criar conexões novas, veja o que já existe:

```bash
nmcli device status
nmcli con show
```

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

### Se a interface ainda estiver no `/etc/network/interfaces`

Se a interface estiver declarada em `/etc/network/interfaces`, o `NetworkManager` normalmente não vai assumir o controle dela.

Edite o arquivo:
```bash
sudo nano /etc/network/interfaces
```

Remova ou comente o bloco da interface que você quer passar para o `NM`.

Depois reinicie o `NetworkManager`:

```bash
sudo systemctl restart NetworkManager
```

---

<a id="7"></a>
## 7. Desativar Wake-on-LAN (WOL)

Verificar WOL atual:
```bash
sudo ethtool enp2s0 | grep Wake-on
```

Desativar (sessão atual):
```bash
sudo ethtool -s enp2s0 wol d
```

Crie o arquivo `/etc/udev/rules.d/70-disable-wol.rules` e ajuste o MAC:
```bash
sudo nano /etc/udev/rules.d/70-disable-wol.rules
```
Cole o conteúdo abaixo:
```text
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
## 8. (Opcional) Renomear a interface para `eth0`

O Debian 12 usa nomes previsíveis (ex.: `enp2s0`). Se preferir `eth0`, use um arquivo `.link` do systemd (método suportado e atual):

1) Anote o MAC da interface (ex.: `00:1a:2b:3c:4d:5e`).

2) Crie o arquivo `/etc/systemd/network/10-eth0.link`:
```bash
sudo nano /etc/systemd/network/10-eth0.link
```
Cole o conteúdo abaixo:
```ini
[Match]
MACAddress=00:1a:2b:3c:4d:5e

[Link]
Name=eth0
```

3) Se você usa `ifupdown`, edite também o `/etc/network/interfaces` antes do reboot e troque `enp2s0` por `eth0`:

```bash
sudo nano /etc/network/interfaces
```

4) Reinicie para aplicar a renomeação:
```bash
sudo reboot
```

5) Depois do reboot, confirme se a interface apareceu com o nome novo:
```bash
ip -br a
```

6) Se a interface não subir sozinha e você já tiver ajustado o `/etc/network/interfaces`, use:

```bash
sudo ifup eth0
```

Se aparecer `ifup: unknown interface eth0`, o `ifupdown` ainda não conhece esse nome novo no `/etc/network/interfaces`.

Se você usa `NetworkManager`, a renomeação pode exigir recriar ou renomear a conexão no `NM`.

---

<a id="9"></a>
## 9. Checklist pós-reboot

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
## 10. Avançado (opcional)

As seções abaixo cobrem cenários mais avançados. Trate esta parte como **referência técnica**, não como sequência cega para copiar e colar em produção.

Qualquer mudança envolvendo:

- roteamento;
- renomeação de interface;
- troca de gerenciador de rede;

precisa ser testada antes em VM, snapshot ou console.

### DNS e `/etc/resolv.conf`

No Debian, o comportamento do DNS depende de quem está gerenciando o `/etc/resolv.conf`.

Antes de editar, verifique:

```bash
ls -l /etc/resolv.conf
```

Possíveis cenários:

- arquivo estático: pode editar manualmente;
- symlink para `systemd-resolved`: gerenciado automaticamente;
- sobrescrito por DHCP ou `NetworkManager`: alteração manual pode ser perdida.

Boa prática:

- definir DNS no gerenciador de rede em uso;
- evitar editar `/etc/resolv.conf` manualmente em produção sem saber quem controla o arquivo.

Exemplo, somente se ele for estático:

```bash
sudo nano /etc/resolv.conf
```

```text
nameserver 1.1.1.1
nameserver 1.0.0.1
```

### Endereço em loopback (anycast / IP de serviço)

Útil para bind de serviço, VIP ou cenários simples de anycast local.

```ini
auto lo
iface lo inet loopback
    up ip addr add 198.51.100.200/32 dev lo
    down ip addr del 198.51.100.200/32 dev lo
```

### Ponto a ponto (`pointopoint`)

Usado em cenários específicos onde o gateway não está no mesmo prefixo.

```ini
auto enp0s3
iface enp0s3 inet static
    address 198.51.100.10
    pointopoint 10.50.50.1
    netmask 255.255.255.255
    gateway 10.50.50.1
```

Uso avançado: valide em VM ou console antes de aplicar remotamente.

### IP privado com origem pública (rota com `src`)

Permite saída com IP público mesmo usando uma vizinhança privada no enlace.

```ini
auto enp0s3
iface enp0s3 inet static
    address 10.33.33.2/30
    up ip addr add 198.51.100.10/32 dev enp0s3
    post-up ip route replace default via 10.33.33.1 src 198.51.100.10
    pre-down ip addr del 198.51.100.10/32 dev enp0s3
```

### IPv6 equivalente

```ini
auto enp0s3
iface enp0s3 inet6 static
    address fd00:a::2/64
    up ip -6 addr add 2001:db8:100::10/128 dev enp0s3
    post-up ip -6 route replace default via fd00:a::1 src 2001:db8:100::10
    pre-down ip -6 addr del 2001:db8:100::10/128 dev enp0s3
```

Não é necessário usar `modprobe ipv6` em Debian 12/13 atual.

### VLAN em `ifupdown`

Na maioria dos casos, basta instalar o pacote `vlan` e usar o nome da subinterface.

```bash
sudo apt install -y vlan
```

```ini
auto enp0s3.256
iface enp0s3.256 inet static
    address 10.88.88.2/24
```

O módulo `8021q` costuma carregar automaticamente. Só force manualmente se realmente precisar.

### PBR — Roteamento baseado em políticas

No Debian 12/13, o arquivo do pacote costuma existir em `/usr/share/iproute2/rt_tables`, mas `/etc/iproute2/rt_tables` pode não existir por padrão. Para não alterar arquivo do pacote, crie o override em `/etc`.

Motivo:

- `/usr/share/iproute2/rt_tables` pertence ao pacote;
- `/etc/iproute2/rt_tables` é o lugar mais correto para customização local do host;
- assim você evita perder ajuste seu em atualização de pacote.

```bash
sudo install -d -m 755 /etc/iproute2
sudo nano /etc/iproute2/rt_tables
```

Adicione uma única linha:

```text
100 cgnat
```

Exemplo:

```ini
auto enp0s3
iface enp0s3 inet static
    address 10.200.200.2/30
    post-up ip route replace default via 10.200.200.1 dev enp0s3 table cgnat
    post-up ip rule add from 100.64.0.0/10 lookup cgnat
    pre-down ip rule del from 100.64.0.0/10 lookup cgnat
```

### Rotas blackhole

```ini
post-up ip route replace blackhole 198.51.100.128/26 metric 250
```

### Comandos úteis (temporários)

```bash
ip addr add 192.168.7.7/32 dev enp0s3
ip -6 addr add 2001:db8:1::1/128 dev enp0s3
ip address
ip addr del 192.168.7.7/32 dev enp0s3
ip -6 addr del 2001:db8:1::1/128 dev enp0s3
```

### Múltiplos IPs na mesma interface

No `ifupdown`, um caminho comum é usar comandos `up`/`down`:

```ini
iface enp2s0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    up ip addr add 192.168.1.101/24 dev enp2s0
    down ip addr del 192.168.1.101/24 dev enp2s0
```

### Rotas com métrica (multi-homing)

```ini
post-up ip route replace default via 192.168.1.1 dev enp2s0 metric 100
post-up ip route replace default via 10.0.0.1 dev enp3s0 metric 200
```

### MTU, offload, velocidade/duplex e EEE

```bash
sudo ethtool enp2s0
sudo ethtool -k enp2s0
sudo ethtool --show-eee enp2s0
```

Se `--show-eee` retornar `Operation not supported`, o driver ou a NIC não expõe controle de EEE via `ethtool`. Nesse caso, ignore esse ajuste.

Exemplo de ajuste, somente se o hardware suportar:

```bash
sudo ethtool --set-eee enp2s0 eee off
sudo ethtool -s enp2s0 autoneg off speed 1000 duplex full
```

Desativar offloads deve ficar só para troubleshooting.

### Persistir ajustes de `ethtool` com `systemd`

Crie o arquivo `/etc/systemd/system/ethtool-enp2s0.service` e cole o conteúdo abaixo:

```bash
sudo nano /etc/systemd/system/ethtool-enp2s0.service
```

```ini
[Unit]
Description=Apply ethtool settings for enp2s0
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool --set-eee enp2s0 eee off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ethtool-enp2s0.service
```

### Bridge (`br0`)

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

### Bonding

```ini
auto bond0
iface bond0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    bond-slaves enp2s0 enp3s0
    bond-mode 802.3ad
    bond-miimon 100
```

LACP exige configuração compatível no switch.

### Alternativa: `systemd-networkd`

Exemplo básico:

Crie o arquivo `/etc/systemd/network/10-wired.network` e cole o conteúdo abaixo:

```bash
sudo nano /etc/systemd/network/10-wired.network
```

```ini
[Match]
Name=enp2s0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=1.1.1.1
IPv6AcceptRA=yes
```

Ativação:

```bash
sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd
```

Aviso operacional:

- só desative `networking` ou `NetworkManager` depois de validar que o `networkd` realmente assumiu a interface;
- não faça essa troca por SSH sem console, snapshot ou outra forma de recuperação.

<a id="11"></a>
## 11. Solução de Problemas

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

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
