# Ubuntu — recuperar após apagar `/lib/modules` (guia de emergência)

*Criado em: 03 de dezembro de 2025*  
*Última atualização em: 11 de março de 2026*

Apagar `/lib/modules` costuma transformar um sistema funcionando em um problema sério em poucos minutos. Este guia registra o que fazer quando um Ubuntu deixa de funcionar por causa dessa remoção, que atinge justamente os módulos e dependências do kernel.

> [!WARNING]
> **Guia de emergência**: isso aqui é para cenário de desastre. O ideal continua sendo ter backup, mas se o sistema já quebrou, este roteiro ajuda a recuperar o boot e reconstruir os módulos do kernel com mais previsibilidade.

---

## Índice rápido
1. [Entendendo o problema](#1)
2. [Preparando o ambiente (Live USB)](#2)
3. [Atualizar repositórios](#3)
4. [Reinstalar o kernel e módulos](#4)
5. [Regenerar módulos e GRUB](#5)
6. [Reiniciar o sistema](#6)
7. [(Opcional) Remover kernel OEM](#7)
8. [Reinstalar driver NVIDIA (opcional)](#8)
9. [Backup preventivo](#9)
10. [Verificações finais](#10)
11. [Dica final](#11)
12. [Resumo dos comandos](#12)

---

<a id="1"></a>
## 1. Entendendo o problema

O diretório `/lib/modules` contém os arquivos essenciais para o kernel do Linux carregar drivers, inclusive os da NVIDIA, Wi-Fi, som, etc.

Se ele for apagado (por exemplo, com `sudo rm -rf /lib/modules`), o sistema ainda pode iniciar em modo de recuperação, mas:
- Nenhum driver do kernel é carregado;
- O `update-initramfs` falha;
- O `grub` pode perder as entradas de inicialização;
- Drivers como NVIDIA ou Wi-Fi deixam de funcionar.

---

<a id="2"></a>
## 2. Preparando o ambiente

Se o sistema ainda inicia no terminal, você pode pular esta parte.

Caso o Ubuntu não inicie:
1. Inicie o **Live CD/USB** do Ubuntu ("Experimentar sem instalar");
2. Abra o terminal;
3. Monte a partição raiz:

```bash
sudo mount /dev/nvme0n1p2 /mnt
```

Exemplo:

- `/dev/nvme0n1p2`: partição raiz
- `/dev/nvme0n1p1`: partição EFI

4. Monte a partição EFI:

```bash
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
```

5. Faça os bind mounts:

```bash
for i in /dev /dev/pts /proc /sys /run; do sudo mount --bind $i /mnt$i; done
```

6. Entre no sistema com `chroot`:

```bash
sudo chroot /mnt
```

Depois disso, os próximos comandos já serão executados dentro do sistema instalado.

---

<a id="3"></a>
## 3. Atualizar repositórios

```bash
sudo apt update
```
Isso garante que o sistema conheça os pacotes mais recentes disponíveis para reinstalação.

---

<a id="4"></a>
## 4. Reinstalar o kernel e módulos

Esta é a etapa crucial que recria o diretório `/lib/modules`.

### Opção A: Kernel genérico (Recomendado para a maioria dos casos)
```bash
sudo apt install --reinstall linux-generic linux-image-generic linux-headers-generic -y
```

### Opção B: Kernel OEM (se você usa um notebook com kernel customizado de fábrica)
```bash
# Exemplo para uma versão específica, ajuste para a sua
sudo apt install --reinstall linux-image-6.8.0-1032-oem linux-modules-6.8.0-1032-oem linux-headers-6.8.0-1032-oem -y
```

Se quiser garantir que todos os módulos extras sejam instalados:
```bash
sudo apt install linux-modules-extra-$(uname -r) -y
```

---

<a id="5"></a>
## 5. Regenerar módulos e GRUB

Após reinstalar o kernel, force a recriação das dependências dos módulos e do menu de boot:

1. Recrie as dependências dos módulos:

```bash
sudo depmod -a
```

2. Atualize o `initramfs`:

```bash
sudo update-initramfs -u -k all
```

3. Atualize o GRUB:

```bash
sudo update-grub
```

---

<a id="6"></a>
## 6. Reiniciar o sistema

1. Saia do `chroot`:

```bash
exit
```

2. Reinicie:

```bash
sudo reboot
```

---

Depois do boot, confira se o kernel e os módulos realmente voltaram:

```bash
uname -r
ls /lib/modules/
```

Se a recuperação deu certo, a saída deve mostrar novamente o diretório do kernel correspondente.

---

<a id="7"></a>
## 7. (Opcional) Remover kernel OEM

Se o kernel genérico estiver funcionando perfeitamente, você pode remover o OEM para evitar conflitos:

```bash
sudo apt purge 'linux-oem*' -y
sudo apt autoremove --purge -y
```

---

<a id="8"></a>
## 8. Reinstalar driver NVIDIA (opcional)

Após restaurar o kernel, o driver da NVIDIA provavelmente precisará ser reinstalado.

### Via repositório Ubuntu:
```bash
sudo apt install nvidia-driver-550 -y
```

### Via instalador `.run`:

1. Entre em modo texto e pare a interface gráfica:

```bash
sudo systemctl isolate multi-user.target
```

2. Execute o instalador:

```bash
sudo ./NVIDIA-Linux-x86_64-580.95.run
```

---

<a id="9"></a>
## 9. Backup preventivo

A melhor cura é a prevenção. Após o sistema estar funcionando, faça um backup do diretório de módulos:

```bash
sudo cp -r /lib/modules /backup-modules/
```

Se algo quebrar no futuro, você pode restaurá-lo rapidamente:
```bash
sudo rm -rf /lib/modules
```

```bash
sudo cp -r /backup-modules /lib/modules
```

---

<a id="10"></a>
## 10. Verificações finais

1. Verifique qual kernel carregou:

```bash
uname -r
```

2. Verifique se os módulos NVIDIA estão ativos:

```bash
lsmod | grep nvidia
```

3. Verifique o driver:

```bash
nvidia-smi
```

4. Verifique se o GRUB reconhece o kernel:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

<a id="11"></a>
## 11. Dica final
> Antes de mexer com driver, kernel ou módulos manualmente, faça um backup rápido de `/lib/modules`:
> ```bash
> sudo cp -r /lib/modules /lib/modules.bkp-$(date +%F)
> ```
> Isso não substitui backup de verdade, mas já ajuda muito quando o problema é local e você precisa voltar rápido.

---

<a id="12"></a>
## 12. Resumo dos comandos principais

Dentro do `chroot` no Live USB:
```bash
sudo apt update
sudo apt install --reinstall linux-generic linux-image-generic linux-headers-generic -y
sudo depmod -a
sudo update-initramfs -u -k all
sudo update-grub
exit
sudo reboot
```

---

## Referências (fontes para consulta)

### Manpages (referência)

- `chroot(1)`: https://man7.org/linux/man-pages/man1/chroot.1.html
- `update-initramfs(8)`: https://manpages.debian.org/bookworm/initramfs-tools/update-initramfs.8.en.html
- `grub-mkconfig(8)`: https://manpages.debian.org/bookworm/grub-common/grub-mkconfig.8.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
