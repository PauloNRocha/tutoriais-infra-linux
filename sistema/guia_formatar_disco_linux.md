# Linux: formatar discos e partições (guia seguro)

*Criado em 04 de dezembro de 2025 e atualizado em 08 de dezembro de 2025*

Formatar um disco no Linux é uma tarefa poderosa, mas que exige **muita atenção**. Um erro pode apagar dados importantes para sempre. Este guia detalha um fluxo seguro, passo a passo, para identificar, particionar e formatar um disco.

---

> [!WARNING]
> **AVISO IMPORTANTE: PERDA DE DADOS**
>  
> Formatar um disco ou partição **apaga permanentemente** todos os dados contidos nele. Verifique **três vezes** o nome do dispositivo (ex.: `/dev/sdX`) antes de executar qualquer comando.

---

## Índice rápido
1. [Passo 1: Identificar o Disco Corretamente](#1)
2. [Passo 2: (Opcional) Criar uma Nova Tabela de Partições](#2)
3. [Passo 3: Criar uma Partição](#3)
4. [Passo 4: Formatar a Nova Partição](#4)
5. [Passo 5: Montar a Partição (Permanente via fstab)](#5)
6. [Solução de Problemas: Erros Comuns no fstab](#6)

---

<a id="1"></a>
## 1. Passo 1: Identificar o Disco Corretamente

Este é o passo mais crítico. Conecte o disco (HD externo, SSD, pendrive) e vamos usar dois comandos para ter certeza absoluta de qual dispositivo vamos formatar.

```bash
sudo fdisk -l
```
Este comando lista todos os discos e suas tabelas de partição. Procure pelo seu disco baseado no **tamanho** (`Disk model` e `Disk size`).

Depois, use `lsblk` para uma visão mais clara:
```bash
lsblk
```
A saída será algo como:
```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 238.5G  0 disk 
├─sda1   8:1    0   512M  0 part /boot/efi
└─sda2   8:2    0   238G  0 part /
sdb      8:16   0 931.5G  0 disk 
└─sdb1   8:17   0 931.5G  0 part /media/usuario/Backup
```
No exemplo acima, `sda` é o disco do sistema (note os `MOUNTPOINT` `/` e `/boot/efi`). `sdb` é um disco externo de `931.5G` montado em `/media/usuario/Backup`. **Nunca execute esses comandos em um disco que tenha partições do sistema montadas.**

Uma vez que você identificou o dispositivo (ex: `/dev/sdb`), desmonte qualquer partição que esteja em uso:
```bash
# Use o nome da partição, ex: /dev/sdb1
sudo umount /dev/sdb1
```

---

<a id="2"></a>
## 2. Passo 2: (Opcional) Criar uma Nova Tabela de Partições

Se o disco é novo ou se você quer apagar absolutamente tudo (incluindo a estrutura de partições antiga), você deve criar uma nova tabela de partições. Para discos modernos (> 2TB) ou para compatibilidade com UEFI, o padrão é **GPT**.

> **Atenção:** O próximo comando apaga a estrutura do disco `/dev/sdb`. Tenha certeza de que é o disco certo!

```bash
# Substitua /dev/sdb pelo seu disco
sudo parted /dev/sdb mklabel gpt
```
Este comando cria uma nova tabela de partições GPT vazia no disco.

---

<a id="3"></a>
## 3. Passo 3: Criar uma Partição

Com o disco "limpo", vamos criar uma partição para ocupar 100% do espaço. Usaremos o `fdisk`, uma ferramenta de linha de comando poderosa.

```bash
# Substitua /dev/sdb pelo seu disco
sudo fdisk /dev/sdb
```
O `fdisk` vai abrir um prompt interativo. Siga esta sequência de comandos:

1.  Digite `g` para garantir que está criando uma tabela de partição GPT vazia (se não fez o passo 2).
2.  Digite `n` para criar uma nova partição.
3.  Pressione `Enter` para aceitar o número padrão da partição (1).
4.  Pressione `Enter` para aceitar o primeiro setor padrão.
5.  Pressione `Enter` novamente para aceitar o último setor padrão (usando o disco todo).
6.  O `fdisk` pode perguntar se deseja remover uma assinatura existente. Digite `Y` se perguntar.
7.  Finalmente, digite `w` para escrever as alterações no disco e sair.

Ao final, você terá uma nova partição, como `/dev/sdb1`.

---

<a id="4"></a>
## 4. Passo 4: Formatar a Nova Partição

Agora que temos a partição `/dev/sdb1`, podemos formatá-la com o sistema de arquivos que quisermos.

### Para uso exclusivo no Linux (Recomendado: **ext4**)
```bash
# Substitua /dev/sdb1 pela sua partição
sudo mkfs.ext4 /dev/sdb1
```

### Para compatibilidade com Windows e outros (Recomendado: **exFAT**)
O exFAT não tem as limitações de 4GB por arquivo do antigo FAT32.
```bash
# Primeiro, instale a ferramenta se não tiver
sudo apt update && sudo apt install exfatprogs -y

# Formate a partição
sudo mkfs.exfat /dev/sdb1
```

### Para compatibilidade com Windows (Formato nativo: **NTFS**)
```bash
# Primeiro, instale a ferramenta se não tiver
sudo apt update && sudo apt install ntfs-3g -y

# Formate a partição
sudo mkfs.ntfs -f /dev/sdb1
```
A opção `-f` realiza uma formatação rápida.

### Verificação Final
Para ter certeza de que tudo funcionou, liste os dispositivos novamente:
```bash
lsblk -f
```
Você deverá ver a sua nova partição com o `FSTYPE` (sistema de arquivos) que você escolheu. Agora o disco está pronto para ser montado.

---

<a id="5"></a>
## 5. Passo 5: Montar a Partição (Permanente via fstab)

Para que o disco seja montado automaticamente a cada inicialização, precisamos adicioná-lo ao arquivo `/etc/fstab`. É altamente recomendado usar o `UUID` da partição, pois ele é único e não muda se a ordem dos discos mudar (diferente de `/dev/sdb1`).

1.  **Descobrir o UUID da nova partição:**
    ```bash
    sudo blkid
    ```
    Anote o `UUID` da sua partição (ex: `1234abcd-5678-efab-cdef-1234567890ab`).

2.  **Crie um ponto de montagem:**
    ```bash
    sudo mkdir /mnt/meudisco
    ```

3.  **Edite o `/etc/fstab`:**
    ```bash
    sudo nano /etc/fstab
    ```
    Adicione a seguinte linha no final do arquivo, substituindo o `UUID` e o tipo de sistema de arquivos conforme o seu caso:
    ```
    UUID=1234abcd-5678-efab-cdef-1234567890ab /mnt/meudisco ext4 defaults 0 2
    ```
    - `UUID=...`: O identificador único da sua partição.
    - `/mnt/meudisco`: O ponto onde a partição será montada.
    - `ext4`: O sistema de arquivos (pode ser `exfat`, `ntfs`, etc.).
    - `defaults`: Opções de montagem padrão (inclui `rw`, `suid`, `dev`, `exec`, `auto`, `nouser`, `async`).
    - `0 2`: `0` para não fazer dump, `2` para verificar o sistema de arquivos na inicialização (a raiz usa `1`).

4.  **Teste o `fstab`:**
    ```bash
    sudo mount -a
    ```
    Se não houver erros, a partição foi montada com sucesso. Você pode verificar com `df -h`.

---

<a id="6"></a>
## 6. Solução de Problemas: Erros Comuns no fstab

Um erro no `/etc/fstab` pode impedir o sistema de inicializar. Se isso acontecer, siga estes passos:

1.  **Modo de Recuperação (Rescue Mode):**
    Quando o sistema travar na inicialização, reinicie e, na tela do GRUB, escolha "Advanced options for Ubuntu" (ou similar) e selecione "Recovery mode" (Modo de recuperação).

2.  **Shell de Root:**
    No menu de recuperação, escolha a opção "root - Drop to root shell prompt".

3.  **Remontar a Raiz (se necessário):**
    O sistema de arquivos raiz pode estar montado como somente leitura. Remonte-o como leitura/escrita:
    ```bash
    mount -o remount,rw /
    ```

4.  **Edite o `fstab`:**
    ```bash
    nano /etc/fstab
    ```
    Comente a linha que você adicionou (coloque um `#` no início) ou corrija o erro.

5.  **Salve, Saia e Reinicie:**
    Salve o arquivo (`Ctrl+O`, `Enter`), saia (`Ctrl+X`) e reinicie o sistema.
    ```bash
    reboot
    ```
    Após reiniciar, você poderá corrigir o `fstab` com calma.

---

## Referências (fontes para consulta)

### Manpages (Debian)

- `lsblk(8)`: https://manpages.debian.org/bookworm/util-linux/lsblk.8.en.html
- `fdisk(8)`: https://manpages.debian.org/bookworm/util-linux/fdisk.8.en.html
- `parted(8)`: https://manpages.debian.org/bookworm/parted/parted.8.en.html
- `mkfs.ext4(8)`: https://manpages.debian.org/bookworm/e2fsprogs/mkfs.ext4.8.en.html
- `fstab(5)`: https://manpages.debian.org/bookworm/mount/fstab.5.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
