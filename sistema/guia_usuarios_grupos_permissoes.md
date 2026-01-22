# Guia Completo de Usuários, Grupos e Permissões no Linux

*Criado em 22 de dezembro de 2025*

No Linux, tudo gira em torno de **usuários**, **grupos** e **permissões**. Entender como gerenciá-los é a base para ter um sistema seguro, organizado e funcional.

Este guia foi criado como uma documentação prática e consultiva. O objetivo não é apenas listar comandos, mas **explicar o que está sendo feito, por quê e quando usar** cada recurso, servindo tanto para estudo quanto para uso real em servidores.

---

## Índice rápido

1.  [Conceitos básicos](#1)
2.  [Tipos de usuários](#2)
3.  [Criação de usuários](#3)
4.  [Modificação de usuários](#4)
5.  [Remoção de usuários](#5)
6.  [Gerenciamento de Grupos](#6)
7.  [Sudo e privilégios administrativos](#7)
8.  [Permissões de arquivos e diretórios](#8)
9.  [Permissões especiais (SUID, SGID, Sticky Bit)](#9)
10. [ACL (Access Control List)](#10)
11. [Umask](#11)
12. [Cenários práticos](#12)
13. [Troubleshooting comum](#13)

---

<a id="1"></a>
## 1. Conceitos básicos

-   Todo processo no sistema é executado por um **usuário**.
-   Todo arquivo pertence a um **usuário** e a um **grupo**.
-   As **permissões** determinam quem pode **ler (read)**, **escrever (write)** ou **executar (execute)** um arquivo ou diretório.

#### UID e GID
-   **UID (User IDentifier):** O número de identificação único de um usuário.
-   **GID (Group IDentifier):** O número de identificação único de um grupo.

O sistema usa esses números, não os nomes, para controlar o acesso.

#### Arquivos importantes
-   `/etc/passwd`: Contém informações básicas dos usuários (nome, UID, GID, diretório home, shell).
-   `/etc/shadow`: Armazena as senhas criptografadas dos usuários. Apenas o `root` pode ler este arquivo.
-   `/etc/group`: Lista todos os grupos e quem pertence a eles.

---

<a id="2"></a>
## 2. Tipos de usuários

### Usuário `root`
-   **UID 0:** O superusuário.
-   Possui controle total e irrestrito sobre o sistema.
-   Deve ser usado apenas para tarefas que exigem privilégios máximos. Para o dia a dia, prefira um usuário normal com `sudo`.

### Usuários normais
-   **UID ≥ 1000:** Usados para login interativo por humanos.
-   Por padrão, têm acesso limitado, mas podem executar tarefas administrativas usando o comando `sudo`.

### Usuários de serviço
-   **UID < 1000 (geralmente):** Contas especiais criadas para rodar aplicações específicas (ex: `www-data` para o Apache, `postgres` para o PostgreSQL).
-   Frequentemente não possuem um shell de login válido para impedir acesso interativo, aumentando a segurança.

---

<a id="3"></a>
## 3. Criação de usuários

### `adduser` (Método recomendado e interativo)
O `adduser` é um script amigável que automatiza várias tarefas.

```bash
sudo adduser joao
```
Este comando irá:
-   Criar o usuário `joao`.
-   Criar um grupo primário `joao`.
-   Criar o diretório home `/home/joao`.
-   Copiar arquivos de esqueleto (como `.bashrc`) de `/etc/skel` para a nova home.
-   Pedir para você definir uma senha e outras informações (opcionais).

### `useradd` (Método de baixo nível, para scripts)
O `useradd` é mais direto e menos interativo, ideal para automação.

```bash
# Criar usuário de serviço sem login interativo
sudo useradd -r -s /usr/sbin/nologin appuser
```
-   `-r`: Cria um usuário de sistema.
-   `-s /usr/sbin/nologin`: Define o shell como `nologin`, impedindo o login.

---

<a id="4"></a>
## 4. Modificação de usuários

O comando principal para modificar um usuário existente é o `usermod`.

-   **Alterar o shell padrão:**
    ```bash
    sudo usermod -s /bin/bash joao
    ```
-   **Adicionar um usuário a um grupo suplementar:**
    ```bash
    # A flag -a (append) é MUITO importante para não remover os outros grupos
    sudo usermod -aG sudo joao
    ```
-   **Bloquear a conta de um usuário (impede login):**
    ```bash
    sudo usermod -L joao
    ```
-   **Desbloquear a conta:**
    ```bash
    sudo usermod -U joao
    ```
-   **Forçar troca de senha no próximo login:**
    ```bash
    sudo chage -d 0 joao
    ```
-   **Ver informações do usuário (UID, GID, grupos):**
    ```bash
    id joao
    ```

---

<a id="5"></a>
## 5. Remoção de usuários

### Remover apenas o usuário
Este comando remove o usuário dos arquivos de sistema, mas **mantém seu diretório home**.

```bash
sudo userdel joao
```

### Remover o usuário e seu diretório home
A flag `-r` remove também o diretório home e o mail spool.

```bash
sudo userdel -r joao
```
> **Atenção:** Em servidores, antes de remover um usuário, verifique se ele não é dono de arquivos ou processos importantes (`find / -user joao`, `ps -u joao`).

---

<a id="6"></a>
## 6. Gerenciamento de Grupos

-   **Criar um novo grupo:**
    ```bash
    sudo groupadd suporte
    ```
-   **Adicionar um usuário a um grupo:**
    (Já visto na seção `usermod`)
    ```bash
    sudo usermod -aG suporte joao
    ```
    > **Importante:** O usuário precisa fazer **logout e login novamente** para que a nova associação de grupo tenha efeito em sua sessão. Para aplicar em uma sessão ativa, use `newgrp grupo`.

-   **Ver os grupos de um usuário:**
    ```bash
    groups joao
    ```
-   **Remover um usuário de um grupo:**
    ```bash
    sudo gpasswd -d joao suporte
    ```

---

<a id="7"></a>
## 7. Sudo e privilégios administrativos

O `sudo` (superuser do) permite que um usuário normal execute comandos como se fosse o `root`.

-   **Adicionar um usuário ao grupo `sudo` (ou `wheel` em sistemas RHEL/CentOS):**
    No Debian/Ubuntu, o grupo que concede privilégios de `sudo` é, por padrão, o `sudo`.
    ```bash
    sudo usermod -aG sudo joao
    ```
-   **Editar o arquivo `sudoers` (SEMPRE use `visudo`):**
    O `visudo` trava o arquivo e verifica a sintaxe antes de salvar, evitando que você quebre o acesso administrativo ao sistema.
    ```bash
    sudo visudo
    ```
-   **Exemplo `sudoers`: permitir que `joao` reinicie o Nginx sem senha:**
    Adicione esta linha ao final do arquivo:
    ```text
    joao ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx
    ```
    > **Princípio do Menor Privilégio:** Evite liberar `sudo` completo (`ALL=(ALL) ALL`) sem necessidade. Conceda permissões específicas para os comandos que o usuário realmente precisa.

---

<a id="8"></a>
## 8. Permissões de arquivos e diretórios

### Visualizando permissões
```bash
ls -l /caminho/do/arquivo
```
Exemplo de saída: `-rwxr-x---`
-   O primeiro caractere indica o tipo (`-` para arquivo, `d` para diretório).
-   Os próximos 9 caracteres são divididos em 3 blocos: **dono (user)**, **grupo (group)** e **outros (others)**.
    -   `r` = read (leitura)
    -   `w` = write (escrita)
    -   `x` = execute (execução)

### `chmod` (change mode)
Altera as permissões.

-   **Modo Numérico (Octal):**
    -   `4` (read) + `2` (write) + `1` (execute) = `7` (rwx)
    -   `4` (read) + `1` (execute) = `5` (r-x)
    -   `4` (read) + `2` (write) = `6` (rw-)
    ```bash
    # Dono pode ler/escrever/executar, grupo pode ler/executar, outros não podem fazer nada.
    chmod 750 arquivo.sh
    ```
-   **Modo Simbólico:**
    -   `u` (user), `g` (group), `o` (others), `a` (all)
    -   `+` (adicionar), `-` (remover), `=` (definir)
    ```bash
    # Adicionar permissão de execução para o dono (user)
    chmod u+x arquivo.sh
    ```

### `chown` (change owner)
Altera o dono e/ou o grupo de um arquivo/diretório.
```bash
# Altera o dono para 'joao' e o grupo para 'suporte'
sudo chown joao:suporte arquivo.sh

# Altera apenas o dono
sudo chown maria arquivo.sh

# Altera recursivamente para um diretório
sudo chown -R joao:suporte /dados/
```

---

<a id="9"></a>
## 9. Permissões especiais

### Sticky Bit (`t`)
Aplicado a diretórios, impede que um usuário apague arquivos de outro usuário, mesmo que todos tenham permissão de escrita no diretório. Cada usuário só pode apagar os arquivos dos quais é dono. Essencial em diretórios compartilhados como `/tmp`.
```bash
ls -ld /tmp
# drwxrwxrwt 13 root root 12288 dez 18 10:00 /tmp
chmod +t /diretorio/compartilhado
```

### SGID (Set Group ID) (`s` no grupo)
-   **Em arquivos executáveis:** O processo roda com o GID do arquivo, não do usuário.
-   **Em diretórios (muito útil):** Qualquer novo arquivo ou subdiretório criado dentro dele **herda o grupo do diretório pai**, em vez do grupo primário do usuário. Perfeito para pastas de projetos em equipe.
    ```bash
    chmod g+s /dados/projeto_x
    ```

### SUID (Set User ID) (`s` no dono)
Aplicado a arquivos executáveis, permite que um usuário rode o programa com as permissões do **dono do arquivo** (geralmente `root`), não do usuário que o executou. O comando `passwd` usa SUID para permitir que um usuário normal altere seu próprio arquivo de senhas (`/etc/shadow`), que é propriedade do `root`.
```bash
chmod u+s /usr/bin/meu_programa
```
> **Risco de Segurança:** Use SUID com extremo cuidado. Um bug em um programa com SUID pode levar a uma escalada de privilégios.

---

<a id="10"></a>
## 10. ACL (Access Control List)

ACLs são usadas quando o modelo tradicional `dono/grupo/outros` não é suficiente. Elas permitem dar permissões a múltiplos usuários e grupos específicos.

-   **Ver ACL de um arquivo:**
    ```bash
    getfacl arquivo.txt
    ```
    Se um arquivo tem ACL, o `ls -l` mostra um `+` no final das permissões (`-rw-r--r--+`).
-   **Dar permissão de leitura/escrita para `maria`:**
    ```bash
    setfacl -m u:maria:rw arquivo.txt
    ```
-   **Dar permissão de leitura para o grupo `marketing`:**
    ```bash
    setfacl -m g:marketing:r arquivo.txt
    ```
-   **Remover todas as ACLs de um arquivo:**
    ```bash
    setfacl -b arquivo.txt
    ```

---

<a id="11"></a>
## 11. Umask

O `umask` define as **permissões padrão** para novos arquivos e diretórios. Ele funciona como uma "máscara" que remove permissões.
-   Permissão base para diretórios: `777`
-   Permissão base para arquivos: `666`

O `umask` padrão comum é `0022` ou `0002`.
-   `umask 0022`:
    -   Diretórios: `777 - 022 = 755` (`drwxr-xr-x`)
    -   Arquivos: `666 - 022 = 644` (`-rw-r--r--`)

Para ver o umask atual:
```bash
umask
```
Para alterar temporariamente na sessão atual:
```bash
umask 0027
```

---

<a id="12"></a>
## 12. Cenários práticos

### Diretório compartilhado para uma equipe
O objetivo é que todos do grupo `suporte` possam criar e editar arquivos entre si.

```bash
# 1. Crie o diretório e defina o grupo
sudo mkdir /srv/compartilhado
sudo chown root:suporte /srv/compartilhado

# 2. Defina permissões e o SGID para herança de grupo
sudo chmod 2770 /srv/compartilhado
```
O `2` no início (`chmod 2770`) é a forma octal de definir o SGID.

### Usuário de aplicação sem acesso ao shell
```bash
# 1. Crie o usuário de serviço
sudo useradd -r -s /usr/sbin/nologin appuser

# 2. Dê a ele posse dos arquivos da aplicação
sudo chown -R appuser:appuser /opt/minha_app
```

---

<a id="13"></a>
## 13. Troubleshooting comum

-   **"Permissão negada" mesmo com `chmod`:**
    -   Verifique o dono/grupo (`ls -l`).
    -   Verifique se há uma ACL ativa (`getfacl`).
    -   Verifique se o sistema de arquivos não está montado como somente leitura (`mount | grep ' ro '`).
    -   Pode ser um sistema de segurança como SELinux ou AppArmor bloqueando o acesso.

-   **Usuário não consegue `sudo`:**
    -   Confirme se ele está no grupo `sudo` (`id nome_usuario`).
    -   Verifique se não há erros de sintaxe no `/etc/sudoers` com `sudo visudo -c`.
    -   O usuário precisa ter feito logout/login.

-   **Novo grupo não funciona na sessão atual:**
    -   O usuário precisa sair e entrar novamente.
    -   Para usar o grupo imediatamente, execute `newgrp nome_do_grupo`. Isso iniciará um novo shell com o GID correto.

---

## Referências (fontes para consulta)

### Manpages (Debian)

- `adduser(8)` / `useradd(8)`: https://manpages.debian.org/bookworm/adduser/adduser.8.en.html • https://manpages.debian.org/bookworm/passwd/useradd.8.en.html
- `groupadd(8)`: https://manpages.debian.org/bookworm/passwd/groupadd.8.en.html
- `chmod(1)` / `chown(1)`: https://manpages.debian.org/bookworm/coreutils/chmod.1.en.html • https://manpages.debian.org/bookworm/coreutils/chown.1.en.html
- `sudoers(5)` / `visudo(8)`: https://manpages.debian.org/bookworm/sudo/sudoers.5.en.html • https://manpages.debian.org/bookworm/sudo/visudo.8.en.html
- `getfacl(1)` / `setfacl(1)`: https://manpages.debian.org/bookworm/acl/getfacl.1.en.html • https://manpages.debian.org/bookworm/acl/setfacl.1.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
