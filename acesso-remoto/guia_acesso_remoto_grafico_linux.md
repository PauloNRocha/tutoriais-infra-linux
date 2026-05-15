# Guia de Produção: Acesso Remoto Gráfico no Debian 12 (XRDP, AnyDesk, TeamViewer)

*Criado em: 04 de dezembro de 2025*  
*Última atualização em: 15 de maio de 2026*

Tem situação em que SSH não basta e eu realmente preciso ver a interface gráfica do servidor. Este guia junta o caminho que usei no Debian 12 para isso, cobrindo **XRDP**, **AnyDesk** e **TeamViewer**, além dos problemas mais comuns que aparecem quando esse tipo de acesso começa a dar trabalho.

Não é um guia de hardening completo para expor acesso gráfico na internet. A ideia aqui é instalação, uso controlado e troubleshooting.

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

- **Use um Firewall:** Libere apenas os IPs necessários ou, no mínimo, apenas a porta que você realmente vai usar (ex.: `3389/tcp` para XRDP).
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

1. Atualize a lista de pacotes:

```bash
sudo apt update
```

2. Instale o XFCE:

```bash
sudo apt install xfce4 xfce4-goodies
```

### 2.3 Instalar e Configurar o XRDP

1. Instale o serviço:

```bash
sudo apt install xrdp
```

2. Ajuste a sessão padrão para usar XFCE:

```bash
echo "startxfce4" | sudo tee /etc/skel/.xsession
echo "startxfce4" | sudo tee ~/.xsession
```

3. Se o servidor usa `nftables`, libere o acesso apenas da rede ou IP de gestão.

Exemplo:

```bash
sudo nft add rule inet filtro input ip saddr 192.0.2.10 tcp dport 3389 counter accept
```

Substitua `192.0.2.10` pelo IP público ou pela rede autorizada a acessar o RDP.

4. Reinicie o serviço:

```bash
sudo systemctl restart xrdp
```

Observação:

- se você já tiver uma policy `drop`, confirme se a regra entrou na chain correta;
- evite liberar `3389/tcp` para a internet inteira;
- em ambiente de produção, prefira acessar o XRDP por VPN ou por uma rede de gestão conhecida.

### 2.4 Solução de Problemas do XRDP
- **Conflito de Sessões:** Para evitar problemas ao logar com o mesmo usuário local e remotamente, a recomendação é **usar um usuário dedicado para o acesso remoto**.
- **Erro "Could not start log":** Geralmente é um problema de permissão. Execute os seguintes comandos para corrigir, caso receba a mensagem:
  ```bash
  sudo chown xrdp:xrdp /var/log/xrdp.log
  sudo chmod 640 /var/log/xrdp.log
  ```

---

<a id="3"></a>
## 3. Acesso via AnyDesk ou TeamViewer

Estas ferramentas são mais fáceis de configurar, pois fazem a "ponte" através dos servidores da empresa, evitando a necessidade de abrir portas no firewall.

O ponto fraco é outro: são ferramentas proprietárias, dependem da infraestrutura de terceiros e podem ter regras de licença para uso profissional. Eu costumo tratar AnyDesk e TeamViewer como alternativa prática para suporte pontual, não como substituto de um acesso bem controlado por VPN, SSH ou rede de gestão.

> **Nota sobre Versões:** Os links de download e métodos de instalação podem mudar. Se os comandos abaixo falharem, sempre consulte o site oficial do AnyDesk ou TeamViewer para obter as instruções mais recentes.

### 3.1 Instalar AnyDesk

1. Baixe o pacote `.deb` apropriado no site oficial do AnyDesk.

Exemplo:

```bash
cd /tmp
wget https://download.anydesk.com/linux/anydesk_8.0.0-1_amd64.deb
```

2. Instale com o `apt`:

```bash
sudo apt install ./anydesk_8.0.0-1_amd64.deb
```

Depois da instalação, execute `anydesk` no terminal para iniciar e obter o ID de conexão.

Observação:

- o uso de `apt-key` foi removido deste guia porque essa abordagem está obsoleta;
- se a versão do pacote mudar no site oficial, ajuste o nome do arquivo baixado antes de instalar.

### 3.2 Instalar TeamViewer

1. Entre em `/tmp` e baixe o pacote:

```bash
cd /tmp
wget https://download.teamviewer.com/download/linux/teamviewer_amd64.deb
```

2. Instale o pacote:

```bash
sudo apt install ./teamviewer_amd64.deb
```
Após a instalação, execute `teamviewer` para configurar o acesso.

Observação:

- verifique a licença aplicável ao seu tipo de uso antes de usar em ambiente profissional;
- se a máquina for de produção, documente quem tem acesso e desative o acesso quando não for mais necessário.

---

<a id="4"></a>
## 4. Solução de Problemas Comuns

### 4.1 Como forçar o encerramento de uma sessão remota
Se um usuário ficou com a sessão "presa", tente primeiro encerrar os processos de forma normal.

```bash
# Substitua NOME_DO_USUARIO pelo login do usuário
sudo pkill -TERM -u NOME_DO_USUARIO
```

Se não resolver, use `KILL` como último recurso:

```bash
sudo pkill -KILL -u NOME_DO_USUARIO
```

Isso encerra todos os processos daquele usuário, desconectando a sessão. Use com cuidado, porque processos abertos podem perder dados não salvos.

### 4.2 Encerramento de VMs ao logar remotamente
Se suas máquinas virtuais (Virt-Manager, etc.) são encerradas ao iniciar uma sessão gráfica, pode ser um conflito de recursos (principalmente RAM).
- **Solução**: Verifique o consumo de recursos com `htop`. Considere administrar as VMs pela linha de comando (`virsh`) para economizar recursos em servidores.

---

## Conclusão

Esse tipo de acesso ajuda bastante em manutenção pontual, mas eu ainda trataria SSH como método principal e deixaria acesso gráfico só para quando realmente fizer sentido. Se for expor isso fora da rede local, o mínimo é colocar por trás de firewall e, de preferência, VPN.

Veja também: [guia de produção de acesso SSH por chave pública](./guia_producao_ssh_chave_publica_linux.md)

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

- `nft(8)` (manpage Debian): https://manpages.debian.org/bookworm/nftables/nft.8.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
