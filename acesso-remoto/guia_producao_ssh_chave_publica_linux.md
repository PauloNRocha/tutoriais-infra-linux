# Guia de Produção: Acesso SSH por chave pública (porta customizada) em servidores Linux

*Criado em: 30 de janeiro de 2026*  

Quando o ambiente começa a ter mais de um servidor, mais de uma chave e mais de um operador, acesso SSH vira bagunça muito fácil. Este guia registra o padrão que eu prefiro para produção: **chave pública**, **porta customizada**, **usuário comum + sudo** e nada de login direto como `root`.

A intenção aqui é deixar o acesso previsível e fácil de manter: saber **qual chave** está em uso, reduzir risco de lockout e evitar confusão no cliente quando existem várias identidades.

---

## Índice rápido
1. [Conceito](#1)
2. [Por que não reutilizar a chave do GitHub](#2)
3. [Preparação no cliente](#3)
4. [Instalar a chave no servidor](#4)
5. [Endurecimento no servidor](#5)
6. [Organização de acessos com aliases](#6)
7. [Como o SSH aplica a configuração quando você usa alias (Host vs alias)](#7)
8. [Boas práticas](#8)
9. [O que NÃO fazer](#9)
10. [Checklist final](#10)

---

<a id="1"></a>
## 1) Conceito

No SSH por chave, você tem:

- **Chave privada**: fica no cliente (sua máquina). É o segredo que prova sua identidade para o servidor e **não deve ser compartilhado**.
- **Chave pública**: fica no servidor, no `~/.ssh/authorized_keys` do usuário remoto. Ela diz ao servidor: “aceite login deste usuário quando o cliente provar que tem a chave privada correspondente”.

Com isso você consegue desativar autenticação por senha no servidor, reduzindo brute force e removendo senha como “plano B”.

---

<a id="2"></a>
## 2) Por que NÃO reutilizar a chave do GitHub para servidores

É comum ter uma chave para GitHub/GitLab. Reutilizar a mesma chave para acessar servidores é má prática porque:

- aumenta o “raio de explosão” (vazou uma, compromete código **e** infra);
- dificulta rotação/revogação (você mistura ciclos de vida);
- piora auditoria (não fica claro o que é chave “infra” vs “dev”).

Diretriz: **use uma chave dedicada para infraestrutura**, separada da sua chave de repositórios.

---

<a id="3"></a>
## 3) Preparação no cliente, sua máquina

### 3.1 Criar o diretório e permissões básicas

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

### 3.2 Criar uma chave dedicada Ed25519 para infraestrutura

Exemplo: use um nome explícito para não confundir com outras chaves:

```bash
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519_infra -C "infra-$(whoami)@$(hostname)"
```

Isso cria `~/.ssh/id_ed25519_infra` (chave privada) e `~/.ssh/id_ed25519_infra.pub` (chave pública).

O nome explícito evita confusão quando você também possui chaves para GitHub, automação ou outros ambientes.

Sobre passphrase: em estação de trabalho, **use passphrase**. Para automação, o desenho muda, não é o foco aqui.

### 3.3 Permissões da chave

```bash
chmod 600 ~/.ssh/id_ed25519_infra
chmod 644 ~/.ssh/id_ed25519_infra.pub
```

---

<a id="4"></a>
## 4) Instalar a chave no servidor com porta não padrão

Premissa: você tem um usuário no servidor (ex.: `operador`) com acesso inicial (senha temporária, console, etc.).

### 4.1 Copiar a chave pública (porta customizada)

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_infra.pub -p 2222 operador@IP_OU_HOST
```

Atenção à ordem dos parâmetros: a porta (`-p`) deve vir antes do `usuario@host`.

Alternativa equivalente: `ssh-copy-id ... -o "Port=2222" ...`

### 4.2 Testar forçando a chave correta antes de mexer no servidor

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519_infra -o IdentitiesOnly=yes operador@IP_OU_HOST
```

Resultado esperado:

- não pede a senha do usuário remoto;
- pode pedir a passphrase da sua chave se você configurou.

Se for solicitado ainda para colocar senha, **não avance** para desativar a senha no servidor.

---

<a id="5"></a>
## 5) Endurecimento no servidor sem senha e sem root

### 5.1 Ajustar `sshd_config`

No servidor, edite:

```bash
sudo nano /etc/ssh/sshd_config
```

Ajuste (ou garanta) pelo menos:

```text
Port 2222
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Observações:

- A porta (`Port 2222`) é exemplo. Use a sua porta real.
- Não habilite login direto como `root`. Use usuário comum + `sudo`.
- Em algumas distros antigas ou imagens customizadas, pode existir `ChallengeResponseAuthentication yes`. Isso pode conflitar com `KbdInteractiveAuthentication` e manter uma forma de autenticação “interativa” ativa.
  - Se existir `ChallengeResponseAuthentication`, desative também (`ChallengeResponseAuthentication no`).

### 5.2 Validar sintaxe e recarregar sem derrubar sessões

Valide a configuração:

```bash
sudo sshd -t
```

`sshd -t` valida apenas **a sintaxe**, não a lógica, não garante que você conseguirá autenticar por chave.

Recarregue o serviço, Debian/Ubuntu normalmente usam `ssh`:

```bash
sudo systemctl reload ssh
```

Boa prática operacional: mantenha uma sessão SSH já conectada aberta e teste uma nova conexão em paralelo antes de fechar a sessão atual.

### 5.3 Permissões no servidor (usuário alvo)

No servidor, para o usuário que vai logar (ex.: `operador`):

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

E garanta dono correto:

```bash
chown -R "$USER:$USER" ~/.ssh
```

Esse comando deve ser executado **logado como o usuário alvo** (ex.: `operador`), porque ele usa a variável `$USER`. Nesse contexto, ele está correto.

Se você executar como `root`, ajuste para o usuário real, exemplo:

```bash
chown -R operador:operador ~operador/.ssh
```

---

<a id="6"></a>
## 6) Organização de acessos com aliases (arquivo + `.bashrc`)

Padrão adotado aqui:

- A **fonte da verdade** é o arquivo `~/.ssh/config` (IP/host, porta, usuário e qual chave usar).
- Os aliases (ex.: `~/.servidores`) são **apenas atalhos** para chamar `ssh <Host>` definido no `~/.ssh/config` (sem repetir IP/porta/chave).

### 6.1 Criar/ajustar `~/.ssh/config` (fonte da verdade)

Edite:

```bash
nano ~/.ssh/config
```

Exemplo, cole o conteúdo e ajuste `HostName`, `Port`, `User` e `IdentityFile` para o seu ambiente:

```text
# ~/.ssh/config

# Padrão em todos os servidores de usando mesma chave
Host servidor-*

# Se a sua chave tem outro nome (ex.: ~/.ssh/infra_ed25519), ajuste aqui.
  IdentityFile ~/.ssh/id_ed25519_infra
  IdentitiesOnly yes

# Cada servidor pode ter porta diferente. Isso fica aqui, em cada Host.

# Servidores DNS Autoritativo
Host servidor-bind9
  HostName ns1.exemplo.com.br
  Port 2222
  User dnsauth

# Servidores DNS Recursivo
Host servidor-unbound
  HostName rec.exemplo.com.br
  Port 22053
  User dnsrec

# Servidores
Host servidor-ntp
  HostName ntp.exemplo.com.br
  Port 22
  User ntpops

Host servidor-wazuh
  HostName wazuh.exemplo.com.br
  Port 22
  User secops

Host servidor-krill
  HostName rpki.exemplo.com.br
  Port 22051
  User rpkiops
```

Ajuste as permissões:

```bash
chmod 600 ~/.ssh/config
```

Teste direto pelo Host, para confirmar que o SSH está aplicando o `~/.ssh/config`:

```bash
ssh servidor-unbound
```

Dica opcional, quando necessário, modo debug:

```bash
ssh -vvv servidor-unbound
```

### 6.2 Criar `~/.servidores` (atalhos)

Edite:

```bash
nano ~/.servidores
```

Conteúdo sugerido (EXEMPLO):

```bash
# ~/.servidores
# Aliases (atalhos) para chamar apenas `ssh <Host>` do ~/.ssh/config

alias bind9="ssh servidor-bind9"
alias unbound="ssh servidor-unbound"
alias ntp="ssh servidor-ntp"
alias wazuh="ssh servidor-wazuh"
alias krill="ssh servidor-krill"
```

### 6.3 Carregar no `.bashrc`

No final do `~/.bashrc`, carregue direto:

```bash
source ~/.servidores
```

Nota: usar `if [ -f ~/.servidores ]; then ... fi` é uma proteção defensiva (ex.: máquinas onde esse arquivo ainda não existe). Em operação individual, `source ~/.servidores` direto costuma ser suficiente e mais simples.

Recarregue:

```bash
source ~/.bashrc
```

Teste:

```bash
unbound
```

---

<a id="7"></a>
## 7) Como o SSH aplica a configuração quando você usa alias (Host vs alias)

### 7.1 O que roda primeiro
- **Alias** é expandido pelo **shell**, antes do SSH.
- O SSH aplica a configuração quando o comando é `ssh <Host>`, e o `<Host>` bate com uma entrada do `~/.ssh/config`.

### 7.2 Como esse padrão evita confusão automaticamente

Se o alias chama apenas `ssh servidor-...`, você evita a armadilha de duplicar IP/porta/chave em dois lugares.

Regra prática:

- `~/.ssh/config` define **HostName**, **Port**, **User**, `IdentityFile` e `IdentitiesOnly`.
- `~/.servidores` define apenas atalhos para `ssh <Host>`.

Exemplo mental:

1) você digita `unbound` (o shell expande para `ssh servidor-unbound`)  
2) o SSH aplica o bloco `Host servidor-unbound` do `~/.ssh/config`

---

<a id="8"></a>
## 8) Boas práticas de produção

- **Cada pessoa, uma chave**: não compartilhe chave privada entre operadores.
- **Revogação rápida**: quando alguém sai, remova a chave pública dela dos servidores imediatamente.
- **Restrinja rede**: libere a porta SSH no firewall apenas para redes de administração/VPN/bastion.
- **Auditoria**: monitore logs de SSH `/var/log/auth.log` em Debian/Ubuntu; `journalctl -u ssh` em alguns setups.
- **Identidades previsíveis**: use `IdentityFile` + `IdentitiesOnly` para evitar “too many authentication failures”.
- **Usuário comum + sudo**: deixe `PermitRootLogin no` e use `sudo` para tarefas administrativas.

---

<a id="9"></a>
## 9) O que NÃO fazer (erros comuns)

- Reutilizar a chave do GitHub/GitLab para acessar servidores de produção.
- Desativar senha antes de validar o login por chave em uma nova sessão.
- Usar permissões erradas:
  - `~/.ssh` diferente de `700`
  - chave privada diferente de `600`
  - `authorized_keys` diferente de `600`
- Criar alias genérico tipo `alias servidor=...` vira armadilha operacional com múltiplos ambientes.
- Logar diretamente como `root`, use usuário comum + sudo.

---

### Troubleshooting rápido (quando algo der errado)

Cliente (use só quando necessário):

```bash
ssh -vvv servidor-unbound
```

Servidor (Debian/Ubuntu):

```bash
sudo journalctl -u ssh -n 100 --no-pager
```

Validar configuração efetiva (o que o `sshd` está enxergando):

```bash
sudo sshd -T | egrep '^(port|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|challengeresponseauthentication|permitrootlogin)\b'
```

<a id="10"></a>
## 10) Checklist rápido (validação)

No cliente:

```bash
ssh servidor-unbound
```

No servidor:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Confirme que a senha está desativada:

```bash
sudo sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication'
```

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
