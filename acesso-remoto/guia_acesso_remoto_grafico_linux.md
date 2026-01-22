# Guia de Acesso Remoto Gráfico no Debian 12 (XRDP, AnyDesk, TeamViewer)

*Criado em 04 de dezembro de 2025 e atualizado em 08 de dezembro de 2025*

Este guia mostra como configurar acesso remoto gráfico em um servidor Debian 12, permitindo acesso por interface gráfica de forma semelhante ao que se faz em servidores Windows. O conteúdo cobre instalação e configuração de **XRDP**, **AnyDesk** e **TeamViewer**, além de soluções para problemas comuns.

---

## Índice rápido
1. [Checklist de Segurança Essencial](#1)
2. [Acesso via XRDP (Remote Desktop Protocol)](#2)
3. [Acesso via AnyDesk ou TeamViewer](#3)
4. [Solução de Problemas Comuns](#4)

---

<a id="1"></a>
## 1. Checklist de Segurança Essencial
Antes de expor qualquer serviço de acesso remoto à internet, siga estas recomendações:

- **Use um Firewall:** Ative e configure um firewall como o `UFW` para permitir acesso apenas de IPs conhecidos ou, no mínimo, apenas na porta que você vai usar (ex: porta `3389` para RDP).
- **Senhas Fortes:** Todos os usuários que podem logar remotamente devem ter senhas fortes e únicas.
- **Usuários Não-Root:** Evite logar diretamente com o usuário `root`. Use um usuário padrão e utilize `sudo` quando precisar de privilégios elevados.
- **Mantenha o Sistema Atualizado:** Aplique atualizações de segurança regularmente com `sudo apt update && sudo apt upgrade`.

---

<a id="2"></a>
## 2. Acesso via XRDP (Remote Desktop Protocol)

O XRDP é uma opção comum por usar o protocolo RDP nativo, presente em qualquer Windows.

### 2.1 Nota sobre Servidores Headless (Sem Monitor)
Para que o XRDP e outras ferramentas gráficas funcionem, é preciso instalar um ambiente de desktop. Isso adiciona uma sobrecarga de recursos (RAM e CPU) ao servidor. Se seu servidor tem recursos limitados, instale um ambiente leve.

### 2.2 Instalar Ambiente Gráfico Leve (XFCE)
Recomendação: **XFCE** (leve e adequado para servidores com poucos recursos).
```bash
sudo apt update
sudo apt install xfce4 xfce4-goodies -y
```

### 2.3 Instalar e Configurar o XRDP
```bash
# Instalar o serviço
sudo apt install xrdp -y

# Dizer ao XRDP para usar a sessão XFCE
echo "startxfce4" | sudo tee /etc/skel/.xsession
echo "startxfce4" | sudo tee ~/.xsession

# Abrir a porta no firewall (se estiver usando UFW)
sudo ufw allow 3389/tcp

# Reiniciar o serviço para aplicar as configurações
sudo systemctl restart xrdp
```

### 2.4 Solução de Problemas do XRDP
- **Conflito de Sessões:** Para evitar problemas ao logar com o mesmo usuário local e remotamente, a recomendação é **usar um usuário dedicado para o acesso remoto**.
- **Erro "Could not start log":** Geralmente é um problema de permissão. Execute os seguintes comandos para corrigir:
  ```bash
  sudo chown xrdp:xrdp /var/log/xrdp.log
  sudo chmod 640 /var/log/xrdp.log
  ```

---

<a id="3"></a>
## 3. Acesso via AnyDesk ou TeamViewer

Estas ferramentas são mais fáceis de configurar, pois fazem a "ponte" através dos servidores da empresa, evitando a necessidade de abrir portas no firewall.

> **Nota sobre Versões:** Os links de download e métodos de instalação podem mudar. Se os comandos abaixo falharem, sempre consulte o site oficial do AnyDesk ou TeamViewer para obter as instruções mais recentes.

### 3.1 Instalar AnyDesk
```bash
# Baixar a chave de repositório e adicionar
wget -qO - https://keys.anydesk.com/repos/DEB-GPG-KEY | sudo apt-key add -
echo "deb http://deb.anydesk.com/ all main" | sudo tee /etc/apt/sources.list.d/anydesk-stable.list

# Instalar
sudo apt update
sudo apt install anydesk -y
```
Após a instalação, execute `anydesk` no terminal para iniciar e obter seu ID de conexão.

### 3.2 Instalar TeamViewer
```bash
# Baixar o pacote .deb para a pasta /tmp
cd /tmp
wget https://download.teamviewer.com/download/linux/teamviewer_amd64.deb

# Instalar o pacote e suas dependências
sudo apt install ./teamviewer_amd64.deb -y
```
Após a instalação, execute `teamviewer` para configurar o acesso.

---

<a id="4"></a>
## 4. Solução de Problemas Comuns

### 4.1 Como forçar o encerramento de uma sessão remota
Se um usuário ficou com a sessão "presa", você pode derrubá-lo com o comando `pkill`.

```bash
# Substitua NOME_DO_USUARIO pelo login do usuário
sudo pkill -9 -u NOME_DO_USUARIO
```
Isso força o encerramento de todos os processos daquele usuário, desconectando-o.

### 4.2 Encerramento de VMs ao logar remotamente
Se suas máquinas virtuais (Virt-Manager, etc.) são encerradas ao iniciar uma sessão gráfica, pode ser um conflito de recursos (principalmente RAM).
- **Solução**: Verifique o consumo de recursos com `htop`. Considere administrar as VMs pela linha de comando (`virsh`) para economizar recursos em servidores.

---

### Conclusão
Configurar acesso remoto gráfico é simples, mas exige cuidado com segurança. Em servidores expostos à internet, a recomendação é usar **acesso via SSH** como método principal e habilitar acesso gráfico apenas quando necessário, preferencialmente protegido por firewall ou VPN.

---

## Referências (fontes para consulta)

### XRDP

- Projeto XRDP: https://xrdp.org/
- `xrdp(8)` (manpage Debian): https://manpages.debian.org/bookworm/xrdp/xrdp.8.en.html

### AnyDesk / TeamViewer (oficial)

- AnyDesk — instalação: https://support.anydesk.com/docs/installation
- TeamViewer — download para Linux: https://www.teamviewer.com/en/download/linux/

### Debian Wiki (complementar)

- TeamViewer no Debian: https://wiki.debian.org/TeamViewer

### Firewall (referência)

- `ufw(8)` (manpage Debian): https://manpages.debian.org/bookworm/ufw/ufw.8.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
