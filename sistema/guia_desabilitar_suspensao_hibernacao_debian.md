# Guia Prático: Desabilitar suspensão e hibernação no Debian com systemd

*Criado em: 04 de dezembro de 2025*  
*Última atualização em: 18 de maio de 2026*

Em servidor, máquina de bancada ou notebook usado como desktop, suspensão e hibernação podem virar problema: serviço para, sessão cai, acesso remoto some e a máquina só volta com intervenção local. Aqui a ideia é deixar registrado um jeito simples de bloquear essas ações pelo `systemd`, sem remover pacotes nem mexer em configuração de energia do ambiente gráfico.

> [!NOTE]
> Em alguns hardwares, opções da BIOS/UEFI também interferem nesse comportamento. Se o sistema continuar suspendendo mesmo depois dos ajustes, verifique opções como `Deep Sleep`, `ACPI Suspend State`, `Wake on Lid Open` ou configurações equivalentes.

---

## Índice rápido

1. [Antes de alterar](#1)
2. [Método 1: bloquear suspensão e hibernação no systemd](#2)
3. [Método 2: ignorar fechamento da tampa do notebook](#3)
4. [Como reverter](#4)
5. [Referências](#referencias)

---

<a id="1"></a>
## 1. Antes de alterar

Confira o estado atual dos alvos de suspensão e hibernação:

```bash
systemctl list-unit-files sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

Em Debian 12/13 com `systemd`, esses alvos controlam os caminhos principais de suspensão, hibernação, suspensão híbrida e suspensão seguida de hibernação.

Em um sistema sem bloqueio manual, a saída costuma aparecer parecida com esta:

```text
UNIT FILE                     STATE  PRESET
hibernate.target              static -
hybrid-sleep.target           static -
sleep.target                  static -
suspend-then-hibernate.target static -
suspend.target                static -

5 unit files listed.
```

Se você está em servidor remoto, faça isso com cuidado. Bloquear suspensão é seguro na maioria dos cenários de servidor, mas reiniciar `systemd-logind` em desktop/notebook pode afetar integração com sessão gráfica. Salve o trabalho aberto antes de aplicar.

---

<a id="2"></a>
## 2. Método 1: bloquear suspensão e hibernação no systemd

Este é o caminho mais direto quando a máquina não deve dormir em nenhuma hipótese. É o método que faz mais sentido em servidor, Proxmox, host de laboratório, máquina de monitoramento ou qualquer equipamento que precisa ficar sempre acessível.

O `systemctl mask` cria um link da unit para `/dev/null`. Na prática, isso impede que o `systemd` inicie aquela unit, mesmo que outro componente tente chamar a suspensão.

Execute:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

Valide:

```bash
systemctl list-unit-files sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

O esperado é que os alvos apareçam como `masked`.

Exemplo:

```text
UNIT FILE                     STATE  PRESET
hibernate.target              masked enabled
hybrid-sleep.target           masked enabled
sleep.target                  masked enabled
suspend-then-hibernate.target masked enabled
suspend.target                masked enabled

5 unit files listed.
```

Neste caso, o campo mais importante é `STATE`. Se ele estiver como `masked`, o bloqueio está aplicado. O `PRESET` pode aparecer como `enabled`; isso é apenas a política de preset do sistema, não quer dizer que a suspensão esteja liberada.

Se quiser testar manualmente, o comando abaixo deve falhar informando que a unit está mascarada:

```bash
systemctl suspend
```

> [!CAUTION]
> Não use esse método em notebook que precisa suspender em uso normal. Ele é pensado para máquina que deve permanecer ligada.

---

<a id="3"></a>
## 3. Método 2: ignorar fechamento da tampa do notebook

Este método é útil quando você não quer bloquear todo o mecanismo de suspensão, mas quer impedir que o notebook suspenda ao fechar a tampa. É comum em notebook usado como desktop, conectado em monitor externo, teclado e mouse.

Em vez de editar diretamente `/etc/systemd/logind.conf`, prefiro criar um arquivo separado em `/etc/systemd/logind.conf.d/`. Fica mais fácil revisar e desfazer depois.

Crie o diretório:

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
```

Crie o arquivo:

```bash
sudo nano /etc/systemd/logind.conf.d/10-disable-lid-switch.conf
```

Cole o conteúdo abaixo:

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

O que cada opção faz:

- `HandleLidSwitch`: ação ao fechar a tampa usando bateria;
- `HandleLidSwitchExternalPower`: ação ao fechar a tampa na tomada;
- `HandleLidSwitchDocked`: ação ao fechar a tampa com dock ou monitor externo.

Reinicie o `systemd-logind` para aplicar:

```bash
sudo systemctl restart systemd-logind.service
```

Confira a configuração carregada:

```bash
systemd-analyze cat-config systemd/logind.conf
```

Procure na saída pelo arquivo `10-disable-lid-switch.conf` e pelas três opções com valor `ignore`.

Exemplo:

```text
# /etc/systemd/logind.conf.d/10-disable-lid-switch.conf
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Podem aparecer outros drop-ins do sistema ou de pacotes, como `unattended-upgrades`, antes ou depois do seu arquivo. Isso é normal. O ponto importante é confirmar que as opções `HandleLidSwitch`, `HandleLidSwitchExternalPower` e `HandleLidSwitchDocked` aparecem com valor `ignore`.

---

<a id="4"></a>
## 4. Como reverter

### 4.1 Reverter o método 1

Remova o bloqueio dos alvos:

```bash
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

Confira:

```bash
systemctl list-unit-files sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

### 4.2 Reverter o método 2

Renomeie o arquivo criado para ignorar a tampa:

```bash
sudo mv /etc/systemd/logind.conf.d/10-disable-lid-switch.conf /etc/systemd/logind.conf.d/10-disable-lid-switch.conf.bak
```

Reinicie o serviço:

```bash
sudo systemctl restart systemd-logind.service
```

Depois de confirmar que o comportamento voltou ao normal, você pode remover o `.bak` se não precisar manter histórico local.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### systemd

- `systemctl(1)` Debian Trixie: https://manpages.debian.org/trixie/systemd/systemctl.1.en.html
- `systemd.special(7)` Debian Trixie: https://manpages.debian.org/trixie/systemd/systemd.special.7.en.html
- `logind.conf.d(5)` Debian Bookworm Backports: https://manpages.debian.org/bookworm-backports/systemd/logind.conf.d.5.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
