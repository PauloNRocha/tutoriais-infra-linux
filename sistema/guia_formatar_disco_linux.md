# Guia Prático: Formatar discos e partições no Linux com segurança

*Criado em: 04 de dezembro de 2025*  
*Última atualização em: 18 de maio de 2026*

Formatar disco no Linux não é difícil, mas é uma daquelas tarefas em que pressa custa caro. O objetivo aqui é seguir um fluxo seguro: identificar o disco certo, criar uma partição, formatar, montar e, se fizer sentido, deixar a montagem fixa no `/etc/fstab`.

> [!WARNING]
> Formatar uma partição apaga os dados dela. Criar uma nova tabela de partições apaga a estrutura do disco inteiro. Antes de rodar `fdisk`, `mkfs` ou qualquer comando parecido, confira o dispositivo pelo tamanho, modelo, serial e ponto de montagem.

Nos comandos abaixo, `/dev/sdX` representa o disco de exemplo e `/dev/sdX1` representa a partição de exemplo. Substitua pelo dispositivo real do seu ambiente.

Os exemplos usam `sudo`. Se você estiver logado diretamente como `root`, execute os comandos sem `sudo`.

Em NVMe, o nome costuma ser diferente:

- disco: `/dev/nvme0n1`
- partição: `/dev/nvme0n1p1`

Na prática, a ordem dos discos pode mudar depois de um reboot. Em uma VM Debian 13, por exemplo, o disco secundário que aparecia como `/dev/sdb` passou a aparecer como `/dev/sda` depois de reiniciar. A montagem continuou correta porque o `/etc/fstab` usava `UUID`, não o nome `/dev/sdX`.

---

## Índice rápido

