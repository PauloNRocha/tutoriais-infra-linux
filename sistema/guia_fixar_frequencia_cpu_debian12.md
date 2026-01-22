# Debian 12: fixar a frequência da CPU no máximo

*Criado em 04 de dezembro de 2025 e atualizado em 08 de dezembro de 2025*

Para extrair o máximo de desempenho de um processador, especialmente em tarefas que exigem muito da CPU, é possível “travar” a frequência de todos os núcleos no valor máximo. Este guia apresenta dois métodos para fazer isso no Debian 12: o clássico com `cpufrequtils` e o moderno com um serviço `systemd`.

> [!CAUTION]
> Forçar a frequência máxima aumenta consumo de energia e geração de calor. Monitore a temperatura da CPU. Este procedimento é mais indicado para servidores ou desktops com boa refrigeração.

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
Antes de tudo, precisamos instalar os pacotes certos e verificar quais frequências nossa CPU suporta.

```bash
# Instalar os pacotes necessários
sudo apt update
sudo apt install linux-cpupower cpufrequtils lm-sensors -y
```

Agora, vamos inspecionar a CPU:
```bash
cpupower frequency-info
```
A saída vai te mostrar os "governadores" de energia disponíveis e as frequências que a CPU pode atingir. O que nos interessa é o governador `performance` e a frequência máxima listada.

---

<a id="2"></a>
## 2. Método 1: Usando `cpufrequtils` (Clássico)
Este método usa um serviço que já vem preparado para gerenciar isso.

### 2.1 Definindo o Governador
Primeiro, vamos setar o governador para `performance`, que prioriza a frequência máxima.
```bash
sudo cpupower frequency-set -g performance
```

### 2.2 Tornando a Alteração Permanente
Para que a configuração não se perca no boot, editamos o arquivo de configuração do serviço:
```bash
sudo nano /etc/default/cpufrequtils
```
Dentro do arquivo, adicione a seguinte linha:
```
GOVERNOR="performance"
```
Agora, basta ativar o serviço para iniciar com o sistema:
```bash
sudo systemctl enable --now cpufrequtils
```

---

<a id="3"></a>
## 3. Método 2: Criando um Serviço `systemd` (Moderno)
Este método costuma ser mais “limpo” porque usa um serviço próprio que faz exatamente o necessário, sem depender de pacotes extras.

### 3.1 Criando o Arquivo do Serviço
```bash
sudo nano /etc/systemd/system/cpufreq-max.service
```
Cole o seguinte conteúdo no arquivo. Lembre-se de **ajustar a frequência** (`-d` e `-u`) para o valor máximo que você viu no `cpupower frequency-info`.

```ini
[Unit]
Description=Fixa a CPU na frequencia maxima ao iniciar

[Service]
Type=oneshot
# Altere a frequência (ex: 3.4GHz) para a máxima do seu processador
ExecStart=/usr/bin/cpupower frequency-set -g performance -d 3.4GHz -u 3.4GHz
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

### 3.2 Ativando o Serviço
Agora, vamos informar ao `systemd` sobre nosso novo serviço e ativá-lo:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cpufreq-max.service
```

---

<a id="4"></a>
## 4. Como Verificar se Funcionou
Após usar um dos métodos, você pode verificar em tempo real se todos os núcleos estão na frequência máxima.

### Monitorar Frequência
```bash
# O 'watch' atualiza o comando a cada 1 segundo
watch -n1 "grep 'cpu MHz' /proc/cpuinfo"
```
Você verá a frequência de cada núcleo. Elas devem estar travadas no valor máximo.

### Monitorar Temperatura
É uma boa ideia ficar de olho na temperatura para garantir que a refrigeração está dando conta.
```bash
# Pode ser necessário rodar 'sudo sensors-detect' na primeira vez
sensors
```
Com isso, você tem total controle sobre o desempenho do seu processador!

---

<a id="5"></a>
## 5. Como Reverter as Alterações
Se você decidir que não quer mais a CPU na frequência máxima, veja como reverter as configurações para cada método.

### Para o Método 1 (`cpufrequtils`)
1.  **Edite o arquivo de configuração:**
    ```bash
    sudo nano /etc/default/cpufrequtils
    ```
    Comente a linha `GOVERNOR="performance"` (adicione `#` no início) ou altere para `GOVERNOR="ondemand"` (ou outro de sua preferência).

2.  **Reinicie o serviço:**
    ```bash
    sudo systemctl restart cpufrequtils
    ```
    Ou, para desabilitá-lo completamente:
    ```bash
    sudo systemctl disable --now cpufrequtils
    ```

### Para o Método 2 (`systemd` customizado)
1.  **Desative e remova o serviço:**
    ```bash
    sudo systemctl disable --now cpufreq-max.service
    sudo rm /etc/systemd/system/cpufreq-max.service
    sudo systemctl daemon-reload
    ```
2.  **Defina o governador padrão (geralmente `ondemand` ou `schedutil`):**
    ```bash
    sudo cpupower frequency-set -g ondemand
```
    Isso deve restaurar o comportamento padrão de gerenciamento de frequência da CPU.

---

## Referências (fontes para consulta)

### Kernel / CPUFreq

- CPU Performance Scaling (Kernel docs): https://www.kernel.org/doc/html/latest/admin-guide/pm/cpufreq.html

### Ferramentas (manpages Debian)

- `cpupower(1)`: https://manpages.debian.org/bookworm/linux-cpupower/cpupower.1.en.html
- `systemd.service(5)`: https://manpages.debian.org/bookworm/systemd/systemd.service.5.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
