# Guia de Produção: Troubleshooting NVIDIA + CUDA no Ubuntu 24.04

*Criado em: 29 de setembro de 2025*  
*Última atualização em: 10 de março de 2026*

Montei este guia para deixar registrado o caminho que costumo seguir quando algo quebra no Ubuntu 24.04 com **driver NVIDIA** e **CUDA**.  
A ideia aqui é ir direto ao ponto: identificar se o problema está em **módulo do kernel**, **Secure Boot**, **headers/DKMS**, **conflito entre métodos de instalação** ou ausência do **CUDA Toolkit**.

Esse troubleshooting foi usado, por exemplo, em um notebook com GPU híbrida e NVIDIA GTX 1650. As versões citadas de driver, CUDA e kernel são **exemplos**; ajuste para o seu ambiente.

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
12. [Módulos compilados para kernel errado / headers faltando](#12)
13. [OEM vs kernel genérico: o que usar?](#13)
14. [“No NVIDIA modules detected in initramfs”](#14)
15. [Escolha “NVIDIA Proprietary” vs “MIT/GPL”](#15)
16. [Checklist final e comandos úteis](#16)

---

<a id="1"></a>
## 1) Fluxo de diagnóstico rápido

Quando algo dá errado com o driver da NVIDIA, comece por estes 5 passos antes de mergulhar nos erros específicos.

1) O driver está carregado?
    ```bash
    nvidia-smi
    ```
    Exemplo genérico de saída saudável:
    ```text
    Fri Sep 29 10:00:00 2025
    +--------------------------------------------------------------+
    | NVIDIA-SMI 580.95.05   Driver Version: 580.95.05             |
    | CUDA Version: 13.0                                           |
    +--------------------------------------------------------------+
    | GPU  Name              Bus-Id               Memory-Usage     |
    | 0    NVIDIA GPU        00000000:01:00.0     512MiB / 4096MiB |
    +--------------------------------------------------------------+
    ```
    Se funcionar, o driver está carregado. Se falhar com “couldn't communicate”, o módulo não subiu.

2) O que os logs do kernel dizem?
    ```bash
    sudo dmesg -T | grep -iE 'nvidia|nouveau|secure boot' | tail -n 20
    ```
    Procure por “module verification failed”, “signature and/or required key missing” (Secure Boot) ou menções ao `nouveau` (conflito).

3) O Secure Boot está ativo?
    ```bash
    mokutil --sb-state
    ```
    Se a resposta for `SecureBoot enabled`, isso é um forte candidato. Desative na BIOS/UEFI ou siga a seção [8](#8) (assinatura).

4) O `nouveau` está realmente desativado?
    ```bash
    lsmod | grep nouveau
    ```
    Se retornar algo, o `nouveau` ainda está ativo e pode conflitar. Revise `/etc/modprobe.d/blacklist-nouveau.conf`.
    Exemplo mínimo esperado do arquivo:
    ```bash
    cat /etc/modprobe.d/blacklist-nouveau.conf
    blacklist nouveau
    options nouveau modeset=0
    ```

5) Você tem os `headers` corretos para o seu kernel?
    ```bash
    uname -r
    sudo apt install linux-headers-$(uname -r)
    ```
    Se o `apt` instalar algo, é provável que essa fosse a causa. Depois, reinstale o driver (se aplicável) e reinicie.

---

<a id="2"></a>
## 2) `ubuntu-drivers autoinstall` tenta trocar a versão do driver

Sintoma  
Ao rodar `ubuntu-drivers autoinstall`, o sistema tenta **remover** a série **580** e **instalar** a **570**.

Causa  
O `ubuntu-drivers` escolhe o **driver recomendado “estável”** pela Canonical (geralmente a série 570), mesmo que você já esteja numa versão mais nova (580).

Como resolver se você quer manter uma versão específica.

Se você decidiu usar driver via .run, evite misturar com autoinstall/metapacotes do Ubuntu.

O "hold" abaixo é opcional e deve ser usado com cuidado: revise depois com `apt-mark showhold`.
```bash
sudo apt-mark hold libnvidia-compute-580 libxnvctrl0 nvidia-driver-580 nvidia-driver-580-open nvidia-driver-570 ubuntu-drivers-common
```
Observação: se o seu objetivo é estabilidade, escolha um método e mantenha consistência (APT *ou* `.run`).

---

<a id="3"></a>
## 3) “You appear to be running an X server”

Sintoma  
Durante o `.run`, aparece aviso pedindo para **não instalar com a interface gráfica ativa**.

Causa  
O instalador precisa parar o **Xorg/Wayland** para compilar/configurar módulos com segurança.

Como resolver

Entrar em TTY:
```bash
Ctrl + Alt + F3
```

Parar o display manager (ajuste ao seu caso):
```bash
sudo systemctl stop gdm
# ou: sudo systemctl stop sddm
# ou: sudo systemctl stop lightdm
```

Executar o instalador:
```bash
sudo ./NVIDIA-Linux-x86_64-580.82.09.run
```

No fim, reinicie:
```bash
sudo reboot
```

---

<a id="4"></a>
## 4) “An alternate method of installing the NVIDIA driver was detected”

Sintoma  
O `.run` detecta que já existe um **driver instalado via pacotes do Ubuntu** e sugere usar o método do sistema.

Causa  
Coexistência de dois métodos (APT vs `.run`) pode gerar conflitos.

Como resolver se você vai ficar com o `.run`
- No prompt, escolha **Continue installation**.  
- Depois da instalação bem-sucedida, remova vestígios dos pacotes APT para reduzir conflito/downgrade.

Importante: este comando é agressivo. Em ambiente desktop, revise o que será removido antes de confirmar.
```bash
sudo apt purge 'nvidia*'
sudo apt autoremove --purge -y
```

(Opcional) “hold” para evitar reinstalação indesejada via APT
```bash
sudo apt-mark hold libnvidia-compute-580 libxnvctrl0 nvidia-driver-580 nvidia-driver-580-open nvidia-driver-570 ubuntu-drivers-common
```

---

<a id="5"></a>
## 5) “Unable to find a suitable destination to install 32-bit compatibility libraries”

Sintoma  
Aviso dizendo que o sistema **não está preparado para 32-bit** e as libs 32-bit não serão instaladas.

Causa  
Arquitetura `i386` não está habilitada, e/ou libs 32-bit não estão presentes. **Para IA/CUDA não é necessário**.

Como resolver se você realmente precisa de 32-bit para jogos/Wine/Steam
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
## 6) GLVND / pkg-config ausentes

Sintoma  
O instalador reclama que não consegue determinar o caminho para arquivos de configuração do **libGLVND**.

Causa  
Falta **pkg-config** e/ou **libglvnd-dev**.

Como resolver
```bash
sudo apt install pkg-config libglvnd-dev -y
# (opcional) libs EGL/GLES
sudo apt install libegl1 libgles2 -y
```

Reexecute o instalador .run
```bash
sudo ./NVIDIA-Linux-x86_64-580.82.09.run
# Se ainda pedir caminho, informe:
# --glvnd-egl-config-path=/usr/share/glvnd/egl_vendor.d
```

---

<a id="7"></a>
## 7) `nvidia-smi` falha: “couldn't communicate with the NVIDIA driver”

Sintoma  
`nvidia-smi` dá erro, e `lsmod | grep nvidia` não mostra módulos carregados.

Causas comuns
- Driver errado (ex.: `580-open`) sem compatibilidade, ou conflito com pacotes APT.  
- **Headers do kernel** ausentes para compilar o módulo.  
- Kernel diferente daquele para o qual o módulo foi construído.

Como resolver (passo a passo)

1. Veja qual driver aparece instalado:

```bash
dpkg -l | grep nvidia-driver
```

2. Garanta os `headers` do kernel atual:

```bash
uname -r
sudo apt install linux-headers-$(uname -r) -y
```

3. Reinstale o driver `.run` e aceite DKMS:

```bash
sudo ./NVIDIA-Linux-x86_64-580.82.09.run
```

4. Atualize o `initramfs`:

```bash
sudo update-initramfs -u
```

5. Reinicie:

```bash
sudo reboot
```

6. Depois do boot, teste novamente:

```bash
nvidia-smi
```

---

<a id="8"></a>
## 8) Módulo não carrega: Secure Boot / assinatura / kernel “tainted”

Sintoma  
Logs do kernel mostram algo como:
```
nvidia: module verification failed: signature and/or required key missing - tainting kernel
```
e/ou o kernel aparece “tainted”.

Causa  
Com **Secure Boot** habilitado, o kernel pode **recusar módulos não assinados**. O “tainted” é um **aviso** de que há código proprietário, é normal com o driver NVIDIA.

Como resolver (opções)

**Opção A — Recomendado para iniciantes:** desativar **Secure Boot** no BIOS/UEFI. Esta é a solução mais simples e direta para resolver o problema de assinatura de módulos.

**Opção B — Assinar o módulo (avançado):** Se você precisa manter o Secure Boot ativo, o caminho é gerar suas próprias chaves e assinar o módulo. Este é um processo mais complexo.

1. Gere o par de chaves:

```bash
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=Local NVIDIA Module Signing/"
```

2. Registre a chave no MOK:

```bash
sudo mokutil --import MOK.der
```

3. Reinicie e siga o menu `MOK manager` para fazer o enrol da chave.

4. Assine o módulo NVIDIA atual:

```bash
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 MOK.priv MOK.der $(modinfo -n nvidia)
```

5. Verifique se a assinatura aparece:

```bash
modinfo nvidia | grep -i signer
```
> Com **DKMS** habilitado, é possível automatizar a assinatura pós-build (fora do escopo do básico).

---

<a id="9"></a>
## 9) `prime-run` não existe

Sintoma  
`prime-run: command not found`

Causa provável  
O wrapper `prime-run` é fornecido por pacotes do Ubuntu (por exemplo, `nvidia-prime`). O instalador `.run` da NVIDIA não cria esse atalho.

Como resolver (usar variáveis de ambiente)

1. Teste manualmente a variável de offload:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
```

2. Se fizer sentido para o seu uso, crie um `alias` permanente no `~/.bashrc`:

```bash
echo "alias prime-run='__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia'" >> ~/.bashrc
```

3. Recarregue o shell:

```bash
source ~/.bashrc
```

---

<a id="10"></a>
## 10) `glxinfo` não encontrado

Sintoma  
`glxinfo: command not found`

Causa provável  
Ferramentas de teste OpenGL não instaladas.

Como resolver

1. Instale o pacote:

```bash
sudo apt install mesa-utils -y
```

2. Teste o renderer padrão:

```bash
glxinfo | grep "OpenGL renderer"
```

3. Teste também com `prime-run`:

```bash
prime-run glxinfo | grep "OpenGL renderer"
```

---

<a id="11"></a>
## 11) `nvcc` ausente (CUDA Toolkit não instalado)

Sintoma  
`nvcc: command not found` e não consegue compilar `.cu`.

Causa provável  
O driver NVIDIA não inclui o **CUDA Toolkit** (apenas runtime).

Como resolver (instalar CUDA Toolkit via repositório NVIDIA)

Observação: a versão exata do Toolkit muda com o tempo. Se você precisa de uma versão específica, valide no repositório e na documentação oficial.

1. Baixe o arquivo de pin do repositório:

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
```

2. Mova o arquivo para `preferences.d`:

```bash
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
```

3. Crie o diretório de keyrings:

```bash
sudo install -d -m 0755 /etc/apt/keyrings
```

4. Importe a chave:

```bash
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub | sudo gpg --dearmor -o /etc/apt/keyrings/nvidia-cuda.gpg
```

5. Adicione o repositório:

```bash
echo "deb [signed-by=/etc/apt/keyrings/nvidia-cuda.gpg] https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /" | sudo tee /etc/apt/sources.list.d/cuda-ubuntu2404.list
```

6. Atualize a lista de pacotes:

```bash
sudo apt update
```

7. Instale o toolkit:

```bash
sudo apt install cuda-toolkit -y
```

8. Ajuste o `PATH`, se necessário:

```bash
echo 'export PATH=/usr/local/cuda/bin${PATH:+:${PATH}}' >> ~/.bashrc
```

9. Ajuste o `LD_LIBRARY_PATH`, se necessário:

```bash
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}' >> ~/.bashrc
```

10. Recarregue o shell:

```bash
source ~/.bashrc
```

11. Teste:

```bash
nvcc --version
```
---

<a id="12"></a>
## 12) Módulos compilados para kernel errado / headers faltando

Sintoma  
Driver compila, mas não carrega; ou `nvidia-smi` falha. Logs mostram caminhos para outro kernel.

Causa provável  
Diferença entre **kernel em uso** e **headers instalados**. Isso acontece com frequência após update de kernel: você reinicia, muda o `uname -r`, mas os headers/DKMS ainda não estão alinhados com o kernel atual.

Como resolver

1. Veja o kernel atual:

```bash
uname -r
```

2. Instale os `headers` corretos:

```bash
sudo apt install linux-headers-$(uname -r) -y
```

3. Reinstale o driver `.run`, atualize o `initramfs` e reinicie:

```bash
sudo ./NVIDIA-Linux-x86_64-580.82.09.run
```

4. Atualize o `initramfs`:

```bash
sudo update-initramfs -u
```

5. Reinicie:

```bash
sudo reboot
```

---

<a id="13"></a>
## 13) Kernel OEM vs kernel genérico — o que importa

No Ubuntu 24.04, a versão exata do kernel pode variar (GA/HWE/OEM). O que importa para o driver NVIDIA é simples:

- o módulo precisa ser compilado para o **kernel que está rodando agora**;
- os `headers` precisam ser do **mesmo** kernel (`uname -r`).

**Regra de ouro**: sempre garanta que você tem os **headers exatos** do kernel atual:
```bash
sudo apt install linux-headers-$(uname -r) -y
```

---

<a id="14"></a>
## 14) “No NVIDIA modules detected in the initramfs”

Sintoma  
O log do instalador mostra que não foram detectados módulos no `initramfs` e que ele **não será reconstruído**.

Causa provável  
Informacional — o driver pode carregar **dinamicamente** no boot. Nem sempre é necessário embutir no initramfs.

Como resolver (se quiser forçar rebuild)

1. Atualize o `initramfs`:

```bash
sudo update-initramfs -u
```

2. Reinicie:

```bash
sudo reboot
```

---

<a id="15"></a>
## 15) Escolha: “NVIDIA Proprietary” vs “MIT/GPL (Open Kernel Module)”

Durante a instalação, você precisa escolher qual “linha” de driver usar.

- **NVIDIA Proprietary (ponto de partida mais previsível)**
  - Módulo do kernel proprietário, mantido pela NVIDIA.
  - Em geral é o caminho com menos surpresas quando o objetivo é estabilidade.

- **Open Kernel Module (MIT/GPL)**
  - Módulo do kernel aberto, mas ainda com partes proprietárias (firmware/bibliotecas).
  - Pode ser útil em cenários específicos, mas não é “automaticamente melhor” para todo mundo.

Recomendação prática  
Comece com **NVIDIA Proprietary**. Considere o Open Kernel Module apenas se você tiver um motivo claro e já tiver validado com o seu kernel/hardware. E mantenha o método consistente: não misture APT e `.run` no mesmo host.

---

<a id="16"></a>
## 16) Checklist final e comandos úteis

### Verificar GPU e driver
```bash
nvidia-smi
lsmod | grep nvidia
dmesg | grep -i nvrm
```

### Testar renderização

1. Instale os utilitários:

```bash
sudo apt install mesa-utils vulkan-tools -y
```

2. Teste OpenGL:

```bash
glxinfo | grep "OpenGL renderer"
```

3. Teste OpenGL com offload NVIDIA:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
```

4. Teste Vulkan com `prime-run`:

```bash
prime-run vulkaninfo | grep "GPU id"
```

### Alternar GPU padrão (permanente)

Observação: `prime-select` e `prime-run` são do ecossistema Ubuntu (pacotes como `nvidia-prime`). Se você instalou tudo via `.run`, isso pode não existir.

1. Para performance:

```bash
sudo prime-select nvidia   # performance
```

2. Para economia:

```bash
sudo prime-select intel    # economia
```

3. Para conferir o perfil atual:

```bash
prime-select query
```

### Alias `prime-run` (se instalou via .run)

1. Crie o `alias`:

```bash
echo "alias prime-run='__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia'" >> ~/.bashrc
```

2. Recarregue o shell:

```bash
source ~/.bashrc
```

### CUDA Toolkit
```bash
nvcc --version
# Compilar teste:
nvcc cuda_test.cu -o cuda_test && ./cuda_test
```

---

Resolvi juntar esse material porque esse tipo de problema costuma consumir tempo demais quando a gente tenta lembrar tudo de cabeça. Então preferi deixar um roteiro de consulta rápida, com os erros mais comuns e o que normalmente resolve cada um deles no Ubuntu 24.04 com NVIDIA + CUDA.

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

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
