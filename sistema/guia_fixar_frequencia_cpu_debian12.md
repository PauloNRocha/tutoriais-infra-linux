# Guia de Produção: Ajuste de frequência da CPU no Debian e Proxmox

*Criado em: 04 de dezembro de 2025*  
*Última atualização em: 18 de maio de 2026*

Em alguns servidores, principalmente Proxmox, hosts com muita VM ou máquinas sensíveis a latência, deixar a CPU variando agressivamente pode atrapalhar a previsibilidade. Este guia mostra como usar o governador `performance` e, quando fizer sentido, limitar a faixa de frequência com `cpupower` no Debian 12/13 e Proxmox.

> [!CAUTION]
> Forçar perfil de desempenho aumenta consumo de energia e geração de calor. Monitore temperatura, refrigeração e throttling. Em produção, aplique primeiro em janela controlada e tenha rollback pronto.

Importante: em CPUs modernas, “fixar frequência” nem sempre significa ver exatamente o mesmo MHz o tempo todo. O driver de scaling, turbo boost, limites térmicos, firmware/BIOS e C-States podem influenciar o valor real.

---

## Índice rápido
1. [Pré-requisitos e verificação](#1)
2. [Método 1: usando cpufrequtils](#2)
3. [Método 2: serviço systemd com cpupower](#3)
4. [Ajuste adicional para latência estável](#4)
5. [Como verificar se funcionou](#5)
6. [Como reverter as alterações](#6)
7. [Referências](#referencias)

---

<a id="1"></a>
## 1. Pré-requisitos e verificação

Antes de tudo, instale os pacotes necessários e verifique quais frequências a sua CPU suporta.

```bash
sudo apt update
sudo apt install -y linux-cpupower cpufrequtils lm-sensors
```

Inspecione a CPU:

```bash
cpupower frequency-info
```

A saída mostra driver, governadores disponíveis e limites de frequência.

Verifique também diretamente pelo `sysfs`:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```

Se o diretório `/sys/devices/system/cpu/cpu0/cpufreq/` não existir, o sistema não está expondo controle de frequência para essa CPU/VM. Isso pode acontecer em algumas máquinas virtuais.

Se `performance` não aparecer em `scaling_available_governors`, não force o procedimento. Primeiro entenda o driver em uso e as opções disponíveis no hardware.

Escolha apenas um método de persistência: `cpufrequtils` ou serviço `systemd` customizado. Usar os dois ao mesmo tempo normalmente não traz vantagem e deixa o troubleshooting mais confuso.

---

<a id="2"></a>
## 2. Método 1: usando cpufrequtils

Este método usa o serviço já preparado para aplicar o governador de frequência no boot. É simples e suficiente quando você só precisa manter o governador `performance`.

### 2.1 Definindo o governador

```bash
sudo cpupower frequency-set -g performance
```

### 2.2 Tornando a alteração permanente

```bash
sudo nano /etc/default/cpufrequtils
```

Adicione:

```text
GOVERNOR="performance"
```

Ative o serviço:

```bash
sudo systemctl enable --now cpufrequtils
```

Valide:

```bash
cpupower frequency-info | grep -E 'driver|governor|available cpufreq governors'
```

---

<a id="3"></a>
## 3. Método 2: serviço systemd com cpupower

Este método costuma ser mais transparente porque deixa claro qual comando será aplicado no boot. É o caminho que prefiro quando quero controlar exatamente o comportamento e registrar a configuração em um serviço próprio.

### 3.1 Criando o arquivo do serviço

```bash
sudo nano /etc/systemd/system/cpufreq-performance.service
```

Cole o conteúdo abaixo:

```ini
[Unit]
Description=Aplica governador performance na CPU ao iniciar
ConditionPathExists=/usr/bin/cpupower

[Service]
Type=oneshot
ExecStart=/usr/bin/cpupower frequency-set -g performance
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Se você realmente precisa fixar também a frequência mínima e máxima, descubra primeiro os valores suportados:

```bash
cpupower frequency-info
```

Depois ajuste o `ExecStart=` com valores reais do seu processador. Exemplo:

```ini
ExecStart=/usr/bin/cpupower frequency-set -g performance -d 3.4GHz -u 3.4GHz
```

Use esse formato somente se o valor existir para sua CPU. Um valor incorreto pode fazer o serviço falhar no boot.

### 3.2 Ativando o serviço

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cpufreq-performance.service
```

Valide:

```bash
systemctl status cpufreq-performance.service --no-pager
cpupower frequency-info | grep -E 'driver|governor|current CPU frequency|available cpufreq governors'
```

---

<a id="4"></a>
## 4. Ajuste adicional para latência estável

Em alguns hardwares, apenas definir o governador não impede quedas de frequência em idle. Para reduzir variações de latência, você pode limitar C-States profundos no boot.

Use esta etapa apenas se você realmente precisa de latência mais previsível. Ela aumenta consumo, temperatura e pode mudar o comportamento de economia de energia do servidor.

Antes de editar, veja como a linha está hoje:

```bash
grep '^GRUB_CMDLINE_LINUX_DEFAULT=' /etc/default/grub
```

Edite o GRUB:

```bash
sudo nano /etc/default/grub
```

Adicione os parâmetros mantendo o que já existir na linha. Exemplo:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet processor.max_cstate=1 idle=nomwait"
```

Se a sua linha já tiver outros parâmetros importantes, como opções de IOMMU, console serial ou alguma configuração específica do servidor, não apague esses valores. Apenas acrescente `processor.max_cstate=1 idle=nomwait` no final da linha.

O que cada parâmetro faz:

- `processor.max_cstate=1`: impede entrada em C-States profundos, reduzindo latência para o core voltar a processar.
- `idle=nomwait`: no x86, evita o uso da instrução `MWAIT` para estados de idle. Em Intel, isso força o uso do driver `acpi_idle` quando as tabelas ACPI permitem.

Na prática, isso mantém a CPU mais pronta para carga, com menor oscilação de frequência e latência.

Aplicando a alteração em Debian/Ubuntu padrão com GRUB:

```bash
sudo update-grub
sudo reboot
```

Em Proxmox, verifique o modo de boot:

```bash
command -v proxmox-boot-tool
sudo proxmox-boot-tool status
```

Se `command -v proxmox-boot-tool` retornar um caminho e o status mostrar entradas de boot gerenciadas pelo Proxmox, atualize as entradas com:

```bash
sudo proxmox-boot-tool refresh
sudo reboot
```

Se o host usa GRUB tradicional, use:

```bash
sudo update-grub
sudo reboot
```

### 4.1 Troubleshooting

#### Frequência não fica fixa (cai para ~1400 MHz em idle)

Causa comum: C-States profundos ainda ativos.

```bash
cat /proc/cmdline
```

Deve conter `processor.max_cstate=1 idle=nomwait`. Se não tiver, refaça a etapa do GRUB e reinicie.

#### Serviço ativo mas não aplica no boot

```bash
systemctl status cpufreq-performance.service
journalctl -u cpufreq-performance.service -b
```

Se falhar, confirme caminho do binário:

```bash
which cpupower
```

Ajuste o `ExecStart=` se necessário.

#### Frequência oscila durante carga pesada

Causa provável: throttling térmico.

```bash
sensors
```

Se a CPU estiver quente demais, ela reduzirá clock automaticamente. Ajuste refrigeração ou fixe um passo abaixo da frequência máxima.

#### Após reboot voltou ao comportamento antigo (Proxmox)

```bash
command -v proxmox-boot-tool
sudo proxmox-boot-tool status
```

Se `command -v proxmox-boot-tool` retornar um caminho, rode `sudo proxmox-boot-tool refresh`. Se não retornar caminho, o host provavelmente usa GRUB tradicional; nesse caso, rode `sudo update-grub`.

---

<a id="5"></a>
## 5. Como verificar se funcionou

Depois de aplicar um dos métodos, valide frequência e temperatura.

### Monitorar frequência

```bash
watch -n1 "grep 'cpu MHz' /proc/cpuinfo"
```

As frequências devem permanecer próximas do valor configurado.

Verificar o governador por CPU:

```bash
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor | sort -u
```

Verificar limites aplicados:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```

### Monitorar temperatura

```bash
sensors
```

Se necessário, rode `sudo sensors-detect` na primeira vez.

---

<a id="6"></a>
## 6. Como reverter as alterações

### 6.1 Para o método 1 (`cpufrequtils`)

1. Edite:

```bash
sudo nano /etc/default/cpufrequtils
```

Comente `GOVERNOR="performance"` ou altere para um governador disponível no seu sistema, como `schedutil` ou `ondemand`.

2. Reinicie ou desabilite:

```bash
sudo systemctl restart cpufrequtils
```

ou

```bash
sudo systemctl disable --now cpufrequtils
```

### 6.2 Para o método 2 (`systemd` customizado)

1. Desative o serviço e preserve uma cópia do arquivo:

```bash
sudo systemctl disable --now cpufreq-performance.service
sudo mv /etc/systemd/system/cpufreq-performance.service /etc/systemd/system/cpufreq-performance.service.bak
sudo systemctl daemon-reload
```

2. Defina governador padrão:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
sudo cpupower frequency-set -g schedutil
```

Se `schedutil` não existir no seu sistema, use outro governador disponível, como `ondemand` ou `powersave`.

### 6.3 Remover parâmetros de C-State no boot

Se você aplicou `processor.max_cstate=1 idle=nomwait`, remova esses parâmetros de:

```bash
sudo nano /etc/default/grub
```

Depois atualize o bootloader conforme seu ambiente.

Debian/Ubuntu ou Proxmox com GRUB:

```bash
sudo update-grub
sudo reboot
```

Proxmox com `proxmox-boot-tool`:

```bash
sudo proxmox-boot-tool refresh
sudo reboot
```

Após reiniciar, confirme:

```bash
cat /proc/cmdline
```

---

<a id="referencias"></a>
## Referências (fontes para consulta)

- CPU Performance Scaling (Kernel docs): https://www.kernel.org/doc/html/latest/admin-guide/pm/cpufreq.html
- CPU Idle Time Management (Kernel docs): https://docs.kernel.org/admin-guide/pm/cpuidle.html
- `intel_idle` CPU Idle Time Management Driver (Kernel docs): https://www.kernel.org/doc/html/latest/admin-guide/pm/intel_idle.html
- `cpupower-frequency-set(1)` Debian: https://manpages.debian.org/unstable/linux-cpupower/cpupower-frequency-set.1.en.html
- `systemd.service(5)` Debian Trixie: https://manpages.debian.org/trixie/systemd/systemd.service.5.en.html
- Proxmox VE bootloader: https://pve.proxmox.com/wiki/Host_Bootloader

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
