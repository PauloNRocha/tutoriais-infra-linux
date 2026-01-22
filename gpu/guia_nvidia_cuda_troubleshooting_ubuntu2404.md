# NVIDIA + CUDA no Ubuntu 24.04 — solução de problemas (Ideapad GTX 1650)

*Criado em 29 de setembro de 2025 e atualizado em 08 de dezembro de 2025*

Este guia reúne **erros reais** que podem ocorrer durante a instalação/configuração, com **causas prováveis** e **correções passo a passo**, para complementar o tutorial principal.  
A estrutura é **didática**: cada seção traz **Sintomas → Causa → Como resolver**, com exemplos de comandos e o que esperar.

> Ambiente-alvo: **Ubuntu 24.04 (Noble)**, **kernel OEM 6.8.0-1032-oem**, **Lenovo Ideapad**, **NVIDIA GTX 1650 Mobile (TU117M)**, **driver proprietário 580.82.09** e **CUDA 13.0**.

---

## Índice rápido
1. [Fluxo de Diagnóstico Rápido](#1)
2. [Instalador quer “downgrade” do driver (580 → 570)](#2)
3. [Aviso: X server em execução](#3)
4. [Aviso: método alternativo detectado (driver via Ubuntu)](#4)
5. [Erro 32-bit compatibility (i386) ausente](#5)
6. [Aviso libGLVND / pkg-config ausente](#6)
7. [`nvidia-smi` falhou (driver não comunica)](#7)
8. [Módulo não carrega / Secure Boot / Kernel “tainted”](#8)
9. [`prime-run` não encontrado](#9)
10. [`glxinfo` não encontrado](#10)
11. [`nvcc` ausente (CUDA Toolkit não instalado)](#11)
12. [APT alerta: repositório sem i386 (OpenVPN3)](#12)
13. [Módulos compilados para kernel errado / headers faltando](#13)
14. [OEM vs Kernel genérico: o que usar?](#14)
15. [“No NVIDIA modules detected in initramfs”](#15)
16. [Escolha “NVIDIA Proprietary” vs “MIT/GPL”](#16)
17. [Checklist final e comandos úteis](#17)

---

<a id="1"></a>
## 1. Fluxo de Diagnóstico Rápido
Quando algo dá errado com o driver da NVIDIA, siga estes 5 passos antes de mergulhar nos erros específicos. Muitas vezes, eles já resolvem o problema.

1.  **O driver está carregado?**
    ```bash
    nvidia-smi
    ```
    *Se funcionar, o driver está OK. Se falhar com "couldn't communicate", o módulo não carregou. Vá para o passo 2.*

2.  **O que os logs do kernel dizem?**
    ```bash
    sudo dmesg -T | grep -iE 'nvidia|nouveau|secure boot' | tail -n 20
    ```
    *Procure por erros óbvios como "module verification failed", "signature and/or required key missing" (indica problema com Secure Boot) ou menções ao `nouveau` (conflito).*

3.  **O Secure Boot está ativo?**
    ```bash
    mokutil --sb-state
    ```
    *Se a resposta for `SecureBoot enabled`, este é um forte candidato a ser a causa do problema. Desative-o na BIOS/UEFI ou veja a seção sobre [assinatura de módulos](#8).*

4.  **O `nouveau` está realmente desativado?**
    ```bash
    lsmod | grep nouveau
    ```
    *Se este comando retornar alguma coisa, o `nouveau` ainda está ativo e causando conflito. Garanta que o arquivo em `/etc/modprobe.d/blacklist-nouveau.conf` está correto.*

5.  **Você tem os `headers` corretos para o seu kernel?**
    ```bash
    uname -r
    sudo apt install linux-headers-$(uname -r)
    ```
    *Se o `apt` instalar um novo pacote, é provável que esta fosse a causa. Após a instalação, reinstale o driver `.run` da NVIDIA e reinicie.*

---

<a id="2"></a>
## 2) **Instalador tenta “downgrade” (580 → 570) com `ubuntu-drivers autoinstall`**

**Sintoma**  
Ao rodar `ubuntu-drivers autoinstall`, o sistema tenta **remover** a série **580** e **instalar** a **570**.

**Causa**  
O `ubuntu-drivers` escolhe o **driver recomendado “estável”** pela Canonical (geralmente a série 570), mesmo que você já esteja numa versão mais nova (580).

**Como resolver (manter 580):**
```bash
# Evite rodar 'ubuntu-drivers autoinstall' se já estiver ok no 580
# Para garantir o 580 proprietário instalado via .run, não use o metapacote do Ubuntu

sudo apt-mark hold libnvidia-compute-580 libxnvctrl0 nvidia-driver-580 nvidia-driver-580-open nvidia-driver-570 ubuntu-drivers-common
```
> Dica: Se quiser **sempre** 580 proprietário (via `.run`), mantenha distância do `autoinstall` e dos metapacotes do Ubuntu.

---

<a id="3"></a>
## 3) **“You appear to be running an X server”**

**Sintoma**  
Durante o `.run`, aparece aviso pedindo para **não instalar com a interface gráfica ativa**.

**Causa**  
O instalador precisa parar o **Xorg/Wayland** para compilar/configurar módulos com segurança.

**Como resolver:**
```bash
# Entrar em TTY:
Ctrl + Alt + F3

# Parar o display manager (ajuste ao seu caso):
sudo systemctl stop gdm
# ou: sudo systemctl stop sddm
# ou: sudo systemctl stop lightdm

# Executar o instalador:
sudo ./NVIDIA-Linux-x86_64-580.82.09.run

# No fim, reinicie:
sudo reboot
```

---

<a id="4"></a>
## 4) **“An alternate method of installing the NVIDIA driver was detected”**

**Sintoma**  
O `.run` detecta que já existe um **driver instalado via pacotes do Ubuntu** e sugere usar o método do sistema.

**Causa**  
Coexistência de dois métodos (APT vs `.run`) pode gerar conflitos.

**Como resolver (ficar com o .run):**
- No prompt, escolha **Continue installation**.  
- Depois da instalação bem-sucedida, remova vestígios dos pacotes APT para evitar conflitos/downgrade:
```bash
sudo apt purge 'nvidia*'
sudo apt autoremove --purge -y

# (Opcional) Segure metapacotes do Ubuntu para evitar reinstalação indesejada
sudo apt-mark hold libnvidia-compute-580 libxnvctrl0 nvidia-driver-580 nvidia-driver-580-open nvidia-driver-570 ubuntu-drivers-common
```

---

<a id="5"></a>
## 5) **“Unable to find a suitable destination to install 32-bit compatibility libraries”**

**Sintoma**  
Aviso dizendo que o sistema **não está preparado para 32-bit** e as libs 32-bit não serão instaladas.

**Causa**  
Arquitetura `i386` não está habilitada, e/ou libs 32-bit não estão presentes. **Para IA/CUDA não é necessário**.

**Como resolver (se precisar 32-bit para jogos/Wine/Steam):**
```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install libegl1:i386 libgles2:i386
# Se o .run pedir um diretório explicitamente:
# sudo ./NVIDIA-Linux-x86_64-580.82.09.run --compat32-libdir=/usr/lib/i386-linux-gnu
```
> Se você **não usa** apps 32-bit, **pode ignorar** este aviso.

---

<a id="6"></a>
## 6) **GLVND / pkg-config ausentes (“Unable to determine the path to install the libglvnd EGL vendor library config files”)**

**Sintoma**  
O instalador reclama que não consegue determinar o caminho para arquivos de configuração do **libGLVND**.

**Causa**  
Falta **pkg-config** e/ou **libglvnd-dev**.

**Como resolver:**
```bash
sudo apt install pkg-config libglvnd-dev -y
# (opcional) libs EGL/GLES
sudo apt install libegl1 libgles2 -y

# Reexecute o instalador .run
sudo ./NVIDIA-Linux-x86_64-580.82.09.run
# Se ainda pedir caminho, informe:
# --glvnd-egl-config-path=/usr/share/glvnd/egl_vendor.d
```

---

<a id="7"></a>
## 7) **`nvidia-smi` falha: “couldn't communicate with the NVIDIA driver”**

**Sintoma**  
`nvidia-smi` dá erro, e `lsmod | grep nvidia` não mostra módulos carregados.

**Causas comuns**
- Driver errado (ex.: `580-open`) sem compatibilidade, ou conflito com pacotes APT.  
- **Headers do kernel** ausentes para compilar o módulo.  
- Kernel diferente daquele para o qual o módulo foi construído.

**Como resolver (passo a passo):**
```bash
# 1) Checar driver instalado
dpkg -l | grep nvidia-driver

# 2) Garantir headers do kernel atual
uname -r
sudo apt install linux-headers-$(uname -r) -y

# 3) Reinstalar o driver proprietário .run e aceitar DKMS
sudo ./NVIDIA-Linux-x86_64-580.82.09.run

# 4) Atualizar initramfs (boa prática)
sudo update-initramfs -u

# 5) Reiniciar e testar
sudo reboot
nvidia-smi
```

---

<a id="8"></a>
## 8) **Módulo não carrega, “module verification failed” / Secure Boot / Kernel “tainted”**

**Sintoma**  
Logs do kernel mostram algo como:
```
nvidia: module verification failed: signature and/or required key missing - tainting kernel
```
e/ou o kernel aparece “tainted”.

**Causa**  
Com **Secure Boot** habilitado, o kernel pode **recusar módulos não assinados**. O “tainted” é um **aviso** de que há código proprietário — é normal com o driver NVIDIA.

**Como resolver (opções):**

**Opção A — Recomendado para iniciantes:** desativar **Secure Boot** no BIOS/UEFI. Esta é a solução mais simples e direta para resolver o problema de assinatura de módulos.

**Opção B — Assinar o módulo (avançado):** Se você precisa manter o Secure Boot ativo, o caminho é gerar suas próprias chaves e assinar o módulo. Este é um processo mais complexo.
```bash
# Gerar par de chaves para assinar módulos
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=Local NVIDIA Module Signing/"

# Registrar a chave no MOK (pede senha e vai pedir para confirmar no boot)
sudo mokutil --import MOK.der

# Reboot e siga o menu "MOK manager" para enrolar a chave

# Assinar o módulo NVIDIA atual
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 MOK.priv MOK.der $(modinfo -n nvidia)

# Verificar assinatura
modinfo nvidia | grep -i signer
```
> Com **DKMS** habilitado, é possível automatizar a assinatura pós-build (fora do escopo do básico).

---

<a id="9"></a>
## 9) **`prime-run` não existe**

**Sintoma**  
`prime-run: command not found`

**Causa**  
O wrapper `prime-run` é fornecido pelos pacotes do Ubuntu. O `.run` da NVIDIA não cria esse atalho.

**Como resolver (usar variáveis de ambiente):**
```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
```
Crie um **alias** permanente no seu `~/.bashrc`:
```bash
echo "alias prime-run='__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia'" >> ~/.bashrc
source ~/.bashrc
```

---

<a id="10"></a>
## 10) **`glxinfo` não encontrado**

**Sintoma**  
`glxinfo: command not found`

**Causa**  
Ferramentas de teste OpenGL não instaladas.

**Como resolver:**
```bash
sudo apt install mesa-utils -y
glxinfo | grep "OpenGL renderer"
prime-run glxinfo | grep "OpenGL renderer"
```

---

<a id="11"></a>
## 11) **`nvcc` ausente (CUDA Toolkit não instalado)**

**Sintoma**  
`nvcc: command not found` e não consegue compilar `.cu`.

**Causa**  
O driver NVIDIA não inclui o **CUDA Toolkit** (apenas runtime).

**Como resolver (instalar CUDA 13.0):**
```bash
# Adicionar repo
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub
sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /"
sudo apt update

# Instalar toolkit
sudo apt install cuda-toolkit-13-0 -y

# PATH
echo 'export PATH=/usr/local/cuda-13.0/bin${PATH:+:${PATH}}' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-13.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}' >> ~/.bashrc
source ~/.bashrc

# Testar
nvcc --version
```

> **Observação:** `apt-key` é depreciado. Versão “bonita”:
> ```bash
> sudo install -d -m 0755 /etc/apt/keyrings
> curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub | sudo gpg --dearmor -o /etc/apt/keyrings/nvidia-cuda.gpg
> echo "deb [signed-by=/etc/apt/keyrings/nvidia-cuda.gpg] https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /" | sudo tee /etc/apt/sources.list.d/cuda-ubuntu2404.list
> sudo apt update
> ```

---

<a id="12"></a>
## 12) **APT avisa repositório sem i386 (OpenVPN3)**

**Sintoma**  
Após habilitar `i386`, aparece:
```
N: Pulando ... 'openvpn3' ... não tem suporte à arquitetura 'i386'
```

**Causa**  
O repositório não fornece pacotes 32-bit.

**Como resolver (limpar aviso):**
- **Restringir para amd64**:
```bash
# Edite /etc/apt/sources.list.d/openvpn3.list e mude para:
# deb [arch=amd64 signed-by=/etc/apt/keyrings/openvpn.asc] https://packages.openvpn.net/openvpn3/debian noble main
```
- **Ou remover o repositório** se não precisar:
```bash
sudo rm /etc/apt/sources.list.d/openvpn3.list
sudo apt update
```

---

<a id="13"></a>
## 13) **Módulos compilados para kernel errado / headers faltando**

**Sintoma**  
Driver compila, mas não carrega; ou `nvidia-smi` falha. Logs mostram caminhos para outro kernel.

**Causa**  
Diferença entre **kernel em uso** e **headers instalados**; ou você está em **kernel OEM 6.8** e os módulos foram instalados para **6.14** (genérico).

**Como resolver:**
```bash
# Ver o kernel atual
uname -r

# Instalar headers corretos
sudo apt install linux-headers-$(uname -r) -y

# Reinstalar driver .run (aceitar DKMS)
sudo ./NVIDIA-Linux-x86_64-580.82.09.run
sudo update-initramfs -u
sudo reboot
```

---

<a id="14"></a>
## 14) **Kernel OEM vs Genérico — qual usar?**

- **Ubuntu 24.04** usa kernel **6.14 genérico** por padrão.  
- Você instalou e usou **kernel OEM 6.8.0-1032-oem**, que é ótimo para notebooks (drivers e patches extras).  
- **Ambos funcionam**, desde que o **módulo NVIDIA seja compilado contra os headers do kernel ativo**.

**Regra de ouro**: sempre garanta que você tem os **headers exatos** do kernel atual:
```bash
sudo apt install linux-headers-$(uname -r) -y
```

---

<a id="15"></a>
## 15) **“No NVIDIA modules detected in the initramfs”**

**Sintoma**  
O log do instalador mostra que não foram detectados módulos no `initramfs` e que ele **não será reconstruído**.

**Causa**  
Informacional — o driver pode carregar **dinamicamente** no boot. Nem sempre é necessário embutir no initramfs.

**Como resolver (se quiser forçar rebuild):**
```bash
sudo update-initramfs -u
sudo reboot
```

---

<a id="16"></a>
## 16) Escolha: “NVIDIA Proprietary” vs “MIT/GPL (Open Kernel Module)”

Durante a instalação, você precisa escolher qual versão do driver usar.

- **NVIDIA Proprietary → (Recomendado)**
  - Usa o módulo binário 100% fechado da NVIDIA.
  - É o driver completo, com todos os recursos (CUDA, OpenGL, Vulkan, etc.).
  - É a escolha padrão para garantir máxima estabilidade e performance, especialmente para jogos, CUDA e inteligência artificial.

- **MIT/GPL (Open Kernel Module) →**
  - É a versão com o módulo do kernel de código aberto, uma iniciativa da NVIDIA para melhorar a integração com o ecossistema Linux.
  - **Ainda depende de firmware e bibliotecas proprietárias** para funcionar (não é totalmente livre).
  - Pode ser útil para compatibilidade com kernels muito novos ou para cenários específicos, mas geralmente não é a primeira escolha para ambientes de produção ou que exigem máxima estabilidade.

**O que escolher?**

Para casos de uso que envolvem IA, CUDA, ou simplesmente a busca por estabilidade e performance, a recomendação é clara: **escolha “NVIDIA Proprietary”**.

---

<a id="17"></a>
## 17) Checklist final e comandos úteis

### Verificar GPU e driver
```bash
nvidia-smi
lsmod | grep nvidia
dmesg | grep -i nvrm
```

### Testar renderização
```bash
sudo apt install mesa-utils vulkan-tools -y
glxinfo | grep "OpenGL renderer"
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
prime-run vulkaninfo | grep "GPU id"
```

### Alternar GPU padrão (permanente)
```bash
sudo prime-select nvidia   # performance
sudo prime-select intel    # economia
prime-select query
```

### Alias `prime-run` (se instalou via .run)
```bash
echo "alias prime-run='__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia'" >> ~/.bashrc
source ~/.bashrc
```

### CUDA Toolkit
```bash
nvcc --version
# Compilar teste:
nvcc cuda_test.cu -o cuda_test && ./cuda_test
```

---

Este guia reúne erros comuns e um fluxo de diagnóstico para manter um **Ideapad + GTX 1650** com **gráficos híbridos** e **CUDA** funcionando.

---

## Referências (fontes para consulta)

### NVIDIA / Ubuntu (base)

- CUDA Installation Guide for Linux: https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html
- PRIME Render Offload (NVIDIA driver README): https://download.nvidia.com/XFree86/Linux-x86_64/550.54.14/README/primerenderoffload.html
- NVIDIA / prime-select (Ubuntu Community Help): https://help.ubuntu.com/community/BinaryDriverHowto/Nvidia

### Manpages (referência)

- `dkms(8)`: https://manpages.debian.org/bookworm/dkms/dkms.8.en.html
- `update-initramfs(8)`: https://manpages.debian.org/bookworm/initramfs-tools/update-initramfs.8.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
