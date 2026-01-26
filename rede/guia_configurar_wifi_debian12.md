# Configuração de Wi‑Fi – Debian 12

*Criado em 29 de setembro de 2025 e atualizado em 08 de dezembro de 2025*

Neste guia prático, o Wi‑Fi é configurado no **Debian 12 (Bookworm)** cobrindo dois caminhos comuns: servidores (`ifupdown` + `wpa_supplicant`) e desktops (`NetworkManager` via `nmcli`). Há exemplos para redes comuns, redes ocultas e WPA2‑Enterprise, além de dicas para resolver problemas.

Veja também: [Configuração de Rede – Debian 12 (Ethernet)](../rede/guia_configurar_rede_debian12.md).

---

## Índice rápido
1. [Começo Rápido (nmcli para Desktop)](#1)
2. [Pré‑requisitos e Identificação](#2)
3. [Método 1: `ifupdown` + `wpa_supplicant` (Servidores)](#3)
4. [Método 2: `NetworkManager` (nmcli) — Detalhado](#4)
5. [Configurações Avançadas (WPA3, Renomear)](#5)
6. [Checklist e Solução de Problemas](#6)

---

<a id="1"></a>
## 1. Começo Rápido (nmcli para Desktop)
Se você está em um ambiente com desktop e só quer se conectar rapidamente, o `nmcli` costuma ser a opção mais prática.

1.  **Liste as redes disponíveis:**
    ```bash
    nmcli dev wifi list
    ```
2.  **Identifique sua interface:**
    ```bash
    iw dev
    # Anote o nome da sua interface, por exemplo: wlp2s0
    ```
3.  **Conecte-se (substitua `wlp2s0` pelo nome correto):**
    ```bash
    nmcli dev wifi connect "NomeDaSuaRede" password "SuaSenha" ifname wlp2s0
    ```
4.  **Verifique a conexão:**
    ```bash
    ip -br a
    ping -c 3 google.com
    ```
Se a conexão funcionar, prossiga para o guia completo abaixo (mais detalhes e alternativas).

---

<a id="2"></a>
## 2. Pré‑requisitos e Identificação
- Acesso `sudo` e internet (para instalar pacotes).
- Escolha um gerenciador: `ifupdown` (servidor) ou `NetworkManager` (desktop).

**Instale as ferramentas:**
```bash
sudo apt update
sudo apt install -y wpasupplicant iw rfkill
```

**Identifique sua interface e verifique o rádio:**
```bash
iw dev                # Lista interfaces Wi-Fi (anote o nome, ex: wlp2s0)
rfkill list           # Verifica se há bloqueio de rádio
sudo rfkill unblock wifi  # Desbloqueia se necessário
```
> **Importante:** Todos os exemplos neste guia usarão `wlp2s0`. Lembre-se de substituir por o nome da sua interface onde for necessário.

---

<a id="3"></a>
## 3. Método 1: `ifupdown` + `wpa_supplicant` (Servidores)

Este método é ideal para servidores, pois depende de arquivos de configuração estáticos.

### 3.1 Gerar o arquivo de configuração do `wpa_supplicant`
> Lembre-se de substituir `wlp2s0` pelo nome da sua interface.
```bash
# Gere a configuração com sua rede e senha
wpa_passphrase "NomeDaSuaRede" "SuaSenha" | sudo tee /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```
Agora, adicione um cabeçalho importante no arquivo recém-criado:
```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```
Insira no **topo** do arquivo:
```ini
ctrl_interface=DIR=/run/wpa_supplicant GROUP=netdev
update_config=1
country=BR

# Abaixo deste cabeçalho ficará o bloco network={...} gerado anteriormente
```

### 3.2 Configurar a interface de rede
Edite o arquivo `/etc/network/interfaces`.
```bash
sudo nano /etc/network/interfaces
```
Adicione o bloco para sua interface, **substituindo `wlp2s0` se necessário**.

**Opção A: DHCP (IP automático)**
```ini
auto wlp2s0
iface wlp2s0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```

**Opção B: IP Fixo**
```ini
auto wlp2s0
iface wlp2s0 inet static
    address 192.168.1.50
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
    wpa-conf /etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf
```

### 3.3 Aplicar a configuração
```bash
# Substitua wlp2s0 se necessário
sudo ifdown wlp2s0 || true
sudo ifup wlp2s0
```

---

<a id="4"></a>
## 4. Método 2: `NetworkManager` (nmcli) — Detalhado

Perfeito para desktops e notebooks. **Lembre-se de substituir `wlp2s0` pelo nome da sua interface.**

### 4.1 Conectar (DHCP)
```bash
# Listar redes
nmcli dev wifi list

# Conectar
nmcli dev wifi connect "NomeDaSuaRede" password "SuaSenha" ifname wlp2s0
```

### 4.2 Conectar com IP Fixo
```bash
# Substitua wlp2s0 e os dados de rede
nmcli con add type wifi ifname wlp2s0 ssid "NomeDaSuaRede" con-name "WiFi-Fixo" \
  wifi-sec.key-mgmt wpa-psk wifi-sec.psk "SuaSenha" \
  ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,1.1.1.1" ipv4.method manual

# Ativar a conexão
nmcli con up "WiFi-Fixo"
```

### 4.3 Rede Oculta (Hidden SSID)
```bash
# Substitua wlp2s0 e os dados de rede
nmcli con add type wifi ifname wlp2s0 ssid "MinhaRedeOculta" con-name "WiFi-Oculto" \
  wifi.hidden yes wifi-sec.key-mgmt wpa-psk wifi-sec.psk "SuaSenha"

nmcli con up "WiFi-Oculto"
```

---

<a id="5"></a>
## 5. Configurações Avançadas

### WPA3-Personal (SAE)
Se sua rede e placa suportam WPA3, no arquivo `/etc/wpa_supplicant/wpa_supplicant-wlp2s0.conf`, adicione:
```ini
network={
    ssid="MinhaRedeWPA3"
    key_mgmt=SAE
    psk="minha_senha"
}
```

### Renomear a interface para `wlan0`
1.  **Descubra o MAC Address:** `ip link show wlp2s0`
2.  **Crie o arquivo de link:**
    ```bash
    sudo nano /etc/systemd/network/10-wlan0.link
    ```
3.  **Adicione o conteúdo (substitua o MAC):**
    ```ini
    [Match]
    MACAddress=aa:bb:cc:dd:ee:ff

    [Link]
    Name=wlan0
    ```
4.  Reinicie o computador (`sudo reboot`) e ajuste os arquivos de configuração para usar `wlan0`.

---

<a id="6"></a>
## 6. Checklist e Solução de Problemas

- **Checklist:**
  - `iw dev wlp2s0 link` → Deve mostrar "Connected".
  - `ip -br a` → Deve mostrar um IP na interface.
  - `ping -c 3 8.8.8.8` → Deve funcionar.

- **Solução de Problemas:**
  - **Rádio bloqueado?** → Use `rfkill list` e `sudo rfkill unblock wifi`.
  - **Firmware faltando?** → `dmesg -T | grep -i firmware`. Instale o pacote `firmware-iwlwifi` (para Intel) ou `firmware-realtek` (procure pelo seu modelo) do repositório `non-free-firmware`.
  - **Logs do wpa_supplicant:** `journalctl -u wpa_supplicant -n 50 --no-pager`.
  - **País regulatório:** `sudo iw reg get` e ajuste `country=BR` no seu `.conf`.
  - **Powersave causando instabilidade?** → `sudo iw dev wlp2s0 set power_save off`.

---

## Referências (fontes para consulta)

### Debian Wiki

- WiFi: https://wiki.debian.org/WiFi

### wpa_supplicant / NetworkManager

- wpa_supplicant (oficial): https://w1.fi/wpa_supplicant/
- `wpa_supplicant.conf(5)` (manpage Debian): https://manpages.debian.org/bookworm/wpasupplicant/wpa_supplicant.conf.5.en.html
- `nmcli(1)` (NetworkManager): https://manpages.debian.org/bookworm/network-manager/nmcli.1.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
