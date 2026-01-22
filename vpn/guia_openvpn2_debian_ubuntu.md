# Guia: OpenVPN **2** no Linux (Debian/Ubuntu)

*Criado em 29 de setembro de 2025 e atualizado em 08 de dezembro de 2025*

> **Objetivo:** reproduzir no Linux a mesma experiência do **OpenVPN GUI do Windows**, usando o **OpenVPN 2 (clássico)**.  
> **Para quem é:** quem possui um arquivo `*.ovpn` fornecido pela empresa/servidor VPN e quer **conectar de forma simples** e confiável.

---

## Índice rápido
1. [Entendendo as versões (OpenVPN 2 vs 3)](#1)
2. [Instalação e Pré‑requisitos](#2)
3. [Método 1: Conexão Rápida via Terminal](#3)
4. [Checklist Rápido de Verificação](#4)
5. [Método 2: Integrar com NetworkManager (Desktop)](#5)
6. [Automação: Iniciar com o Sistema](#6)
7. [Dicas e Soluções (FAQ)](#7)

---

<a id="1"></a>
## 1. Entendendo as versões (OpenVPN 2 vs 3)
- **OpenVPN 2 (`openvpn`)**: A versão clássica, estável e amplamente documentada. O "OpenVPN GUI" do Windows gerencia o OpenVPN 2. **Este guia foca nesta versão.**
- **OpenVPN 3 (`openvpn3`)**: Uma reimplementação mais nova. Funciona bem, mas tem comandos diferentes e menos tutoriais disponíveis.

> **Recomendação:** para uma experiência mais próxima do Windows e compatibilidade máxima, use **OpenVPN 2**.

---

<a id="2"></a>
## 2. Instalação e Pré‑requisitos
- Um arquivo de configuração `cliente.ovpn`.
- Certificados/chaves, se não estiverem embutidos no `.ovpn`.
- Acesso `sudo` no seu sistema.

**Comando de Instalação (Debian/Ubuntu):**
```bash
# Instala o cliente de linha de comando
sudo apt update
sudo apt install -y openvpn

# Opcional, para integrar com o ícone de rede do desktop:
sudo apt install -y network-manager-openvpn network-manager-openvpn-gnome
sudo systemctl restart NetworkManager
```

---

<a id="3"></a>
## 3. Método 1: Conexão Rápida via Terminal

Esta é a forma mais direta de se conectar, ideal para testes e uso em servidores.

1.  **Organize seus arquivos:**
    Coloque seu arquivo `cliente.ovpn` em uma pasta, por exemplo: `~/VPN/`.

2.  **Execute a conexão:**
    ```bash
    cd ~/VPN/
    sudo openvpn --config cliente.ovpn
    ```
    Mantenha este terminal aberto. Ele mostrará os logs da conexão. Para desconectar, pressione `Ctrl+C`.

**Se a sua VPN pede usuário e senha**, você pode automatizar isso:
1.  **Arquivo:** `~/VPN/auth.txt`
2.  **Comando:** `nano ~/VPN/auth.txt`
3.  **Conteúdo:**
    ```
    SEU_USUARIO
    SUA_SENHA
    ```
4.  **Proteja o arquivo:**
    ```bash
    chmod 600 ~/VPN/auth.txt
    ```
5.  **Edite seu `.ovpn`** (`nano cliente.ovpn`) e troque a linha `auth-user-pass` por:
    ```
    auth-user-pass auth.txt
    ```

---

<a id="4"></a>
## 4. Checklist Rápido de Verificação
Após conectar, abra um **novo terminal** e verifique se tudo está funcionando:

- **1. A interface do túnel existe?**
  ```bash
  ip addr show tun0
  ```
  *Você deve ver uma interface chamada `tun0` com um IP local (ex: 10.8.0.x).*

- **2. Qual é o seu IP público?**
  ```bash
  curl ifconfig.me ; echo
  ```
  *O IP exibido deve ser o do servidor VPN, não o da sua casa/escritório.*

- **3. O tráfego está saindo pela rota correta?**
  ```bash
  ip route
  ```
  *Você deve ver rotas como `0.0.0.0/1 via 10.8.0.x dev tun0`, indicando que o tráfego padrão está sendo enviado pelo túnel.*

Se todos os passos estiverem corretos, a VPN estará funcionando.

---

<a id="5"></a>
## 5. Método 2: Integrar com NetworkManager (Desktop)

Este método adiciona a VPN ao menu de rede do seu ambiente gráfico.

- **Importar via terminal (nmcli):**
  ```bash
  nmcli connection import type openvpn file ~/VPN/cliente.ovpn
  ```
- **Conectar:**
  ```bash
  # Encontre o nome da conexão
  nmcli connection show

  # Conecte usando o nome
  nmcli connection up "NOME_DA_CONEXAO"
  ```
- **Desconectar:**
  ```bash
  nmcli connection down "NOME_DA_CONEXAO"
  ```
Você também pode fazer o mesmo pela interface gráfica: **Configurações → Rede → VPN → “+” → Importar de arquivo…**.

---

<a id="6"></a>
## 6. Automação: Iniciar com o Sistema

**Opção A: Com `systemd` (ideal para servidores)**
1.  **Arquivo:** `/etc/openvpn/client/cliente.conf` (o nome antes de `.conf` é importante)
2.  **Comando:**
    ```bash
    sudo mkdir -p /etc/openvpn/client
    sudo cp ~/VPN/cliente.ovpn /etc/openvpn/client/cliente.conf
    ```
3.  **Habilite e inicie o serviço:**
    ```bash
    sudo systemctl enable --now openvpn-client@cliente.service
    ```
    A VPN irá conectar automaticamente a cada boot. Para ver os logs: `journalctl -u openvpn-client@cliente`.

**Opção B: Com NetworkManager (ideal para desktops)**
```bash
nmcli connection modify "NOME_DA_CONEXAO" connection.autoconnect yes
```

---

<a id="7"></a>
## 7. Dicas e Soluções (FAQ)

- **VPN conecta, mas não navego:** Quase sempre é um problema de DNS ou rota.
  - Verifique `ip route`. Se não houver uma rota padrão via `tun0`, o servidor pode não estar forçando (split-tunnel).
  - Verifique os DNS com `resolvectl status`.

- **Erro `TLS Error: TLS key negotiation failed`:**
  - Verifique se o endereço e a porta no campo `remote` do seu `.ovpn` estão corretos.
  - Pode ser um bloqueio de firewall. Tente mudar o protocolo de `udp` para `tcp` e a porta para `443` no seu `.ovpn`, se o servidor suportar.
  - Verifique se a data e hora do seu sistema estão corretas.

- **"DNS Leak" (Vazamento de DNS):**
  Se estiver usando `systemd-resolved`, o NetworkManager geralmente lida bem com isso. Se não, no perfil da conexão VPN no NetworkManager, vá em "IPv4" e "IPv6", marque "Usar esta conexão somente para recursos em sua rede" e configure manualmente um DNS seguro (ex: 1.1.1.1) se necessário.

- **Kill Switch (Bloquear tráfego se a VPN cair):**
  Use um firewall como o `ufw`. A lógica é:
  1. `sudo ufw default deny outgoing` (bloqueia tudo).
  2. `sudo ufw allow out on tun0` (permite tráfego só pela VPN).
  3. `sudo ufw allow out to IP_DO_SERVIDOR_VPN port PORTA` (permite a conexão inicial com o servidor).
  > **Aviso:** Isso é avançado e pode te trancar fora do sistema se mal configurado. Use com extremo cuidado.

---

## Referências (fontes para consulta)

### OpenVPN

- Manpage (OpenVPN 2.4): https://community.openvpn.net/openvpn/wiki/Openvpn24ManPage

### NetworkManager / systemd (referência)

- `nmcli(1)` (Debian manpage): https://manpages.debian.org/bookworm/network-manager/nmcli.1.en.html
- `systemd-resolved` / `resolvectl(1)`: https://manpages.debian.org/bookworm/systemd/resolvectl.1.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
