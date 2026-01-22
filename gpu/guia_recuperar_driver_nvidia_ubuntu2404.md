# Driver NVIDIA no Ubuntu 24.04 — recuperar e manter (passo a passo)

*Criado em 03 de dezembro de 2025 e atualizado em 08 de dezembro de 2025*

> **Compatível com Ubuntu 24.04 LTS (Noble).**  
> Foco: notebooks híbridos (Intel + NVIDIA, ex.: Lenovo IdeaPad GTX 1650).  
> O guia cobre instalação via `.run` **e** via pacotes `apt`, diferenças entre kernel **genérico vs OEM**, uso de PRIME/Optimus e correções para o erro **`unable to load the nvidia-drm kernel module`**.

---

## Atalhos de Cenário (Para onde ir?)
Este guia é longo. Encontre seu problema e pule para a seção correta.

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

```bash
# Ver kernel ativo
uname -r

# Secure Boot deve estar desativado
mokutil --sb-state

# Adicionar KMS da NVIDIA e bloquear simpledrm (após instalação do driver)
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-kms.conf
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="nvidia-drm.modeset=1 modprobe.blacklist=simpledrm /' /etc/default/grub
sudo update-grub
```

> **Dica:** Para notebooks, recomendamos **PRIME on-demand** (seção 7).

---

<a id="1"></a>
## 1) Entendendo kernels no Ubuntu

| Tipo | Exemplo | Quando usar | Observações |
|------|---------|-------------|------------|
| **generic (suportado)** | `6.8.0-87-generic` | **Recomendado** | Integra com DKMS, drivers, atualizações oficiais. |
| **oem** | `6.8.0-1032-oem` | Alguns notebooks | Pode dar conflito com `.run` da NVIDIA. |
| **mainline (não suportado)** | `6.14.0-27-generic` | Testes | Sem integração com DKMS/apt; costuma quebrar NVIDIA. |

**Conclusão:** Prefira **generic**. Evite *mainline* e, se usar **OEM**, prefira driver via `apt`.

---

<a id="2"></a>
## 2) Trocar para o kernel genérico (se ainda não estiver)

```bash
sudo apt update
sudo apt install -y linux-generic
sudo reboot
# Depois do boot:
uname -r   # deve ficar algo como 6.8.0-xx-generic
```

Se ainda inicializar no kernel antigo, defina o genérico no GRUB:

```bash
# Descobrir entradas do GRUB
sudo grep "menuentry '" /boot/grub/grub.cfg | cut -d"'" -f2

# Editar e fixar o genérico como padrão (copie o nome exato que aparecer)
sudo nano /etc/default/grub
# Substituir a linha por algo como:
# GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-87-generic"

sudo update-grub
sudo reboot
```

> (Opcional) Remover kernels antigos/confusos (ex.: 6.14.*):  
> `sudo apt remove --purge "linux-image-6.14.*" "linux-headers-6.14.*"`

---

<a id="3"></a>
## 3) Escolha **um** método de driver (não misture)

### Opção A — Driver via **pacotes Ubuntu** (mais integrado)
```bash
sudo apt purge -y 'nvidia*'
sudo apt update
sudo apt install -y nvidia-driver-550
sudo reboot
```

### Opção B — Driver via **instalador oficial** `.run` da NVIDIA
> Ideal quando precisa da **versão exata** do site da NVIDIA (ex.: 580.95.05) ou CUDA mais novo.

```bash
# Pare a interface gráfica (modo texto)
sudo systemctl isolate multi-user.target

# Instale o .run (exemplo de arquivo na pasta Downloads)
sudo sh ~/Downloads/NVIDIA-Linux-x86_64-580.95.05.run --dkms --no-cc-version-check

# NÃO deixe o instalador rodar nvidia-xconfig em notebooks híbridos (responda "No")
sudo update-initramfs -u -k $(uname -r)
sudo reboot
```

> **Não misture** `.run` com pacotes `apt`. Se for trocar, **desinstale** antes:  
> `sudo sh NVIDIA-Linux-*.run --uninstall` **ou** `sudo apt purge 'nvidia*'` e **reboot**.

---

<a id="4"></a>
## 4) Bloquear Nouveau e habilitar KMS (obrigatório para estabilidade)

```bash
# Bloquear Nouveau
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf

# Habilitar KMS da NVIDIA e neutralizar simpledrm
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-kms.conf
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="nvidia-drm.modeset=1 modprobe.blacklist=simpledrm /' /etc/default/grub

sudo update-initramfs -u -k $(uname -r)
sudo update-grub
sudo reboot
```

---

