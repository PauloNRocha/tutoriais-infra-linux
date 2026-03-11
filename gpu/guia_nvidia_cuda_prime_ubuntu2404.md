# NVIDIA + CUDA + PRIME no Ubuntu 24.04 — instalação e configuração (GTX 1650)

*Criado em: 25 de setembro de 2025*  
*Última atualização em: 11 de março de 2026*

Montei este guia para deixar registrado o processo que funcionou aqui para instalar o **driver proprietário NVIDIA**, habilitar o **CUDA Toolkit** e configurar a **renderização híbrida (Intel + NVIDIA)** no Ubuntu 24.04.

O cenário principal foi notebook **Lenovo IdeaPad com GPU NVIDIA GTX 1650 Mobile**, mas boa parte do fluxo também serve como referência para outros casos parecidos.

> **Ambiente testado:** Ubuntu 24.04 LTS (Noble Numbat).

---

## Índice rápido
1. [Preparação Inicial](#1)
2. [Remover drivers antigos e o `nouveau`](#2)
3. [Instalar dependências para compilar o driver](#3)
4. [Instalar o Driver NVIDIA Proprietário `.run`](#4)
5. [Validar instalação do driver e proteger contra `apt`](#5)
6. [Instalar CUDA Toolkit](#6)
7. [Configuração gráfica híbrida (Intel + NVIDIA)](#7)
8. [Solução de Problemas](#8)

---

<a id="1"></a>
## 1. Preparação Inicial

### Verificar a GPU instalada
```bash
lspci | grep -i nvidia
```
Saída esperada:
```
01:00.0 VGA compatible controller: NVIDIA Corporation TU117M [GeForce GTX 1650 Mobile / Max-Q] (rev a1)
01:00.1 Audio device: NVIDIA Corporation Device 10fa (rev a1)
```

### Atualizar o sistema

1. Atualize a lista de pacotes:

```bash
sudo apt update
```

2. Faça o upgrade:

```bash
sudo apt upgrade -y
```

---

<a id="2"></a>
## 2. Remover drivers antigos e o `nouveau`

O `nouveau` é o driver open-source da NVIDIA e precisa ser desativado para evitar conflitos.

1. Remova drivers NVIDIA antigos:

```bash
sudo apt purge 'nvidia*'
sudo apt autoremove --purge -y
```

2. Crie o arquivo de blacklist do `nouveau`:

```bash
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
```

3. Atualize o `initramfs`:

```bash
sudo update-initramfs -u
```

4. Reinicie o sistema:
```bash
sudo reboot
```

---

<a id="3"></a>
## 3. Instalar dependências para compilar o driver

1. Instale as dependências:

```bash
sudo apt install build-essential gcc make dkms linux-headers-$(uname -r) -y
```

---

<a id="4"></a>
## 4. Instalar o Driver NVIDIA Proprietário `.run`

> ### Por que usar o `.run` em vez do driver do Ubuntu?
>
> **Vantagens:**
>
> - **Versão mais recente:** Você obtém o driver mais novo diretamente da NVIDIA (ex: 580.82.09), com as últimas otimizações e recursos.
> - **Suporte completo:** Garante o melhor suporte para CUDA, OpenGL, Vulkan e frameworks de IA (TensorFlow, PyTorch).
> - **Independência:** Não fica preso à versão que o `ubuntu-drivers` considera "recomendada", que geralmente é mais antiga.
>
> **Desvantagens:**
>
> - **Manutenção manual:** Pode ocorrer de quando o kernel do Linux for atualizado, você precisar reinstalar o driver `.run` manualmente.
> - **Mais trabalho:** Exige um pouco mais de atenção do que instalar um pacote `.deb` via `apt`, que é gerenciado automaticamente pelo DKMS.

1. Baixe o driver do site da NVIDIA:  
   [Drivers NVIDIA](https://www.nvidia.com/Download/index.aspx)

2. Dê permissão de execução ao arquivo:
   ```bash
   chmod +x NVIDIA-Linux-x86_64-580.82.09.run
   ```

3. Pare a interface gráfica:
   ```bash
   sudo systemctl stop gdm   # ou lightdm/sddm, dependendo do sistema
   ```

4. Execute o instalador:
   ```bash
   sudo ./NVIDIA-Linux-x86_64-580.82.09.run
   ```

5. Durante a instalação:
   - **Escolha o tipo de driver:**
     - **`NVIDIA Proprietary`**: aqui é o caminho mais previsível para usar CUDA, OpenGL, Vulkan, IA e notebook híbrido sem inventar moda.
     - **`MIT/GPL (Open Kernel Module)`**: usa o módulo aberto da NVIDIA no kernel, mas ainda depende de partes proprietárias em outros pontos da pilha. Pode funcionar bem em alguns cenários, mas eu não usaria como primeira escolha aqui.
     - **Para este guia, eu ficaria com `NVIDIA Proprietary`.**
   - Responda **Yes** para **DKMS**.  
   - Ignore aviso sobre bibliotecas **32-bit** se você não usa jogos, Wine ou software legado 32-bit.  
   - Responda **No** no `nvidia-xconfig` (mantém Intel como padrão e NVIDIA sob demanda).

6. Reinicie:
   ```bash
   sudo reboot
   ```

---

<a id="5"></a>
## 5. Validar instalação do driver e proteger contra `apt`

### Testar driver e CUDA runtime

1. Verifique o driver:

```bash
nvidia-smi
```
Saída esperada (exemplo):
```
NVIDIA-SMI 580.82.09   Driver Version: 580.82.09   CUDA Version: 13.0
GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC
0  NVIDIA GeForce GTX 1650        Off | 00000000:01:00.0 Off | N/A
```

### Confirmar se módulo está carregado

2. Verifique se os módulos estão ativos:

```bash
lsmod | grep nvidia
```

### (Opcional, mas recomendado) Proteger o driver contra `apt`
Para evitar que o `apt` ou o `ubuntu-drivers` tentem fazer o "downgrade" ou substituir seu driver instalado manualmente, bloqueie os pacotes do repositório:

3. Se quiser evitar troca automática de versão via `apt`, aplique o `hold`:

```bash
sudo apt-mark hold libnvidia-compute-580 libxnvctrl0 nvidia-driver-580 nvidia-driver-580-open nvidia-driver-570 ubuntu-drivers-common
```

---

<a id="6"></a>
## 6. Instalar CUDA Toolkit

### Adicionar repositório oficial NVIDIA

1. Baixe o arquivo de pin:

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

4. Importe a chave do repositório:

```bash
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub | sudo gpg --dearmor -o /etc/apt/keyrings/nvidia-cuda.gpg
```

5. Adicione o repositório:

```bash
echo "deb [signed-by=/etc/apt/keyrings/nvidia-cuda.gpg] https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /" | sudo tee /etc/apt/sources.list.d/cuda-ubuntu2404.list
```

6. Atualize a lista:

```bash
sudo apt update
```

### Instalar CUDA Toolkit 13.0

1. Instale o toolkit:

```bash
sudo apt install cuda-toolkit-13-0 -y
```

### Configurar PATH (no `~/.bashrc` ou equivalente)

1. Ajuste o `PATH`:

```bash
echo 'export PATH=/usr/local/cuda-13.0/bin${PATH:+:${PATH}}' >> ~/.bashrc
```

2. Ajuste o `LD_LIBRARY_PATH`:

```bash
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-13.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}' >> ~/.bashrc
```

3. Recarregue o shell:
```bash
source ~/.bashrc
```

### Validar `nvcc`
```bash
nvcc --version
```

### Teste CUDA simples
```c
#include <stdio.h>
__global__ void helloCUDA() {
    printf("Hello from GPU!\n");
}
int main() {
    helloCUDA<<<1,1>>>();
    cudaDeviceSynchronize();
    return 0;
}
```
Compile e rode:
```bash
nvcc cuda_test.cu -o cuda_test
./cuda_test
```
Saída esperada:
```
Hello from GPU!
```

---

<a id="7"></a>
## 7. Configuração gráfica híbrida (Intel + NVIDIA)

### Instalar utilitários gráficos

1. Instale os utilitários:

```bash
sudo apt install mesa-utils vulkan-tools -y
```

### Testar renderização Intel

2. Teste a renderização Intel:

```bash
glxinfo | grep "OpenGL renderer"
```
Exemplo de saída:
```
OpenGL renderer string: Mesa Intel(R) UHD Graphics
```

### Testar renderização NVIDIA sob demanda

3. Teste a renderização NVIDIA sob demanda:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep "OpenGL renderer"
```
Exemplo:
```
OpenGL renderer string: NVIDIA GeForce GTX 1650/PCIe/SSE2
```

### Criar alias `prime-run`
1. Adicione no `~/.bashrc`:
```bash
alias prime-run='__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia'
```

2. Recarregue:
```bash
source ~/.bashrc
```

Agora você pode usar:

```bash
prime-run glxgears
prime-run vulkaninfo | grep "GPU id"
```

### Alternar GPUs permanentemente

4. Para forçar NVIDIA sempre:
  ```bash
  sudo prime-select nvidia
  ```

5. Para voltar para Intel:
  ```bash
  sudo prime-select intel
  ```

6. Para conferir o perfil atual:
  ```bash
  prime-select query
  ```

---

<a id="8"></a>
## 8. Solução de Problemas
Se ocorrer algum erro durante ou após a instalação (tela preta, falha ao carregar o driver, `nvidia-smi` não funcionando), use o guia de troubleshooting abaixo.

**Consulte o [Guia de Solução de Problemas NVIDIA + CUDA](guia_nvidia_cuda_troubleshooting_ubuntu2404.md) para um passo a passo detalhado.**

---

## Referências (fontes para consulta)

### NVIDIA (oficial)

- CUDA Installation Guide for Linux: https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html
- NVIDIA Linux Driver README (instalação do driver): https://download.nvidia.com/XFree86/Linux-x86_64/580.82.09/README/installdriver.html
- PRIME Render Offload (NVIDIA driver README): https://download.nvidia.com/XFree86/Linux-x86_64/550.54.14/README/primerenderoffload.html

### Ubuntu (referência)

- NVIDIA / prime-select (Ubuntu Community Help): https://help.ubuntu.com/community/BinaryDriverHowto/Nvidia

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