1. [Instalar ferramentas necessárias](#1)
2. [Identificar o disco correto](#2)
3. [Desmontar partições em uso](#3)
4. [Criar tabela GPT e uma partição](#4)
5. [Formatar a partição](#5)
6. [Montar manualmente para testar](#6)
7. [Montagem permanente via fstab](#7)
8. [Problemas comuns no fstab](#8)
9. [Referências](#referencias)

---

<a id="1"></a>
## 1. Instalar ferramentas necessárias

No Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y util-linux fdisk parted e2fsprogs
```

Na prática, `util-linux` fornece ferramentas como `lsblk`, `blkid`, `findmnt` e `mount`; `fdisk` fornece o particionador; `parted` fornece o `partprobe`; e `e2fsprogs` fornece o `mkfs.ext4`.

Se for formatar em exFAT ou NTFS, instale os pacotes específicos na etapa de formatação.

---

<a id="2"></a>
## 2. Identificar o disco correto

Este é o passo mais importante do guia.

Liste os discos com colunas úteis:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Exemplo:

```text
NAME   SIZE TYPE FSTYPE LABEL UUID                                 MOUNTPOINTS MODEL        SERIAL
sda  238.5G disk                                                   SSD-SISTEMA 123ABC
├─sda1 512M part vfat         1111-2222                            /boot/efi
└─sda2 238G part ext4         aaaa-bbbb-cccc-dddd                  /
sdb  931.5G disk                                                   HD-EXTERNO  987XYZ
└─sdb1 931G part ext4  Backup 3333-4444-5555-6666                  /media/paulo/Backup
```

No exemplo acima:

- `/dev/sda` é o disco do sistema, porque tem `/` e `/boot/efi`;
- `/dev/sdb` é um disco externo de 931,5 GB.

Confira também com `fdisk`:

```bash
sudo fdisk -l /dev/sdX
```

Antes de continuar, confirme:

- o tamanho do disco;
- o modelo;
- o serial, quando disponível;
- se não há partição do sistema montada nele;
- se você realmente quer apagar esse disco ou partição.

> [!CAUTION]
> Não continue se o disco tiver `/`, `/boot`, `/boot/efi`, `/home` ou qualquer montagem importante que você não pretende apagar.

---

<a id="3"></a>
## 3. Desmontar partições em uso

Se a partição estiver montada, desmonte antes de alterar tabela de partição ou formatar.

Exemplo:

```bash
sudo umount /dev/sdX1
```

Se alguma partição desse disco estiver sendo usada como swap, desative antes:

```bash
sudo swapoff /dev/sdX2
```

Confira se ainda existe montagem ativa:

```bash
findmnt /dev/sdX1
```

Se `findmnt` não retornar nada para a partição, ela não está montada.

---

<a id="4"></a>
## 4. Criar tabela GPT e uma partição

Esta etapa apaga a estrutura de partições anterior do disco.

Abra o disco inteiro no `fdisk`, não a partição:

```bash
sudo fdisk /dev/sdX
```

Dentro do `fdisk`, siga esta sequência:

1. Digite `g` para criar uma nova tabela GPT.
2. Digite `n` para criar uma nova partição.
3. Pressione `Enter` para aceitar o número padrão da partição.
4. Pressione `Enter` para aceitar o primeiro setor padrão.
5. Pressione `Enter` para aceitar o último setor padrão e usar o disco todo.
6. Se o `fdisk` perguntar se deve remover uma assinatura antiga, responda `Y`.
7. Digite `p` para revisar a tabela antes de gravar.
8. Digite `w` para gravar as alterações e sair.

Depois, peça ao kernel para reler a tabela de partições:

```bash
sudo partprobe /dev/sdX
```

Confira:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL /dev/sdX
```

Ao final, você deve ter uma partição nova, como `/dev/sdX1`.

---

<a id="5"></a>
## 5. Formatar a partição

Formate a partição, não o disco inteiro.

Correto:

```bash
sudo mkfs.ext4 -L dados /dev/sdX1
```

Errado:

```bash
sudo mkfs.ext4 /dev/sdX
```

### 5.1 ext4 para uso Linux

Para disco usado apenas em Linux, `ext4` costuma ser a escolha simples e estável.

```bash
sudo mkfs.ext4 -L dados /dev/sdX1
```

### 5.2 exFAT para compatibilidade com Windows, macOS e Linux

exFAT é uma boa opção para pendrive, HD externo e SSD externo que precisa circular entre sistemas diferentes.

Instale a ferramenta:

```bash
sudo apt update
sudo apt install -y exfatprogs
```

Formate:

```bash
sudo mkfs.exfat -L DADOS /dev/sdX1
```

### 5.3 NTFS para uso específico com Windows

Use NTFS quando o disco precisa ser usado principalmente em Windows.

Instale a ferramenta:

```bash
sudo apt update
sudo apt install -y ntfs-3g
```

Formate:

```bash
sudo mkfs.ntfs -f -L DADOS /dev/sdX1
```

A opção `-f` faz formatação rápida.

Ao montar um volume NTFS no Linux com `ntfs-3g`, comandos como `findmnt` ou `df -T` podem mostrar o tipo como `fuseblk`. Isso é normal.

### 5.4 Verificar resultado

Confira o sistema de arquivos criado:

```bash
lsblk -f /dev/sdX
sudo blkid /dev/sdX1
```

---

<a id="6"></a>
## 6. Montar manualmente para testar

Crie um ponto de montagem:

```bash
sudo mkdir -p /mnt/meudisco
```

Monte a partição:

```bash
sudo mount /dev/sdX1 /mnt/meudisco
```

Confira:

```bash
findmnt /mnt/meudisco
df -hT /mnt/meudisco
```

Se for um disco `ext4` novo para uso do usuário atual, ajuste o dono da raiz da partição:

```bash
sudo chown "$USER":"$USER" /mnt/meudisco
```

Se montou corretamente, teste criar um arquivo simples:

```bash
touch /mnt/meudisco/teste-gravacao
ls -l /mnt/meudisco/teste-gravacao
```

Depois, se quiser desmontar:

```bash
sudo umount /mnt/meudisco
```

---

<a id="7"></a>
## 7. Montagem permanente via fstab

Para montar automaticamente no boot, use `/etc/fstab`.

Antes de editar, faça backup:

```bash
sudo cp -a /etc/fstab /etc/fstab.bak.$(date +%F_%H%M%S)
```

Descubra o UUID da partição:

```bash
sudo blkid /dev/sdX1
```

Crie o ponto de montagem:

```bash
sudo mkdir -p /mnt/meudisco
```

Edite o arquivo e adicione a linha correspondente ao sistema de arquivos escolhido:

```bash
sudo nano /etc/fstab
```

### 7.1 Exemplo para ext4

Para disco secundário, externo ou não crítico para o boot, eu prefiro usar `nofail` e timeout curto. Assim, se o disco não estiver presente, o sistema não fica preso no boot.

```fstab
UUID=1234abcd-5678-efab-cdef-1234567890ab /mnt/meudisco ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2
```

Campos:

- `UUID=...`: identificador único da partição;
- `/mnt/meudisco`: ponto de montagem;
- `ext4`: sistema de arquivos;
- `defaults,nofail,x-systemd.device-timeout=10s`: opções de montagem;
- `0`: não usar `dump`;
- `2`: permitir checagem do sistema de arquivos depois da raiz.

Se o disco for crítico para um serviço e você quer que falha de montagem pare o boot, remova `nofail` de propósito e documente isso no ambiente.

### 7.2 Exemplo para exFAT

```fstab
UUID=1234-ABCD /mnt/meudisco exfat defaults,nofail,x-systemd.device-timeout=10s,uid=1000,gid=1000,umask=022 0 0
```

Em exFAT e NTFS, ajuste `uid` e `gid` conforme o usuário que deve gravar no disco.

Para descobrir seu UID e GID:

```bash
id
```

### 7.3 Exemplo para NTFS

```fstab
UUID=1234567890ABCDEF /mnt/meudisco ntfs defaults,nofail,x-systemd.device-timeout=10s,uid=1000,gid=1000,umask=022 0 0
```

### 7.4 Testar antes de reiniciar

Em Debian com `systemd`, o próprio `/etc/fstab` avisa que as units de montagem são geradas a partir desse arquivo. Depois de salvar o `fstab`, recarregue o `systemd` antes de testar:

```bash
sudo systemctl daemon-reload
```

Valide o `/etc/fstab`:

```bash
sudo findmnt --verify --verbose
```

O comando valida o arquivo inteiro. Em alguns sistemas podem aparecer avisos não relacionados ao disco novo, por exemplo uma entrada antiga de CD-ROM/ISO. O ponto principal é não haver erro de parse e confirmar que a linha do novo ponto de montagem aparece traduzindo o `UUID` para a partição correta.

Monte tudo que estiver configurado para montagem automática:

```bash
sudo mount -a
```

Confira:

```bash
findmnt /mnt/meudisco
df -hT /mnt/meudisco
```

Não reinicie antes de testar o `fstab`. Um erro nesse arquivo pode atrapalhar o boot.

Depois que o teste manual passar, você pode reiniciar e confirmar se a montagem voltou sozinha:

```bash
sudo reboot
```

Após o boot:

```bash
findmnt /mnt/meudisco
df -hT /mnt/meudisco
lsblk -f
```

Exemplo de saída em Debian 13:

```text
TARGET        SOURCE    FSTYPE OPTIONS
/mnt/meudisco /dev/sda1 ext4   rw,relatime
```

Mesmo que o `SOURCE` apareça como `/dev/sda1`, `/dev/sdb1` ou outro nome depois do reboot, o importante é o ponto de montagem estar correto. O `/etc/fstab` com `UUID` protege justamente contra essa troca de ordem dos discos.

---

<a id="8"></a>
## 8. Problemas comuns no fstab

### 8.1 O sistema não monta depois do boot

Confira se o UUID mudou:

```bash
sudo blkid /dev/sdX1
grep meudisco /etc/fstab
```

Se você formatou de novo, o UUID provavelmente mudou. Atualize o `/etc/fstab`.

### 8.2 O boot demora esperando disco externo

Para disco externo, secundário ou que pode não estar conectado, use:

```fstab
nofail,x-systemd.device-timeout=10s
```

Sem isso, o sistema pode esperar o dispositivo por mais tempo durante o boot.

### 8.3 Erro no fstab impediu o boot normal

Entre pelo modo de recuperação do GRUB, escolha um shell de root e remonte a raiz como leitura/escrita:

```bash
mount -o remount,rw /
```

Edite o arquivo:

```bash
nano /etc/fstab
```

Comente a linha problemática colocando `#` no início, salve e reinicie:

```bash
reboot
```

Depois que o sistema voltar, corrija com calma usando o backup criado antes da alteração.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### Manpages Debian

- `lsblk(8)` Debian Trixie: https://manpages.debian.org/trixie/util-linux/lsblk.8.en.html
- `fdisk(8)` Debian Trixie: https://manpages.debian.org/trixie/fdisk/fdisk.8.en.html
- `parted(8)` Debian Trixie: https://manpages.debian.org/trixie/parted/parted.8.en.html
- `mkfs.ext4(8)` Debian Trixie: https://manpages.debian.org/trixie/e2fsprogs/mkfs.ext4.8.en.html
- `mkfs.exfat(8)` Debian Trixie: https://manpages.debian.org/trixie/exfatprogs/mkfs.exfat.8.en.html
- `mkfs.ntfs(8)` Debian Trixie: https://manpages.debian.org/trixie/ntfs-3g/mkfs.ntfs.8.en.html
- `blkid(8)` Debian Trixie: https://manpages.debian.org/trixie/util-linux/blkid.8.en.html
- `findmnt(8)` Debian Trixie: https://manpages.debian.org/trixie/util-linux/findmnt.8.en.html
- `fstab(5)` Debian Trixie: https://manpages.debian.org/trixie/mount/fstab.5.en.html
- `systemd.mount(5)` Debian Trixie: https://manpages.debian.org/trixie/systemd/systemd.mount.5.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
