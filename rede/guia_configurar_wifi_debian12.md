# Guia de Produção: Configuração de Wi-Fi no Debian 12

*Criado em: 29 de setembro de 2025*  
*Última atualização em: 18 de março de 2026*

Montei este guia para deixar registrado um caminho previsível para configurar Wi-Fi no **Debian 12 (Bookworm)**, cobrindo dois cenários comuns: **servidor** com `ifupdown` + `wpa_supplicant` e **desktop/notebook** com `NetworkManager` via `nmcli`.

O foco aqui é colocar a interface no ar com segurança e sem conflito entre gerenciadores de rede. O guia não cobre modo AP/hotspot nem automações mais exóticas.

Veja também: [guia de configuração de rede no Debian 12/13 (Ethernet)](./guia_configurar_rede_debian12_13.md)

---

## Índice rápido
1. [Começo rápido com `nmcli`](#1)
2. [Pré-requisitos e identificação](#2)
3. [Método 1: `ifupdown` + `wpa_supplicant`](#3)
4. [Método 2: `NetworkManager` com `nmcli`](#4)
5. [Casos específicos e ajustes avançados](#5)
6. [Checklist e solução de problemas](#6)

---

<a id="1"></a>
## 1. Começo rápido com `nmcli`

Se a máquina já usa desktop e o `NetworkManager`, esse costuma ser o caminho mais direto.

1. Ligue o rádio Wi-Fi, se necessário:

```bash
nmcli radio wifi on
```

2. Liste as interfaces e conexões existentes:

```bash
nmcli device status
nmcli con show
```

3. Liste as redes disponíveis:

```bash
nmcli dev wifi list
```

4. Conecte na rede:

```bash
nmcli dev wifi connect "NomeDaRede" password "SuaSenha" ifname wlp2s0
```

5. Valide:

```bash
ip -br a
ping -c 3 1.1.1.1
```

Se funcionar, você já pode parar aqui. O restante do guia aprofunda os dois caminhos.

---

<a id="2"></a>
## 2. Pré-requisitos e identificação

Escolha primeiro quem vai gerenciar a interface:

- `ifupdown` + `wpa_supplicant`: faz mais sentido em servidor ou ambiente enxuto
- `NetworkManager`: faz mais sentido em desktop e notebook

Instale as ferramentas base:

```bash
sudo apt update
sudo apt install -y wpasupplicant iw rfkill
```

Se você for seguir pelo `NetworkManager` e o `nmcli` não existir na máquina:

```bash
sudo apt install -y network-manager
```

Identifique a interface e veja se o rádio está bloqueado:

```bash
iw dev
rfkill list
sudo rfkill unblock wifi
```

Nos exemplos abaixo, vou usar `wlp2s0`. Troque pelo nome real da sua interface quando necessário.

---

<a id="3"></a>
## 3. Método 1: `ifupdown` + `wpa_supplicant`

Esse caminho faz mais sentido em servidor, VM com placa Wi-Fi passada direto ou ambiente onde você quer tudo em arquivo e sem `NetworkManager`.

### 3.1 Gerar o arquivo do `wpa_supplicant`

Crie o arquivo da interface com `wpa_passphrase`:

```bash
wpa_passphrase "NomeDaRede" "SuaSenha" | sudo tee /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
sudo chmod 600 /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```

Edite o arquivo:

```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```

Deixe o topo assim:

```ini
ctrl_interface=DIR=/run/wpa_supplicant GROUP=netdev
update_config=1
country=BR
```

Logo abaixo deve ficar o bloco `network={...}` gerado pelo `wpa_passphrase`.

Se quiser reduzir exposição da senha em claro, remova a linha comentada `#psk="SuaSenha"` que o `wpa_passphrase` costuma adicionar.

### 3.2 Configurar a interface no `ifupdown`

Se for mexer em `/etc/network/interfaces`, faça backup antes:

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```

Edite o arquivo:

```bash
sudo nano /etc/network/interfaces
```

Exemplo com DHCP:

```ini
auto wlp2s0
iface wlp2s0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```

Exemplo com IP fixo:

```ini
auto wlp2s0
iface wlp2s0 inet static
    address 192.168.1.50
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
    wpa-conf /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```

Observação importante sobre DNS:

- em `ifupdown`, `dns-nameservers` só tem efeito quando existe algo aplicando isso ao `resolv.conf`, como `openresolv`
- se o host usa outro gerenciador para DNS, valide primeiro quem controla `/etc/resolv.conf`

### 3.3 Aplicar a configuração

Se a interface ainda não estava ativa:

```bash
sudo ifup wlp2s0
```

Se você está reaplicando a configuração:

```bash
sudo ifdown wlp2s0
sudo ifup wlp2s0
```

Se isso for feito por SSH, tenha console ou algum plano de volta antes de derrubar a interface.

---

<a id="4"></a>
## 4. Método 2: `NetworkManager` com `nmcli`

Esse é o caminho mais confortável em desktop e notebook. O importante aqui é não misturar `NetworkManager` com `ifupdown` na mesma interface.

Antes de criar conexão nova, veja o que já existe:

```bash
nmcli device status
nmcli con show
```

### 4.1 Conexão comum com DHCP

```bash
nmcli dev wifi list
nmcli dev wifi connect "NomeDaRede" password "SuaSenha" ifname wlp2s0
```

### 4.2 Conexão com IP fixo

```bash
nmcli con add type wifi ifname wlp2s0 ssid "NomeDaRede" con-name "wifi-fixo" \
  wifi-sec.key-mgmt wpa-psk wifi-sec.psk "SuaSenha" \
  ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 1.1.1.1" ipv4.method manual ipv6.method auto
nmcli con up "wifi-fixo"
```

### 4.3 Rede oculta

```bash
nmcli con add type wifi ifname wlp2s0 ssid "MinhaRedeOculta" con-name "wifi-oculto" \
  wifi.hidden yes wifi-sec.key-mgmt wpa-psk wifi-sec.psk "SuaSenha"
nmcli con up "wifi-oculto"
```

### 4.4 Se a interface ainda estiver em `/etc/network/interfaces`

Se a interface estiver declarada em `/etc/network/interfaces`, o `NetworkManager` normalmente não vai assumir o controle dela.

Edite o arquivo:

```bash
sudo nano /etc/network/interfaces
```

Comente ou remova o bloco da interface que você quer passar para o `NetworkManager`.

Depois recarregue o serviço:

```bash
sudo systemctl restart NetworkManager
```

---

<a id="5"></a>
## 5. Casos específicos e ajustes avançados

### WPA3-Personal (SAE)

Se a rede e a placa suportarem WPA3, no arquivo do `wpa_supplicant` o bloco pode ficar assim:

```ini
network={
    ssid="MinhaRedeWPA3"
    key_mgmt=SAE
    psk="minha_senha"
}
```

### Renomear a interface para `wlan0`

Isso só faz sentido se você realmente quiser abandonar o nome previsível atual.

1. Descubra o MAC da interface:

```bash
ip link show wlp2s0
```

2. Crie o arquivo `/etc/systemd/network/10-wlan0.link`:

```bash
sudo nano /etc/systemd/network/10-wlan0.link
```

3. Cole o conteúdo abaixo e troque o MAC:

```ini
[Match]
MACAddress=aa:bb:cc:dd:ee:ff

[Link]
Name=wlan0
```

4. Reinicie a máquina:

```bash
sudo reboot
```

5. Depois ajuste os arquivos de rede para usar `wlan0`.

---

<a id="6"></a>
## 6. Checklist e solução de problemas

Checklist rápido:

- `iw dev wlp2s0 link` deve mostrar que a interface está associada
- `ip -br a` deve mostrar IP na interface
- `ping -c 3 1.1.1.1` deve responder
- `ping -c 3 debian.org` deve responder se DNS estiver correto

Comandos úteis de diagnóstico:

```bash
rfkill list
iw dev
iw dev wlp2s0 link
ip -br a
```

Problemas comuns:

- rádio bloqueado: `sudo rfkill unblock wifi`
- firmware faltando: `dmesg -T | grep -i firmware`
- logs do `wpa_supplicant`: `journalctl -u wpa_supplicant -n 50 --no-pager`
- logs do `NetworkManager`: `journalctl -u NetworkManager -n 50 --no-pager`
- país regulatório: confirme `country=BR` no arquivo do `wpa_supplicant`
- economia de energia causando instabilidade: `sudo iw dev wlp2s0 set power_save off`

Se a placa for Intel ou Realtek e o firmware estiver faltando, instale o pacote correto do repositório `non-free-firmware`.

---

## Referências (fontes para consulta)

### Debian

- Debian Wiki — WiFi: https://wiki.debian.org/WiFi
- Debian Wiki — NetworkManager: https://wiki.debian.org/NetworkManager

### Manpages

- `wpa_supplicant(8)`: https://manpages.debian.org/bookworm/wpasupplicant/wpa_supplicant.8.en.html
- `wpa_supplicant.conf(5)`: https://manpages.debian.org/bookworm/wpasupplicant/wpa_supplicant.conf.5.en.html
- `nmcli(1)`: https://manpages.debian.org/bookworm/network-manager/nmcli.1.en.html
- `resolvconf.conf(5)`: https://manpages.debian.org/bookworm/openresolv/resolvconf.conf.5.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
