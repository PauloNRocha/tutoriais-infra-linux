# Debian (12/13) e Proxmox: fixar a frequência da CPU no máximo ou para latência estável

*Criado em 04 de dezembro de 2025 e atualizado em 16 de fevereiro de 2026*

Para extrair o máximo de desempenho de um processador, especialmente em tarefas que exigem muito da CPU, é possível travar a frequência de todos os núcleos no valor máximo. Este guia apresenta dois métodos para fazer isso no Debian 12: o clássico com `cpufrequtils` e o moderno com serviço `systemd`.

> [!CAUTION]
> Forçar a frequência máxima aumenta consumo de energia e geração de calor. Monitore temperatura da CPU. Este procedimento é mais indicado para servidores ou desktops com boa refrigeração.

---

## Índice rápido
1. [Pré-requisitos e Verificação](#1)
2. [Método 1: Usando `cpufrequtils` (Clássico)](#2)
3. [Método 2: Criando um Serviço `systemd` (Moderno)](#3)
4. [Como Verificar se Funcionou](#4)
5. [Como Reverter as Alterações](#5)

---

<a id="1"></a>
## 1. Pré-requisitos e Verificação

Antes de tudo, instale os pacotes necessários e verifique quais frequências a sua CPU suporta.

```bash
sudo apt update
sudo apt install linux-cpupower cpufrequtils lm-sensors -y
```

Inspecione a CPU:

```bash
cpupower frequency-info
```

A saída mostra os governadores disponíveis e as frequências suportadas. O foco deste guia é usar o governador `performance` e, quando necessário, fixar também frequência mínima/máxima.

---

<a id="2"></a>
## 2. Método 1: Usando `cpufrequtils` (Clássico)

Este método usa o serviço já preparado para governadores de frequência.

### 2.1 Definindo o Governador

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

---

<a id="3"></a>
## 3. Método 2: Criando um Serviço `systemd` (Moderno)

Este método costuma ser mais direto porque usa um serviço dedicado para aplicar exatamente o que você definiu no boot.

### 3.1 Criando o arquivo do serviço

```bash
sudo nano /etc/systemd/system/cpufreq-max.service
```

Cole o conteúdo abaixo. Ajuste `-d` e `-u` para o valor máximo real do seu processador, conforme `cpupower frequency-info`.

```ini
[Unit]
Description=Fixa a CPU na frequencia maxima ao iniciar

[Service]
Type=oneshot
# Ajuste para a frequencia maxima real da sua CPU (exemplo)
ExecStart=/usr/bin/cpupower frequency-set -g performance -d 3.4GHz -u 3.4GHz
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

### 3.2 Ativando o serviço

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cpufreq-max.service
```

### 3.3 Ajuste adicional para servidores (Proxmox / latência estável)

Em alguns hardwares, apenas definir o governador não impede quedas de frequência em idle. Para manter latência previsível, desative C-States profundos no boot.

Edite o GRUB:

```bash
sudo nano /etc/default/grub
```

Altere para:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet processor.max_cstate=1 idle=nomwait"
```

O que cada parâmetro faz:

- `processor.max_cstate=1`: impede entrada em C-States profundos, reduzindo latência para o core voltar a processar.
- `idle=nomwait`: desativa mecanismo de idle que tende a derrubar clock em períodos curtos sem carga.

Na prática, isso mantém a CPU mais pronta para carga, com menor oscilação de frequência e latência.

Aplicando a alteração:

Debian/Ubuntu padrão:

```bash
sudo update-grub
sudo reboot
```

Se o sistema for Proxmox:

```bash
sudo proxmox-boot-tool refresh
sudo reboot
```

### 3.4 Troubleshooting (problemas comuns)

#### Frequência não fica fixa (cai para ~1400 MHz em idle)

Causa comum: C-States profundos ainda ativos.

```bash
cat /proc/cmdline
```

Deve conter `processor.max_cstate=1 idle=nomwait`. Se não tiver, refaça a etapa do GRUB e reinicie.

#### Serviço ativo mas não aplica no boot

```bash
systemctl status cpufreq-max.service
journalctl -u cpufreq-max.service -b
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
sudo proxmox-boot-tool refresh
sudo reboot
```

---

<a id="4"></a>
## 4. Como Verificar se Funcionou

Depois de aplicar um dos métodos, valide frequência e temperatura.

### Monitorar frequência

```bash
watch -n1 "grep 'cpu MHz' /proc/cpuinfo"
```

As frequências devem permanecer próximas do valor configurado.

### Monitorar temperatura

```bash
sensors
```

Se necessário, rode `sudo sensors-detect` na primeira vez.

---

<a id="5"></a>
## 5. Como Reverter as Alterações

### Para o Método 1 (`cpufrequtils`)

1. Edite:

```bash
sudo nano /etc/default/cpufrequtils
```

Comente `GOVERNOR="performance"` ou altere para `ondemand`/`schedutil`.

2. Reinicie ou desabilite:

```bash
sudo systemctl restart cpufrequtils
```

ou

```bash
sudo systemctl disable --now cpufrequtils
```

### Para o Método 2 (`systemd` customizado)

1. Desative e remova o serviço:

```bash
sudo systemctl disable --now cpufreq-max.service
sudo rm /etc/systemd/system/cpufreq-max.service
sudo systemctl daemon-reload
```

2. Defina governador padrão:

```bash
sudo cpupower frequency-set -g ondemand
```

Isso restaura o comportamento padrão de gerenciamento de frequência da CPU.

---

## Referências (fontes para consulta)

- CPU Performance Scaling (Kernel docs): https://www.kernel.org/doc/html/latest/admin-guide/pm/cpufreq.html
- `cpupower(1)`: https://manpages.debian.org/bookworm/linux-cpupower/cpupower.1.en.html
- `systemd.service(5)`: https://manpages.debian.org/bookworm/systemd/systemd.service.5.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
