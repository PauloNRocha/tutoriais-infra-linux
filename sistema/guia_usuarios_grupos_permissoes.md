# Guia Prático: Usuários, Grupos e Permissões no Linux

*Criado em: 22 de dezembro de 2025*  
*Última atualização em: 19 de maio de 2026*

Usuário, grupo e permissão parecem assunto básico, mas é justamente aí que muita manutenção de servidor começa a dar problema: acesso demais, acesso de menos, usuário de serviço com shell, pasta compartilhada com grupo errado ou `sudo` liberado além do necessário.

Aqui a ideia é deixar um material de consulta para Linux em geral, com foco prático em Debian e Ubuntu. O guia cobre criação e manutenção de usuários, grupos, permissões tradicionais, permissões especiais, ACL, `umask` e alguns cuidados para uso em servidor.

---

## Índice rápido

1. [Conceitos básicos](#1)
2. [Tipos de usuários](#2)
3. [Criação de usuários](#3)
4. [Modificação de usuários](#4)
5. [Remoção de usuários](#5)
6. [Gerenciamento de grupos](#6)
7. [Sudo e privilégios administrativos](#7)
8. [Permissões de arquivos e diretórios](#8)
9. [Permissões especiais (SUID, SGID, Sticky Bit)](#9)
10. [ACL (Access Control List)](#10)
11. [Umask](#11)
12. [Cenários práticos](#12)
13. [Troubleshooting comum](#13)
14. [Referências](#referencias)

---

<a id="1"></a>
## 1. Conceitos básicos

- Todo processo no sistema é executado por um **usuário**.
- Todo arquivo pertence a um **usuário** e a um **grupo**.
- As **permissões** determinam quem pode **ler (read)**, **escrever (write)** ou **executar (execute)** um arquivo ou diretório.

#### UID e GID

- **UID (User IDentifier):** número de identificação único de um usuário.
- **GID (Group IDentifier):** número de identificação único de um grupo.

O sistema usa esses números, não os nomes, para controlar o acesso.

#### Arquivos importantes

- `/etc/passwd`: contém informações básicas dos usuários, como nome, UID, GID, diretório home e shell.
- `/etc/shadow`: armazena hashes de senha e informações de expiração. Apenas o `root` deve conseguir ler este arquivo.
- `/etc/group`: lista os grupos locais e seus membros.
- `/etc/sudoers` e `/etc/sudoers.d/`: controlam permissões administrativas via `sudo`.

---

<a id="2"></a>
## 2. Tipos de usuários

### Usuário `root`

- **UID 0:** é o superusuário.
- Possui controle total sobre o sistema.
- Deve ser usado apenas para tarefas que exigem privilégio máximo. Para rotina administrativa, prefira um usuário normal com `sudo`.

### Usuários normais

- **UID >= 1000:** normalmente usados para login interativo por pessoas.
- Por padrão, têm acesso limitado, mas podem executar tarefas administrativas quando autorizados pelo `sudo`.

### Usuários de serviço

- **UID < 1000, geralmente:** contas criadas para rodar aplicações e serviços, como `www-data`, `postgres` ou `backup`.
- Em servidor, normalmente devem usar shell sem login, como `/usr/sbin/nologin`, para reduzir risco de acesso interativo indevido.

---

<a id="3"></a>
## 3. Criação de usuários

### `adduser` para usuário interativo

Em Debian e Ubuntu, eu prefiro `adduser` para criar usuário humano, porque ele segue as políticas da distribuição e já cria home, grupo primário e configuração inicial.

```bash
sudo adduser joao
```

Esse comando normalmente:

- cria o usuário `joao`;
- cria o grupo primário `joao`;
- cria o diretório `/home/joao`;
- copia arquivos de `/etc/skel` para a nova home;
- pede senha e informações opcionais.

### `useradd` para automação e usuários de serviço

O `useradd` é mais direto e costuma fazer mais sentido em scripts ou quando você quer controlar exatamente o que será criado.

Exemplo de usuário de serviço sem login interativo:

```bash
sudo useradd -r -s /usr/sbin/nologin appuser
```

O que importa nesse exemplo:

- `-r`: cria um usuário de sistema;
- `-s /usr/sbin/nologin`: impede login interativo pelo shell.

Depois confira:

```bash
id appuser
getent passwd appuser
```

---

<a id="4"></a>
## 4. Modificação de usuários

O comando principal para modificar um usuário existente é o `usermod`.

### Alterar o shell padrão

```bash
sudo usermod -s /bin/bash joao
```

Para usuário de serviço, normalmente é melhor usar shell sem login:

```bash
sudo usermod -s /usr/sbin/nologin appuser
```

### Adicionar um usuário a um grupo suplementar

```bash
sudo usermod -aG sudo joao
```

O `-a` é importante. Sem ele, o `usermod -G` substitui a lista de grupos suplementares e pode remover o usuário de grupos que ele já tinha.

### Bloquear senha e desativar conta

Para bloquear apenas a senha:

```bash
sudo usermod -L joao
```

Isso não é a mesma coisa que bloquear todos os meios de acesso. Se o usuário tiver chave SSH, dependendo da configuração do ambiente, ainda pode haver acesso. Para desativar a conta de forma mais completa, expire a conta também:

```bash
sudo usermod -L joao
sudo usermod --expiredate 1 joao
```

Valide o estado da senha e da expiração:

```bash
sudo passwd -S joao
sudo chage -l joao
```

Para reverter:

```bash
sudo usermod -U joao
sudo usermod --expiredate -1 joao
```

Depois valide novamente:

```bash
sudo passwd -S joao
sudo chage -l joao
```

### Forçar troca de senha no próximo login

```bash
sudo chage -d 0 joao
```

### Ver informações do usuário

```bash
id joao
getent passwd joao
```

---

<a id="5"></a>
## 5. Remoção de usuários

Em Debian e Ubuntu, prefira `deluser` para remoção manual de usuários. O `userdel` existe, mas é uma ferramenta mais baixa; a própria documentação do Debian recomenda `deluser` para administradores.

Antes de remover, confira processos e arquivos do usuário:

```bash
id joao
ps -u joao
sudo find / -xdev -user joao -ls
```

Em servidor de produção, eu costumo primeiro bloquear e observar, em vez de remover direto:

```bash
sudo usermod -L joao
sudo usermod --expiredate 1 joao
```

### Remover apenas o usuário

Remove a conta, mas mantém home e arquivos:

```bash
sudo deluser joao
```

### Remover usuário com backup da home

Quando a intenção for remover a home também, faça backup antes:

```bash
sudo mkdir -p /root/backups-usuarios
sudo deluser --backup --backup-to /root/backups-usuarios --remove-home joao
```

Em sistemas que não possuem `deluser`, o caminho equivalente costuma ser `userdel -r joao`, mas sem o mesmo fluxo de backup do `deluser`.

> [!CAUTION]
> Remoção de usuário pode afetar serviços, scripts, chaves SSH, crons e arquivos espalhados pelo sistema. Não rode em produção sem validar impacto.

---

<a id="6"></a>
## 6. Gerenciamento de grupos

### Criar um grupo

```bash
sudo groupadd suporte
```

### Adicionar usuário a um grupo

```bash
sudo usermod -aG suporte joao
```

O usuário precisa fazer logout e login novamente para a nova associação de grupo aparecer na sessão dele. Para abrir uma nova sessão de shell já com o grupo, use:

```bash
newgrp suporte
```

### Ver grupos de um usuário

```bash
id joao
groups joao
```

### Remover usuário de um grupo

```bash
sudo gpasswd -d joao suporte
```

---

<a id="7"></a>
## 7. Sudo e privilégios administrativos

O `sudo` permite que um usuário execute comandos com privilégio administrativo, normalmente como `root`.

### Adicionar usuário ao grupo `sudo`

No Debian e Ubuntu, o grupo que concede privilégio administrativo amplo costuma ser `sudo`:

```bash
sudo usermod -aG sudo joao
```

Depois o usuário precisa sair e entrar novamente. Valide com:

```bash
id joao
sudo -l -U joao
```

### Editar permissões específicas com `visudo`

Evite editar `/etc/sudoers` diretamente. Para regras locais, prefira um arquivo em `/etc/sudoers.d/` usando `visudo`, porque ele valida a sintaxe antes de salvar.

Exemplo: permitir que `joao` reinicie o Nginx sem senha.

```bash
sudo visudo -f /etc/sudoers.d/joao-nginx
```

Conteúdo:

```text
joao ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx
```

Valide a sintaxe:

```bash
sudo visudo -c
```

> [!CAUTION]
> Evite liberar `ALL=(ALL) ALL` ou `NOPASSWD: ALL` sem necessidade real. Em servidor, regra de `sudo` deve seguir o menor privilégio possível.

---

<a id="8"></a>
## 8. Permissões de arquivos e diretórios

### Visualizando permissões

```bash
ls -l /caminho/do/arquivo
```

Exemplo de saída:

```text
-rwxr-x---
```

O primeiro caractere indica o tipo: `-` para arquivo, `d` para diretório, `l` para link simbólico. Os próximos 9 caracteres são divididos em 3 blocos:

- dono do arquivo;
- grupo do arquivo;
- outros usuários.

Permissões:

- `r`: leitura;
- `w`: escrita;
- `x`: execução.

### `chmod` (change mode)

Altera as permissões.

Modo numérico, também chamado de octal:

- `4`: leitura;
- `2`: escrita;
- `1`: execução.

Exemplo:

```bash
chmod 750 arquivo.sh
```

Nesse caso:

- dono: `7`, leitura, escrita e execução;
- grupo: `5`, leitura e execução;
- outros: `0`, sem acesso.

Modo simbólico:

```bash
chmod u+x arquivo.sh
chmod g-w arquivo.sh
chmod o= arquivo.sh
```

Onde:

- `u`: dono;
- `g`: grupo;
- `o`: outros;
- `a`: todos.

### `chown` (change owner)

Altera o dono e/ou o grupo de um arquivo ou diretório.

```bash
# Altera o dono para joao e o grupo para suporte
sudo chown joao:suporte arquivo.sh

# Altera apenas o dono
sudo chown maria arquivo.sh

# Altera recursivamente para um diretório
sudo chown -R joao:suporte /dados/
```

> [!CAUTION]
> `chown -R` em caminho errado causa estrago rápido. Antes de usar em produção, confirme o diretório com `pwd`, `ls -ld /dados/` e, se possível, teste em um caminho pequeno.

---

<a id="9"></a>
## 9. Permissões especiais

### Sticky Bit (`t`)

Aplicado a diretórios, impede que um usuário apague arquivos de outro usuário, mesmo que todos tenham permissão de escrita no diretório. Cada usuário só pode apagar os arquivos dos quais é dono. É o caso clássico do `/tmp`.

```bash
ls -ld /tmp
sudo chmod +t /srv/compartilhado
```

### SGID (Set Group ID) (`s` no grupo)

Em arquivos executáveis, o processo roda com o GID do arquivo. Em diretórios, que é o uso mais comum em servidor, novos arquivos e subdiretórios herdam o grupo do diretório pai. Isso ajuda bastante em pastas de equipe.

```bash
sudo chmod g+s /srv/compartilhado
```

### SUID (Set User ID) (`s` no dono)

Aplicado a arquivos executáveis, permite que o programa rode com as permissões do dono do arquivo, frequentemente `root`, e não com as permissões do usuário que executou o programa. O `passwd` usa SUID para permitir que um usuário normal altere sua própria senha.

Exemplo conceitual:

```bash
chmod u+s /usr/local/bin/meu_programa
```

> [!CAUTION]
> SUID é sensível. Um bug em binário SUID pode virar escalada de privilégio. Em produção, não aplique SUID em programa próprio sem auditoria.

---

<a id="10"></a>
## 10. ACL (Access Control List)

ACLs são usadas quando o modelo tradicional `dono/grupo/outros` não é suficiente. Elas permitem dar permissões a usuários e grupos específicos sem trocar o dono principal do arquivo.

Instale as ferramentas, se necessário:

```bash
sudo apt install -y acl
```

Ver ACL de um arquivo:

```bash
getfacl arquivo.txt
```

Se um arquivo tem ACL, o `ls -l` mostra um `+` no final das permissões, como `-rw-r--r--+`.

Dar permissão de leitura e escrita para `maria`:

```bash
setfacl -m u:maria:rw arquivo.txt
```

Dar permissão de leitura para o grupo `marketing`:

```bash
setfacl -m g:marketing:r arquivo.txt
```

Remover todas as ACLs de um arquivo:

```bash
setfacl -b arquivo.txt
```

---

<a id="11"></a>
## 11. Umask

O `umask` define as permissões padrão para novos arquivos e diretórios. Ele funciona como uma máscara que remove permissões da base.

Bases comuns:

- diretórios partem de `777`;
- arquivos partem de `666`, porque arquivo comum não nasce executável por padrão.

Exemplo com `umask 0022`:

- diretórios: `777 - 022 = 755`, ou `drwxr-xr-x`;
- arquivos: `666 - 022 = 644`, ou `-rw-r--r--`.

Para ver o `umask` atual:

```bash
umask
```

Para alterar temporariamente na sessão atual:

```bash
umask 0027
```

Em servidor, `0027` costuma ser uma escolha mais restritiva para contas que não devem expor arquivos para outros usuários locais.

---

<a id="12"></a>
## 12. Cenários práticos

### Diretório compartilhado para uma equipe

O objetivo é que todos do grupo `suporte` possam criar e editar arquivos entre si.

```bash
sudo groupadd suporte
sudo mkdir -p /srv/compartilhado
sudo chown root:suporte /srv/compartilhado
sudo chmod 2770 /srv/compartilhado
```

O `2` no início de `2770` ativa SGID no diretório. Com isso, novos arquivos e subdiretórios tendem a herdar o grupo `suporte`. Para garantir melhor as permissões padrão de escrita do grupo, use ACL padrão junto com o SGID.

Aplique ACL no diretório atual e ACL padrão para novos itens:

```bash
sudo setfacl -m g:suporte:rwx /srv/compartilhado
sudo setfacl -m d:g:suporte:rwx /srv/compartilhado
```

### Usuário de aplicação sem acesso ao shell

```bash
sudo useradd -r -s /usr/sbin/nologin appuser
sudo chown -R appuser:appuser /opt/minha_app
```

Depois confira:

```bash
id appuser
getent passwd appuser
ls -ld /opt/minha_app
```

### Usuário dedicado para receber backup em um diretório

Um caso comum é criar, no servidor de destino, uma conta usada por outro sistema apenas para enviar backups. A ideia é separar esse acesso do usuário administrativo e limitar a escrita ao diretório de backup.

Exemplo para backups do servidor `servidor1`:

```bash
sudo groupadd backup-remoto
sudo mkdir -p /srv/backups/servidor1
sudo useradd -r -d /srv/backups/servidor1 -s /usr/sbin/nologin -g backup-remoto backup_servidor1
sudo chown backup_servidor1:backup-remoto /srv/backups/servidor1
sudo chmod 0750 /srv/backups/servidor1
```

Valide:

```bash
id backup_servidor1
getent passwd backup_servidor1
ls -ld /srv/backups/servidor1
```

Esse exemplo amarra a permissão pelo dono, grupo e modo do diretório. Ele não substitui uma política de SSH, chave restrita, `chroot` ou comando forçado. Se o backup for enviado por `rsync` via SSH, valide o método de acesso com cuidado, porque `rsync` normalmente precisa executar comando no destino.

---

<a id="13"></a>
## 13. Troubleshooting comum

### Permissão negada mesmo com `chmod`

Confira dono, grupo e permissões:

```bash
ls -ld /caminho
ls -l /caminho/arquivo
```

Confira ACL:

```bash
getfacl /caminho/arquivo
```

Confira se o sistema de arquivos está somente leitura:

```bash
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /caminho
```

Também pode haver bloqueio por AppArmor ou SELinux, dependendo da distribuição e do serviço envolvido.

### Usuário não consegue `sudo`

```bash
id nome_usuario
sudo -l -U nome_usuario
sudo visudo -c
```

Se o usuário acabou de entrar no grupo `sudo`, ele precisa sair e entrar novamente.

### Novo grupo não aparece na sessão atual

```bash
id
newgrp nome_do_grupo
id
```

O `newgrp` abre um novo shell com o grupo aplicado. Para valer em uma sessão gráfica ou SSH normal, faça logout e login novamente.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### Manpages e documentação Debian

- `adduser(8)`: https://manpages.debian.org/bookworm/adduser/adduser.8.en.html
- `useradd(8)`: https://manpages.debian.org/bookworm/passwd/useradd.8.en.html
- `usermod(8)`: https://manpages.debian.org/bookworm/passwd/usermod.8.en.html
- `deluser(8)`: https://manpages.debian.org/bookworm/adduser/deluser.8.en.html
- `userdel(8)`: https://manpages.debian.org/bookworm/passwd/userdel.8.en.html
- `groupadd(8)`: https://manpages.debian.org/bookworm/passwd/groupadd.8.en.html
- `chmod(1)`: https://manpages.debian.org/bookworm/coreutils/chmod.1.en.html
- `chown(1)`: https://manpages.debian.org/bookworm/coreutils/chown.1.en.html
- `sudoers(5)`: https://manpages.debian.org/bookworm/sudo/sudoers.5.en.html
- `visudo(8)`: https://manpages.debian.org/bookworm/sudo/visudo.8.en.html
- `getfacl(1)`: https://manpages.debian.org/bookworm/acl/getfacl.1.en.html
- `setfacl(1)`: https://manpages.debian.org/bookworm/acl/setfacl.1.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
