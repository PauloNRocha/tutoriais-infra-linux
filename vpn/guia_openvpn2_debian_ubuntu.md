# Guia Prático: Cliente OpenVPN 2 no Debian e Ubuntu

*Criado em: 29 de setembro de 2025*  
*Última atualização em: 21 de maio de 2026*

Anotei este guia para aquele cenário bem comum: a empresa, cliente ou fornecedor envia um arquivo `.ovpn`, no Windows a pessoa usaria o OpenVPN GUI, mas no Linux precisa conectar de forma simples e previsível. O foco aqui é o cliente OpenVPN 2, tanto pelo terminal quanto pelo NetworkManager no desktop e pelo `systemd` em servidor.

Não é um guia para criar servidor OpenVPN, gerar certificados ou desenhar uma topologia de VPN. A ideia é pegar um perfil já entregue e fazer ele funcionar bem em Debian/Ubuntu.

---

## Índice rápido

1. [OpenVPN 2 ou OpenVPN 3](#1)
2. [Pré-requisitos](#2)
3. [Instalação](#3)
4. [Método 1: conectar pelo terminal](#4)
5. [Usuário e senha no arquivo de autenticação](#5)
6. [Checklist de validação](#6)
7. [Método 2: importar no NetworkManager](#7)
8. [Iniciar automaticamente com systemd](#8)
9. [Troubleshooting](#9)
10. [Referências](#referencias)

---

<a id="1"></a>
## 1. OpenVPN 2 ou OpenVPN 3

Existem duas linhas que aparecem muito quando se pesquisa OpenVPN no Linux:

- **OpenVPN 2 (`openvpn`)**: cliente clássico, maduro, compatível com a maioria dos arquivos `.ovpn` entregues por empresas, firewalls, MikroTik, pfSense, OPNsense e provedores.
- **OpenVPN 3 (`openvpn3`)**: cliente mais novo, com comandos diferentes e outro fluxo de gerenciamento.

Neste guia eu uso **OpenVPN 2**, porque é o caminho mais próximo do OpenVPN GUI do Windows e costuma ser o mais direto para aproveitar um arquivo `.ovpn` já pronto.

---

<a id="2"></a>
## 2. Pré-requisitos

Você precisa ter em mãos:

- arquivo `cliente.ovpn`;
- certificados e chaves, se não estiverem embutidos dentro do `.ovpn`;
- usuário e senha, se a VPN usar `auth-user-pass`;
- acesso `sudo` no Linux.

Crie uma pasta local para organizar os arquivos:

```bash
mkdir -p "$HOME/VPN"
```

Copie o arquivo `.ovpn` para essa pasta e confira:

```bash
ls -l "$HOME/VPN"
```

> [!CAUTION]
> Arquivos `.ovpn`, certificados, chaves e senhas podem dar acesso à rede interna. Não deixe esses arquivos em diretórios compartilhados, backup público ou pasta sincronizada sem criptografia.

---

<a id="3"></a>
## 3. Instalação

Instale o cliente OpenVPN 2 e ferramentas básicas para teste:

```bash
sudo apt update
sudo apt install -y openvpn curl
```

Confira a versão:

```bash
openvpn --version
```

Para usar no desktop pelo gerenciador de rede do GNOME, Cinnamon, KDE ou outro ambiente que use NetworkManager, instale também:

```bash
sudo apt install -y network-manager-openvpn network-manager-openvpn-gnome
```

Se a opção de importar OpenVPN não aparecer no ambiente gráfico depois da instalação, reinicie a sessão. Em último caso, reinicie o NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

> [!CAUTION]
> Reiniciar o NetworkManager derruba e reconecta interfaces gerenciadas por ele. Em desktop normalmente é tranquilo; em servidor remoto, valide antes para não perder acesso.

---

<a id="4"></a>
## 4. Método 1: conectar pelo terminal

Este é o melhor primeiro teste, porque mostra o log completo da conexão.

Entre na pasta onde está o perfil:

```bash
cd "$HOME/VPN"
```

Conecte:

```bash
sudo openvpn --config cliente.ovpn
```

Mantenha o terminal aberto. Ele fica preso nos logs da VPN, e isso é normal. Para continuar usando o computador, abra outro terminal.

Para desconectar, pressione `Ctrl+C` no terminal onde o OpenVPN está rodando.

Para conectar novamente depois:

```bash
cd "$HOME/VPN"
sudo openvpn --config cliente.ovpn
```

Se a intenção for deixar a VPN conectada sem depender de uma janela de terminal, use o método com `systemd` mais abaixo.

Quando funcionar, você deve ver algo parecido com:

```text
Initialization Sequence Completed
```

Se aparecer erro de certificado, senha, rota ou DNS, resolva pelo terminal primeiro. Só depois importe no NetworkManager ou coloque no `systemd`.

---

<a id="5"></a>
## 5. Usuário e senha no arquivo de autenticação

Algumas VPNs pedem usuário e senha além do certificado. Nesse caso, o arquivo `.ovpn` costuma ter esta linha:

```text
auth-user-pass
```

Para teste manual, o OpenVPN pode pedir usuário e senha no terminal. Para automação, crie um arquivo separado:

```bash
nano "$HOME/VPN/auth.txt"
```

Conteúdo:

```text
SEU_USUARIO
SUA_SENHA
```

Proteja o arquivo:

```bash
chmod 600 "$HOME/VPN/auth.txt"
```

No `cliente.ovpn`, troque:

```text
auth-user-pass
```

por:

```text
auth-user-pass auth.txt
```

Teste novamente:

```bash
cd "$HOME/VPN"
sudo openvpn --config cliente.ovpn
```

> [!CAUTION]
> Salvar senha em arquivo facilita automação, mas aumenta risco. Use somente em máquina confiável e proteja permissões do arquivo.

---

<a id="6"></a>
## 6. Checklist de validação

Depois de conectar, abra outro terminal e valide.

### 6.1 Interface do túnel

```bash
ip addr show tun0
```

Se a interface tiver outro nome, liste interfaces do tipo túnel:

```bash
ip -brief addr
```

### 6.2 Rotas

```bash
ip route
```

Em VPN full tunnel, é comum aparecer rota padrão ou rotas como `0.0.0.0/1` e `128.0.0.0/1` pelo túnel.

Em split tunnel, pode aparecer apenas a rota da rede interna, por exemplo `10.0.0.0/8`, `172.16.0.0/12` ou uma rede específica da empresa.

### 6.3 IP público

```bash
curl ifconfig.me
```

Se a VPN for full tunnel, o IP deve mudar para o IP de saída da VPN. Se for split tunnel, o IP público pode continuar sendo o da sua rede normal, e isso não significa erro.

### 6.4 DNS

```bash
resolvectl status
```

Confira se a VPN adicionou DNS esperado ou domínio de busca, quando isso fizer parte da configuração.

---

<a id="7"></a>
## 7. Método 2: importar no NetworkManager

Este método é mais confortável para desktop, porque a VPN passa a aparecer no menu de rede.

Importe o arquivo:

```bash
nmcli connection import type openvpn file "$HOME/VPN/cliente.ovpn"
```

Liste as conexões:

```bash
nmcli connection show
```

Conecte:

```bash
nmcli connection up "NOME_DA_CONEXAO"
```

Desconecte:

```bash
nmcli connection down "NOME_DA_CONEXAO"
```

Também é possível importar pela interface gráfica:

```text
Configurações -> Rede -> VPN -> + -> Importar de arquivo
```

Se o perfil usa usuário e senha, talvez o NetworkManager peça as credenciais na primeira conexão.

---

<a id="8"></a>
## 8. Iniciar automaticamente com systemd

Este método faz sentido em servidor ou máquina que precisa manter a VPN sempre ativa sem depender do login gráfico.

No Debian/Ubuntu, o serviço `openvpn-client@NOME.service` espera um arquivo em:

```text
/etc/openvpn/client/NOME.conf
```

Crie o diretório:

```bash
sudo mkdir -p /etc/openvpn/client
```

Copie o perfil:

```bash
sudo cp "$HOME/VPN/cliente.ovpn" /etc/openvpn/client/cliente.conf
```

Se o `.ovpn` usa arquivos externos, como `ca.crt`, `client.crt`, `client.key`, `ta.key` ou `auth.txt`, copie esses arquivos também:

```bash
sudo cp "$HOME/VPN/ca.crt" /etc/openvpn/client/
sudo cp "$HOME/VPN/client.crt" /etc/openvpn/client/
sudo cp "$HOME/VPN/client.key" /etc/openvpn/client/
sudo cp "$HOME/VPN/ta.key" /etc/openvpn/client/
sudo cp "$HOME/VPN/auth.txt" /etc/openvpn/client/
```

Ajuste permissões do arquivo principal:

```bash
sudo chown root:root /etc/openvpn/client/cliente.conf
sudo chmod 600 /etc/openvpn/client/cliente.conf
```

Se houver chave privada ou arquivo de senha, proteja cada arquivo existente. Exemplos:

```bash
sudo chmod 600 /etc/openvpn/client/client.key
sudo chmod 600 /etc/openvpn/client/ta.key
sudo chmod 600 /etc/openvpn/client/auth.txt
```

Use apenas os comandos dos arquivos que existem no seu perfil. O importante é que tudo que o `.conf` referencia esteja acessível dentro de `/etc/openvpn/client/`.

Inicie a conexão:

```bash
sudo systemctl enable --now openvpn-client@cliente.service
```

Valide:

```bash
systemctl status openvpn-client@cliente.service --no-pager
journalctl -u openvpn-client@cliente.service -f
```

Para parar:

```bash
sudo systemctl disable --now openvpn-client@cliente.service
```

---

<a id="9"></a>
## 9. Troubleshooting

### 9.1 Erro de autenticação

Confira se a VPN usa `auth-user-pass`:

```bash
grep 'auth-user-pass' "$HOME/VPN/cliente.ovpn"
```

Se estiver usando `auth.txt`, confirme permissões:

```bash
ls -l "$HOME/VPN/auth.txt"
```

O esperado é algo como:

```text
-rw-------
```

### 9.2 Erro de arquivo não encontrado no systemd

Veja os logs:

```bash
journalctl -u openvpn-client@cliente.service -n 80 --no-pager
```

Normalmente isso acontece quando o `.conf` aponta para arquivo relativo que não foi copiado para `/etc/openvpn/client/`.

Confira diretório e arquivos:

```bash
ls -l /etc/openvpn/client
```

### 9.3 TLS key negotiation failed

Esse erro costuma indicar problema de comunicação com o servidor.

Confira no `.ovpn`:

```bash
grep '^remote ' "$HOME/VPN/cliente.ovpn"
grep '^proto ' "$HOME/VPN/cliente.ovpn"
```

Pontos comuns:

- endereço ou porta errada;
- UDP bloqueado no caminho;
- servidor fora do ar;
- firewall bloqueando;
- relógio do sistema muito errado.

Confira horário:

```bash
timedatectl
```

### 9.4 Diagnosticar VPN importada no NetworkManager

Quando a VPN foi importada no NetworkManager, a interface gráfica pode mostrar apenas uma mensagem genérica. Para ver melhor o erro, teste pelo terminal.

Liste as conexões:

```bash
nmcli connection show
```

Tente subir a VPN pelo `nmcli`:

```bash
nmcli connection up "NOME_DA_VPN"
```

Acompanhe o log do NetworkManager em outro terminal:

```bash
sudo journalctl -fu NetworkManager
```

Ou consulte os logs do boot atual com filtro:

```bash
sudo journalctl -u NetworkManager -b --no-pager | grep -iE 'vpn|openvpn|tls|cert|certificate|expired|failed|error|timeout|auth|password' | tail -n 120
```

Se o filtro não mostrar nada útil, rode o `journalctl` sem o `grep` e leia os eventos próximos ao horário da tentativa.

### 9.5 Certificado expirado

Quando o log mostrar algo parecido com:

```text
sslv3 alert certificate expired
TLS handshake failed
```

algum certificado usado na conexão expirou ou está fora do período de validade.

Confira primeiro a data e hora do computador:

```bash
date
timedatectl
```

Depois confira a validade dos certificados locais, quando eles existirem como arquivos separados:

```bash
openssl x509 -in "$HOME/VPN/client.crt" -noout -subject -issuer -dates
openssl x509 -in "$HOME/VPN/ca.crt" -noout -subject -issuer -dates
```

Observe principalmente o campo:

```text
notAfter=
```

Se o perfil usa arquivo `.p12`, confira assim:

```bash
openssl pkcs12 -in "$HOME/VPN/client.p12" -nokeys -clcerts | openssl x509 -noout -subject -issuer -dates
```

Se o certificado do usuário expirou, solicite ou gere um novo certificado e atualize o perfil da VPN. Se o certificado estiver embutido dentro do `.ovpn`, normalmente o fornecedor precisa entregar um `.ovpn` atualizado ou orientar a troca correta do bloco de certificado.

### 9.6 Atualizar certificado em VPN já importada no NetworkManager

Quando a VPN já existe no NetworkManager e apenas o certificado foi renovado, nem sempre é necessário apagar e importar tudo de novo.

Organize os arquivos novos em uma pasta própria:

```bash
mkdir -p "$HOME/.cert/vpn-suporte"
chmod 700 "$HOME/.cert/vpn-suporte"
```

Copie os certificados e chaves novos para essa pasta. Exemplo:

```bash
cp "$HOME/Downloads/ca.crt" "$HOME/.cert/vpn-suporte/"
cp "$HOME/Downloads/client.crt" "$HOME/.cert/vpn-suporte/"
cp "$HOME/Downloads/client.key" "$HOME/.cert/vpn-suporte/"
```

Ajuste permissões conforme os arquivos existentes:

```bash
chmod 644 "$HOME/.cert/vpn-suporte/ca.crt"
chmod 644 "$HOME/.cert/vpn-suporte/client.crt"
chmod 600 "$HOME/.cert/vpn-suporte/client.key"
```

Antes de editar, faça backup do perfil do NetworkManager. Primeiro localize o arquivo:

```bash
sudo ls -l /etc/NetworkManager/system-connections/
```

Depois copie o arquivo da VPN para um local seguro. Exemplo:

```bash
sudo mkdir -p /root/backup-networkmanager-vpn
sudo cp -av "/etc/NetworkManager/system-connections/NOME_DA_VPN.nmconnection" /root/backup-networkmanager-vpn/
```

Abra o editor de conexões:

```bash
nm-connection-editor
```

Edite a VPN e atualize os campos conforme o perfil usa:

```text
CA Certificate
User Certificate
Private Key
```

Depois teste:

```bash
nmcli connection up "NOME_DA_VPN"
```

### 9.7 Timeout por protocolo errado: TCP x UDP

Quando o OpenVPN mostra timeout, confira se cliente e servidor estão usando o mesmo protocolo. Cliente e servidor precisam estar coerentes em protocolo e porta.

No arquivo `.ovpn`:

```bash
grep -E '^remote |^proto ' "$HOME/VPN/cliente.ovpn"
```

Exemplos:

```text
proto tcp-client
remote 203.0.113.10 62491
```

ou:

```text
proto udp
remote 203.0.113.10 62491
```

No NetworkManager, veja os dados do perfil:

```bash
nmcli connection show "NOME_DA_VPN" | grep -Ei 'vpn.data|remote|proto|port'
```

Se o servidor estiver em TCP e o cliente tentar UDP, a porta pode estar correta e mesmo assim a VPN não fecha.

Para testar porta TCP, instale o `nc` se necessário:

```bash
sudo apt install -y netcat-openbsd
```

Teste:

```bash
nc -vz IP_DO_SERVIDOR PORTA
```

Exemplo:

```bash
nc -vz 203.0.113.10 62491
```

> [!NOTE]
> `nc -vz` testa TCP. Ele não confirma funcionamento de OpenVPN em UDP.

### 9.8 Derrubar conexão presa no NetworkManager

Para desconectar pelo NetworkManager:

```bash
nmcli connection down "NOME_DA_VPN"
```

Veja se ainda existe processo do OpenVPN iniciado pelo NetworkManager:

```bash
pgrep -a nm-openvpn
```

Se o processo ficou preso, encerre como último recurso:

```bash
sudo pkill -f nm-openvpn
```

Se ainda assim o estado do NetworkManager ficar inconsistente, reinicie o serviço:

```bash
sudo systemctl restart NetworkManager
```

> [!CAUTION]
> Reiniciar o NetworkManager derruba e reconecta interfaces gerenciadas por ele. Em máquina remota, isso pode cortar seu acesso.

### 9.9 Conecta, mas não navega

Confira rota e DNS:

```bash
ip route
resolvectl status
```

Se a VPN for split tunnel, talvez ela não deva alterar a navegação geral. Nesse caso, só redes internas passam pela VPN.

### 9.10 DNS leak

DNS leak depende do desenho da VPN. Antes de mexer, entenda se a VPN é full tunnel ou split tunnel.

No NetworkManager, a opção “Usar esta conexão somente para recursos em sua rede” cria comportamento de split tunnel. Não marque isso se a intenção for mandar todo o tráfego pela VPN.

### 9.11 Kill switch

Kill switch é uma regra de firewall para impedir saída fora da VPN. Não recomendo copiar comandos genéricos sem adaptar ao ambiente, porque isso pode derrubar acesso remoto ou bloquear atualizações.

Em servidor, trate kill switch em um guia próprio de firewall, de preferência com `nftables`, política de rollback e acesso de emergência.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### OpenVPN

- OpenVPN 2.6 Manual: https://openvpn.net/community-docs/community-articles/openvpn-2-6-manual.html
- Debian Wiki - OpenVPN: https://wiki.debian.org/OpenVPN

### NetworkManager / systemd

- `nmcli` Reference Manual: https://networkmanager.pages.freedesktop.org/NetworkManager/NetworkManager/nmcli.html
- `resolvectl(1)` Debian: https://manpages.debian.org/bookworm/systemd/resolvectl.1.en.html
- `openssl-x509(1)` OpenSSL: https://docs.openssl.org/3.3/man1/openssl-x509/

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
