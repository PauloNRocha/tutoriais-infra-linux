# Debian: desabilitar suspensão e hibernação (systemd)

*Criado em 04 de dezembro de 2025 e atualizado em 08 de dezembro de 2025*

Às vezes, especialmente em servidores ou em notebooks com hardware específico, os modos de suspensão e hibernação podem causar mais problemas do que soluções. Este guia mostra formas corretas de desabilitar esses recursos usando `systemd`.

> [!NOTE]
> **Configurações da BIOS/UEFI**: em alguns hardwares, opções de energia na BIOS/UEFI podem sobrepor ou interferir nas configurações do sistema operacional. Se, mesmo após aplicar os passos deste guia, o comportamento persistir, verifique opções como “Deep Sleep” e “ACPI Suspend State” na BIOS/UEFI.

---

## Índice rápido
1. [Método 1: A Forma Rápida e Radical (mask)](#1)
2. [Método 2: Desabilitar Apenas a Suspensão ao Fechar a Tampa do Notebook](#2)
3. [Como Reverter as Alterações](#3)

---

<a id="1"></a>
## 1. Método 1: A Forma Rápida e Radical (mask)

Se você quer desabilitar completamente qualquer tentativa do sistema de suspender ou hibernar, o comando `systemctl mask` é a ferramenta certa. O que ele faz é criar um link simbólico de um serviço para `/dev/null`, efetivamente "apagando" o serviço da visão do `systemd` sem removê-lo.

Execute o seguinte comando para mascarar todos os alvos relacionados a `sleep`:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```
Com isso, as opções de suspender e hibernar desaparecerão dos menus do seu ambiente gráfico e nenhuma ação (como fechar a tampa do notebook) irá acioná-las.

---

<a id="2"></a>
## 2. Método 2: Desabilitar Apenas a Suspensão ao Fechar a Tampa do Notebook

Talvez você não queira desabilitar tudo, mas apenas impedir que o notebook suspenda ao fechar a tampa. Isso é muito comum para quem usa o notebook como um "desktop", conectado a um monitor externo.

A configuração para isso fica no arquivo `logind.conf`.

1.  **Edite o arquivo de configuração:**
    ```bash
    sudo nano /etc/systemd/logind.conf
    ```
2.  **Encontre e altere as seguintes linhas:**
    Procure por estas linhas, que provavelmente estarão comentadas com um `#` no início:
    ```ini
    #HandleLidSwitch=suspend
    #HandleLidSwitchExternalPower=suspend
    #HandleLidSwitchDocked=suspend
    ```
    Remova o `#` do início e mude o valor para `ignore`. Deve ficar assim:
    ```ini
    HandleLidSwitch=ignore
    HandleLidSwitchExternalPower=ignore
    HandleLidSwitchDocked=ignore
    ```
    - `HandleLidSwitch`: Comportamento quando a tampa é fechada (na bateria).
    - `HandleLidSwitchExternalPower`: Comportamento quando a tampa é fechada (conectado na tomada).
    - `HandleLidSwitchDocked`: Comportamento quando a tampa é fechada (conectado a uma docking station).

3.  **Aplique as alterações:**
    Para que as novas configurações tenham efeito, você pode reiniciar o serviço `logind` sem precisar reiniciar o computador.
    ```bash
    sudo systemctl restart systemd-logind.service
    ```
    Se a alteração foi aplicada corretamente, fechar a tampa não fará mais nada.

---

<a id="3"></a>
## 3. Como Reverter as Alterações

Se você se arrepender e quiser os recursos de volta, o processo é igualmente simples.

- **Para o Método 1 (mask):**
  Use o comando `unmask` para reativar os serviços.
  ```bash
  sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
  ```
- **Para o Método 2 (logind.conf):**
  Simplesmente edite o arquivo `/etc/systemd/logind.conf` novamente e comente as linhas que você alterou (coloque o `#` de volta no início) ou restaure o valor original (`suspend`). Depois, reinicie o serviço novamente.

---

## Referências (fontes para consulta)

### systemd (manpages Debian)

- `systemctl(1)`: https://manpages.debian.org/bookworm/systemd/systemctl.1.en.html
- `logind.conf(5)`: https://manpages.debian.org/bookworm/systemd/logind.conf.5.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