<a id="5"></a>
## 5) Testes de verificação pós-instalação

```bash
# Módulos carregados
lsmod | grep -E 'nvidia|drm'

# Status do driver
nvidia-smi

# OpenGL e Vulkan (com PRIME)
glxinfo | grep "OpenGL renderer"        # pacote: mesa-utils
vulkaninfo | grep "GPU id"              # pacote: vulkan-tools

# Se usar on-demand:
prime-run glxinfo | grep "OpenGL renderer"
prime-run vulkaninfo | grep "GPU id"
```

> Se `nvidia-smi` falhar, veja logs:  
> `sudo dmesg -T | tail -n 200 | grep -iE 'nvidia|drm|nouveau'`

---

<a id="6"></a>
## 6) PRIME / Alternar Intel ↔ NVIDIA

```bash
# Ver perfil atual
prime-select query

# Híbrido (recomendado para notebooks): Intel padrão, NVIDIA sob demanda
sudo prime-select on-demand
sudo reboot

# Forçar NVIDIA como primária
sudo prime-select nvidia
sudo reboot

# Forçar Intel como primária
sudo prime-select intel
sudo reboot

# Persistência (evita descarregar módulo quando inativo)
sudo systemctl enable --now nvidia-persistenced
# ou também:
sudo nvidia-smi -pm 1
```

> Se notar instabilidade com **Wayland**, teste com **Xorg**: na tela de login, engrenagem → “Ubuntu on Xorg”.  
> (para fixar) `sudo sed -i 's/^#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf && sudo systemctl restart gdm3`

---

<a id="7"></a>
## 7) CUDA (opcional, para IA/ML)

```bash
# Repositório CUDA (24.04)
# (Se já adicionou, pode pular. Evite apt-key; use chaves em /usr/share/keyrings)
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub \
  | sudo gpg --dearmor -o /usr/share/keyrings/cuda-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/cuda-keyring.gpg] https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /" \
  | sudo tee /etc/apt/sources.list.d/cuda-ubuntu2404.list

sudo apt update
sudo apt install -y cuda-toolkit-13
# Teste rápido:
cat > cuda_test.cu <<'EOF'
#include <stdio.h>
__global__ void helloCUDA(){ printf("Hello from GPU!\n"); }
int main(){ helloCUDA<<<1,1>>>(); cudaDeviceSynchronize(); return 0; }
EOF
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
   # Se necessário, limpe e reinstale (ajuste versão):
   sudo dkms remove -m nvidia -v 580.95.05 --all || true
   sudo dkms build  -m nvidia -v 580.95.05 -k $(uname -r)
   sudo dkms install -m nvidia -v 580.95.05 -k $(uname -r)
   ```

4. **KMS ativo e simpledrm bloqueado?**
   ```bash
   echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-kms.conf
   sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="nvidia-drm.modeset=1 modprobe.blacklist=simpledrm /' /etc/default/grub
   sudo update-initramfs -u -k $(uname -r)
   sudo update-grub
   sudo reboot
   ```

5. **O módulo está no initramfs?**
   ```bash
   lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'nvidia.*\.ko' || echo "NVIDIA não está no initramfs"
   ```

6. **Carregar manualmente e ver logs:**
   ```bash
   sudo modprobe -v nvidia nvidia-modeset nvidia-uvm
   sudo modprobe -v nvidia-drm modeset=1
   sudo dmesg -T | tail -n 200 | grep -iE 'nvidia|drm|nouveau'
   ```

7. **Se OEM continuar problemático → migre para genérico:**
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

```bash
# Kernel e módulos
uname -r
ls /lib/modules/$(uname -r)/updates/dkms/
modinfo nvidia_drm | egrep 'filename|vermagic|depends' || echo "sem nvidia_drm para este kernel"

# Initramfs contém os módulos?
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'nvidia.*\.ko' | head || echo "sem nvidia no initramfs"

# Log do kernel
sudo dmesg -T | tail -n 200 | grep -iE 'nvidia|drm|nouveau'

# Estado do DKMS
sudo dkms status
```

---

<a id="12"></a>
### Conclusão

- Para ter mais estabilidade no Ubuntu 24.04 com notebooks híbridos, o caminho mais simples e seguro costuma ser **kernel genérico + driver NVIDIA via `apt`**.  
- Se você realmente precisar usar o `.run` (para uma versão específica ou CUDA), lembre-se: **mantenha o kernel genérico**, Secure Boot desligado, Nouveau bloqueado e KMS ativado.  
- E, claro, se o erro `nvidia-drm` aparecer, siga o **Playbook Rápido** (seção 8).

Fim.

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

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
