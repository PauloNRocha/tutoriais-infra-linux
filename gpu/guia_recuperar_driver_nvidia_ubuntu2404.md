# Driver NVIDIA no Ubuntu 24.04 — recuperar e manter (passo a passo)

*Criado em: 03 de dezembro de 2025*  
*Última atualização em: 10 de março de 2026*

Fiz este guia para deixar anotado o processo que costumo seguir quando preciso recuperar o driver da NVIDIA no Ubuntu 24.04 ou evitar quebrar tudo de novo depois de update de kernel, troca de driver ou tentativa de instalação via `.run`.

O foco aqui é notebook híbrido com Intel + NVIDIA, mas boa parte do fluxo também serve como referência para outros cenários. O guia cobre instalação via `.run` e via `apt`, diferença entre kernel **genérico vs OEM**, uso de PRIME/Optimus e correções para o erro **`unable to load the nvidia-drm kernel module`**.

---

## Atalhos de cenário (para onde ir)

Se você já sabe qual é o sintoma, pode pular direto para a seção correspondente.

- **"Acabei de atualizar o kernel e a tela ficou preta!"**
  - **Causa provável:** O módulo DKMS da NVIDIA não foi recompilado para o novo kernel.
  - **Solução:** Vá direto para o [**Playbook Rápido (Seção 8)**](#8), focando nos passos 2 (Headers) e 3 (DKMS).

- **"O comando `nvidia-smi` falhou ou diz que não consegue se comunicar com o driver."**
  - **Causa provável:** O módulo do kernel não está carregado. Isso pode ser por um conflito, Secure Boot ou `nouveau` ativo.
  - **Solução:** Comece pelo [**Playbook Rápido (Seção 8)**](#8). Verifique o Secure Boot (passo 1) e o KMS (passo 4).

- **"Quero fazer uma instalação limpa do zero."**
  - **Solução:** Siga o guia em ordem. Comece pela [**escolha do método de driver (Seção 3)**](#3) e depois garanta que o [**Nouveau está bloqueado (Seção 4)**](#4).

- **"Instalei o driver, mas os jogos/apps não usam a placa NVIDIA."**
  - **Causa provável:** O sistema PRIME não está configurado corretamente.
  - **Solução:** Pule para a [**Seção 6: PRIME / Alternar Intel ↔ NVIDIA**](#6).

---

## Índice rápido
1. [Entendendo kernels no Ubuntu](#1)
2. [Trocar para o kernel genérico](#2)
3. [Escolha um método de driver (não misture)](#3)
4. [Bloquear Nouveau e habilitar KMS](#4)
5. [Testes de verificação pós-instalação](#5)
6. [PRIME / Alternar Intel ↔ NVIDIA](#6)
7. [CUDA (opcional, para IA/ML)](#7)
8. [Playbook Rápido: erro `unable to load nvidia-drm`](#8)
9. [Manutenção e prevenção](#9)
10. [Checklist de validação final](#10)
11. [Diagnóstico rápido](#11)
12. [Conclusão](#12)

---

## Pré‑requisitos rápidos

Antes de mexer no driver, eu costumo conferir estes três pontos:

1. Ver qual kernel está rodando
```bash
uname -r
```

2. Confirmar se o Secure Boot está desligado
```bash
mokutil --sb-state
```

3. Se o driver já estiver instalado, garantir KMS da NVIDIA e bloquear `simpledrm`
```bash
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-kms.conf
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="nvidia-drm.modeset=1 modprobe.blacklist=simpledrm /' /etc/default/grub
sudo update-grub
```

> **Dica:** Para notebook, eu costumo usar **PRIME on-demand** (seção 6).
>
> **Nota rápida sobre `simpledrm`:** ele é um driver DRM bem simples que o kernel usa cedo no boot para garantir saída de vídeo básica. Em alguns cenários, essa camada inicial atrapalha a transição para o KMS da NVIDIA, então faz sentido bloquear `simpledrm` quando o objetivo é estabilizar o driver proprietário.

---

<a id="1"></a>
## 1) Entendendo kernels no Ubuntu

| Tipo | Exemplo | Quando usar | Observações |
|------|---------|-------------|------------|
| **generic (suportado)** | `6.8.0-87-generic` | **Recomendado** | Integra com DKMS, drivers, atualizações oficiais. |
| **oem** | `6.8.0-1032-oem` | Alguns notebooks | Pode dar conflito com `.run` da NVIDIA. |
| **mainline (não suportado)** | `6.14.0-27-generic` | Testes | Sem integração com DKMS/apt; costuma quebrar NVIDIA. |

**Na prática:** se puder escolher, prefira **generic**. Evite *mainline* e, se for usar **OEM**, eu tenderia a ficar com driver via `apt`.

---

<a id="2"></a>
## 2) Trocar para o kernel genérico (se ainda não estiver)

1. Instale o meta-pacote do kernel genérico:

```bash
sudo apt update
sudo apt install -y linux-generic
```

2. Reinicie o sistema:

```bash
sudo reboot
```

3. Depois do boot, confirme o kernel carregado:

```bash
uname -r
```

Resultado esperado:

- algo como `6.8.0-xx-generic`.

Se ainda inicializar no kernel antigo, defina o genérico no GRUB:

4. Liste as entradas disponíveis no GRUB:

```bash
sudo grep "menuentry '" /boot/grub/grub.cfg | cut -d"'" -f2
```

5. Edite o arquivo do GRUB e fixe o kernel genérico como padrão:

```bash
sudo nano /etc/default/grub
```

Exemplo de linha:


_GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-87-generic"_

6. Recarregue o GRUB e reinicie:

```bash
sudo update-grub
sudo reboot
```

7. Opcionalmente, remova kernels antigos ou confusos (ex.: `6.14.*`):

```bash
sudo apt remove --purge "linux-image-6.14.*" "linux-headers-6.14.*"
```

---

<a id="3"></a>
## 3) Escolha **um** método de driver (não misture)

### Opção A — Driver via **pacotes Ubuntu** (mais integrado)

1. Limpe restos de driver antigo:

```bash
sudo apt purge -y 'nvidia*'
```

2. Atualize a lista de pacotes:

```bash
sudo apt update
```

3. Instale o driver:

```bash
sudo apt install -y nvidia-driver-550
```

4. Reinicie:

```bash
sudo reboot
```

### Opção B — Driver via **instalador oficial** `.run` da NVIDIA
> Esse caminho faz mais sentido quando você realmente precisa da **versão exata** do site da NVIDIA (ex.: 580.95.05) ou de um CUDA mais novo.

1. Saia da interface gráfica e vá para modo texto:

```bash
sudo systemctl isolate multi-user.target
```

2. Execute o instalador `.run`:

```bash
sudo sh ~/Downloads/NVIDIA-Linux-x86_64-580.95.05.run --dkms --no-cc-version-check
```

3. Atualize o `initramfs`:

```bash
sudo update-initramfs -u -k $(uname -r)
```

4. Reinicie:

```bash
sudo reboot
```

Observação:

- em notebook híbrido, eu não deixaria o instalador rodar `nvidia-xconfig`.

> **Não misture** `.run` com pacotes `apt`. Se for trocar, **desinstale** antes:  
> `sudo sh NVIDIA-Linux-*.run --uninstall` **ou** `sudo apt purge 'nvidia*'` e **reboot**.

---

<a id="4"></a>
## 4) Bloquear Nouveau e habilitar KMS (obrigatório para estabilidade)

1. Bloqueie o `nouveau`:

```bash
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
```

2. Habilite o KMS da NVIDIA e neutralize o `simpledrm`:

```bash
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-kms.conf
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="nvidia-drm.modeset=1 modprobe.blacklist=simpledrm /' /etc/default/grub
```

3. Recrie o `initramfs`:

```bash
sudo update-initramfs -u -k $(uname -r)
```

4. Atualize o GRUB:

```bash
sudo update-grub
```

5. Reinicie:

```bash
sudo reboot
```

---

<a id="5"></a>
## 5) Testes de verificação pós-instalação

1. Veja se os módulos carregaram:

```bash
lsmod | grep -E 'nvidia|drm'
```

2. Confira o estado do driver:

```bash
nvidia-smi
```

3. Teste OpenGL e Vulkan:

```bash
glxinfo | grep "OpenGL renderer"        # pacote: mesa-utils
vulkaninfo | grep "GPU id"              # pacote: vulkan-tools
```

4. Se estiver usando `on-demand`, teste também com `prime-run`:

```bash
prime-run glxinfo | grep "OpenGL renderer"
prime-run vulkaninfo | grep "GPU id"
```

5. Se `nvidia-smi` falhar, confira o log do kernel:

```bash
sudo dmesg -T | tail -n 200 | grep -iE 'nvidia|drm|nouveau'
```

---

<a id="6"></a>
## 6) PRIME / Alternar Intel ↔ NVIDIA

1. Veja qual perfil está ativo:

```bash
prime-select query
```

2. Para usar o perfil híbrido (`on-demand`):

```bash
sudo prime-select on-demand
sudo reboot
```

3. Para forçar NVIDIA como primária:

```bash
sudo prime-select nvidia
sudo reboot
```

4. Para forçar Intel como primária:

```bash
sudo prime-select intel
sudo reboot
```

5. Se quiser manter persistência do módulo:

```bash
sudo systemctl enable --now nvidia-persistenced
```

Alternativa:

```bash
sudo nvidia-smi -pm 1
```

> Se notar instabilidade com **Wayland**, teste com **Xorg**: na tela de login, engrenagem → “Ubuntu on Xorg”.
>
> Se quiser fixar isso:
> ```bash
> sudo sed -i 's/^#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf
> sudo systemctl restart gdm3
> ```

---

<a id="7"></a>
## 7) CUDA (opcional, para IA/ML)

1. Adicione a chave do repositório CUDA:

```bash
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub \
  | sudo gpg --dearmor -o /usr/share/keyrings/cuda-keyring.gpg
```

2. Adicione o repositório:

```bash
echo "deb [signed-by=/usr/share/keyrings/cuda-keyring.gpg] https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /" \
  | sudo tee /etc/apt/sources.list.d/cuda-ubuntu2404.list
```

3. Atualize a lista:

```bash
sudo apt update
```

4. Instale o toolkit:

```bash
sudo apt install -y cuda-toolkit-13
```

5. Crie um teste mínimo:

```bash
cat > cuda_test.cu <<'EOF'
#include <stdio.h>
__global__ void helloCUDA(){ printf("Hello from GPU!\n"); }
int main(){ helloCUDA<<<1,1>>>(); cudaDeviceSynchronize(); return 0; }
EOF
```

6. Compile e execute:

```bash
nvcc cuda_test.cu -o cuda_test && ./cuda_test
```

---

<a id="8"></a>
## 8) **Playbook Rápido**: erro `unable to load the nvidia-drm kernel module`

1. **Secure Boot desativado?**
   ```bash
   mokutil --sb-state
   ```

2. **Headers corretos do kernel?**
   ```bash
   uname -r
   sudo apt install --reinstall -y linux-headers-$(uname -r)
   ```

3. **DKMS da NVIDIA compila para ESTE kernel?**
   ```bash
   sudo dkms status
   ```
   Se aparecer algo como `nvidia/<VERSAO_REAL>`, use essa mesma versão nos comandos abaixo.

4. **Se necessário, remover e reconstruir o módulo DKMS**
   ```bash
   sudo dkms remove -m nvidia -v <VERSAO_REAL> --all || true
   sudo dkms build  -m nvidia -v <VERSAO_REAL> -k $(uname -r)
   sudo dkms install -m nvidia -v <VERSAO_REAL> -k $(uname -r)
   ```

5. **KMS ativo e simpledrm bloqueado?**
   ```bash
   echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-kms.conf
   sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="nvidia-drm.modeset=1 modprobe.blacklist=simpledrm /' /etc/default/grub
   sudo update-initramfs -u -k $(uname -r)
   sudo update-grub
   ```

6. **O módulo está no initramfs?**
   ```bash
   lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'nvidia.*\.ko' || echo "NVIDIA não está no initramfs"
   ```

7. **Carregar manualmente e ver logs:**
   ```bash
   sudo modprobe -v nvidia nvidia-modeset nvidia-uvm
   sudo modprobe -v nvidia-drm modeset=1
   sudo dmesg -T | tail -n 200 | grep -iE 'nvidia|drm|nouveau'
   ```

8. **Reiniciar**
   ```bash
   sudo reboot
   ```

9. **Se OEM continuar problemático → migre para genérico:**
   ```bash
   sudo apt install -y linux-generic
   # defina no GRUB conforme seção 2
   ```

---

<a id="9"></a>
## 9) Manutenção e prevenção

- **Evitar que o apt troque seu driver** (se usa `.run`):
  ```bash
  sudo apt-mark hold nvidia-driver-570 nvidia-driver-580 libnvidia-compute-580 libxnvctrl0 ubuntu-drivers-common
  apt-mark showhold
  ```

- **Backup dos módulos do kernel (restore rápido):**
  ```bash
  sudo cp -r /lib/modules /lib/modules.bkp-$(date +%F)
  # restore (em emergência):
  # sudo rm -rf /lib/modules && sudo cp -r /lib/modules.bkp-AAAA-MM-DD /lib/modules
  ```

- **Limpar kernels antigos e entradas confusas (ex.: 6.14.*):**
  ```bash
  sudo apt remove --purge "linux-image-6.14.*" "linux-headers-6.14.*"
  sudo update-grub
  ```

---

<a id="10"></a>
## 10) Checklist de validação final

- [ ] `uname -r` mostra **kernel genérico** (ou OEM estável com `apt`).
- [ ] `mokutil --sb-state` → **SecureBoot disabled**.
- [ ] `lsmod | grep nvidia` mostra módulos ativos.
- [ ] `nvidia-smi` lista a GPU e versão do driver.
- [ ] `prime-select on-demand` ok (para notebooks).
- [ ] `glxinfo`/`vulkaninfo` funcionam (com/sem `prime-run`).

---

<a id="11"></a>
## 11) Diagnóstico rápido (quando algo sair do trilho)

1. Confira kernel e módulos DKMS:

```bash
uname -r
ls /lib/modules/$(uname -r)/updates/dkms/
modinfo nvidia_drm | egrep 'filename|vermagic|depends' || echo "sem nvidia_drm para este kernel"
```

2. Veja se o `initramfs` contém os módulos NVIDIA:

```bash
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'nvidia.*\.ko' | head || echo "sem nvidia no initramfs"
```

3. Leia o log do kernel:

```bash
sudo dmesg -T | tail -n 200 | grep -iE 'nvidia|drm|nouveau'
```

4. Confira o estado do DKMS:

```bash
sudo dkms status
```

---

<a id="12"></a>
## 12) Conclusão

Resumindo este guia em poucas linhas:

- para ter menos dor de cabeça no Ubuntu 24.04 com notebook híbrido, o caminho mais previsível costuma ser **kernel genérico + driver NVIDIA via `apt`**;
- se realmente precisar usar o `.run`, mantenha o ambiente o mais simples possível: **kernel genérico**, Secure Boot desligado, Nouveau bloqueado e KMS ativado;
- se aparecer erro com `nvidia-drm`, volte para o **Playbook Rápido** da seção 8 e siga a ordem sem tentar corrigir no improviso.

---

## Referências (fontes para consulta)

### NVIDIA / Ubuntu (base)

- CUDA Installation Guide for Linux: https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html
- NVIDIA Linux Driver README (instalação do driver): https://download.nvidia.com/XFree86/Linux-x86_64/580.82.09/README/installdriver.html
- PRIME Render Offload (NVIDIA driver README): https://download.nvidia.com/XFree86/Linux-x86_64/550.54.14/README/primerenderoffload.html
- NVIDIA / prime-select (Ubuntu Community Help): https://help.ubuntu.com/community/BinaryDriverHowto/Nvidia

### Manpages (referência)

- `dkms(8)`: https://manpages.debian.org/bookworm/dkms/dkms.8.en.html
- `update-initramfs(8)`: https://manpages.debian.org/bookworm/initramfs-tools/update-initramfs.8.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
