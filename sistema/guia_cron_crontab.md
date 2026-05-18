# Guia Prático: Cron e crontab no Linux

*Criado em: 04 de dezembro de 2025*  
*Última atualização em: 18 de maio de 2026*

Cron ainda é uma das formas mais simples de agendar tarefas no Linux. Eu costumo usar para rotinas pequenas, scripts pessoais, checagens simples e tarefas que não precisam de toda a estrutura de um serviço `systemd`. Para automações críticas de produção, especialmente quando dependem de rede, discos montados, outros serviços ou recuperação após boot/falha, vale avaliar `systemd timers`, mas entender cron continua sendo obrigatório para administrar servidores Linux com segurança.

---

## Índice rápido
1. [O que é o Cron](#1)
2. [Instalando e verificando o serviço](#2)
3. [Editando a crontab do usuário](#3)
4. [Sintaxe do cron](#4)
5. [Operadores e atalhos especiais](#5)
6. [Exemplos práticos](#6)
7. [Boas práticas recomendadas](#7)
8. [Checklist de depuração](#8)
9. [Crontab do sistema vs. crontab do usuário](#9)
10. [Quando usar cron ou systemd timer](#10)
11. [Ferramentas úteis](#11)
12. [Referências](#referencias)

---

<a id="1"></a>
## 1. O que é o Cron

O Cron é um serviço que roda em segundo plano e executa comandos em horários definidos. Cada usuário pode ter sua própria lista de tarefas, chamada de `crontab`.

Exemplos comuns:

- executar backup todo dia de madrugada;
- rodar uma checagem de disco a cada 10 minutos;
- limpar arquivos temporários uma vez por semana;
- disparar um script de relatório no primeiro dia do mês.

---

<a id="2"></a>
## 2. Instalando e verificando o serviço

Em muitos servidores o cron já vem instalado, mas em instalações minimalistas, imagens cloud, containers ou ambientes enxutos isso nem sempre acontece.

No Debian/Ubuntu:
```bash
sudo apt update
sudo apt install -y cron
sudo systemctl enable --now cron
systemctl status cron --no-pager
```

No AlmaLinux/Rocky Linux:
```bash
sudo dnf install -y cronie
sudo systemctl enable --now crond
systemctl status crond --no-pager
```

---

<a id="3"></a>
## 3. Editando a crontab do usuário

Antes de editar, gosto de salvar uma cópia da crontab atual. Isso evita perder agendamentos por engano.

```bash
crontab -l > "$HOME/crontab.backup.$(date +%F_%H%M%S)"
```

Se ainda não existir crontab para o usuário, o comando pode avisar `no crontab for usuario`. Nesse caso, não há nada para salvar ainda; siga para a criação com `crontab -e`.

Para editar a crontab do usuário atual:

```bash
crontab -e
```

Na primeira vez, o sistema pode pedir para escolher um editor de texto. Se você não tiver preferência, `nano` costuma ser a opção mais simples.

Para listar as tarefas agendadas:

```bash
crontab -l
```

Para remover a crontab inteira, existe o comando abaixo, mas use com cuidado:

```bash
crontab -r
```

Na prática, antes de usar `crontab -r`, confira se você tem backup da crontab atual.

---

<a id="4"></a>
## 4. Sintaxe do cron

Cada linha da crontab representa uma tarefa e segue esta estrutura:

```text
┌───────────── minuto (0 - 59)
│ ┌───────────── hora (0 - 23)
│ │ ┌───────────── dia do mês (1 - 31)
│ │ │ ┌───────────── mês (1 - 12)
│ │ │ │ ┌───────────── dia da semana (0 - 7, domingo = 0 ou 7)
│ │ │ │ │
* * * * * comando_para_executar
```

Campos:

- **Minuto**: `0` a `59`
- **Hora**: `0` a `23`
- **Dia do mês**: `1` a `31`
- **Mês**: `1` a `12`, ou nomes em inglês como `jan`, `feb`, `mar`
- **Dia da semana**: `0` a `7`, sendo domingo `0` ou `7`

### 4.1 Atenção ao combinar dia do mês e dia da semana

Uma pegadinha comum é preencher ao mesmo tempo o campo **dia do mês** e o campo **dia da semana**.

Exemplo:

```cron
0 0 1 * 1 /home/paulo/scripts/rotina.sh
```

Isso não significa necessariamente "rodar no dia 1 se for segunda-feira". No cron tradicional, quando os dois campos estão restritos, a regra funciona como **OU**:

- roda no dia `1` de cada mês;
- e também roda toda segunda-feira.

Se a intenção for uma condição mais específica, normalmente é melhor validar dentro do próprio script.

---

<a id="5"></a>
## 5. Operadores e atalhos especiais

| Operador | Função | Exemplo |
|---|---|---|
| `*` | Todos os valores | `* * * * *` roda todo minuto |
| `/` | Intervalo por passo | `*/15 * * * *` roda a cada 15 minutos |
| `-` | Faixa | `0 8-17 * * *` roda de 8h até 17h |
| `,` | Lista de valores | `0 8,12,18 * * *` roda às 8h, 12h e 18h |

### 5.1 Atalhos especiais

O cron também aceita alguns atalhos no lugar dos cinco campos de tempo.

| Atalho | Equivalente | Função |
|---|---|---|
| `@reboot` | sem equivalente direto | roda quando o daemon do cron inicia |
| `@hourly` | `0 * * * *` | roda uma vez por hora |
| `@daily` / `@midnight` | `0 0 * * *` | roda uma vez por dia |
| `@weekly` | `0 0 * * 0` | roda uma vez por semana |
| `@monthly` | `0 0 1 * *` | roda uma vez por mês |
| `@yearly` / `@annually` | `0 0 1 1 *` | roda uma vez por ano |

Atenção com `@reboot`: ele roda quando o cron inicia, não necessariamente quando rede, discos remotos ou outros serviços já estão prontos. Para tarefas dependentes do boot, `systemd timers` costuma ser mais previsível.

---

<a id="6"></a>
## 6. Exemplos práticos

| Tarefa | Linha no crontab |
|---|---|
| Rodar backup todo dia às 2h | `0 2 * * * /home/paulo/scripts/backup.sh` |
| Verificar disco a cada 10 minutos | `*/10 * * * * /home/paulo/scripts/check_disk.sh` |
| Executar limpeza toda segunda-feira à 1h30 | `30 1 * * 1 /home/paulo/scripts/limpeza.sh` |
| Rodar relatório no primeiro dia de cada mês | `0 0 1 * * /usr/local/bin/atualizar_relatorios` |
| Rodar tarefa todo domingo às 3h | `0 3 * * 0 /home/paulo/scripts/tarefa_domingo.sh` |

---

<a id="7"></a>
## 7. Boas práticas recomendadas

### 7.1 Use caminhos absolutos

O cron não executa com o mesmo ambiente do terminal interativo. Por isso, evite depender de aliases, funções do shell ou caminhos relativos.

Evite:

```cron
* * * * * meu_script.sh
```

Prefira:

```cron
* * * * * /home/paulo/scripts/meu_script.sh
```

### 7.2 Defina ambiente quando necessário

Se o script depende de `bash` ou de comandos em caminhos específicos, você pode definir variáveis no topo da crontab.

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Isso ajuda a evitar aquele problema clássico: o comando roda manualmente, mas falha no cron.

### 7.3 Registre logs das tarefas importantes

Evite jogar tudo em `/dev/null` quando a tarefa for importante. Para rotina de produção, é melhor guardar pelo menos um log simples.

```cron
*/10 * * * * /home/paulo/scripts/check_disk.sh >> /home/paulo/logs/check_disk.log 2>&1
```

Antes, garanta que a pasta existe:

```bash
mkdir -p /home/paulo/logs
```

Se a tarefa realmente não precisa gerar log:

```cron
*/10 * * * * /home/paulo/scripts/check_disk.sh >/dev/null 2>&1
```

### 7.4 Use script quando houver mais de um comando

Se a automação precisa de mais de um comando, crie um arquivo `.sh` e chame esse script pelo cron. Fica mais fácil testar, versionar e corrigir.

Exemplo:

```bash
nano /home/paulo/scripts/backup.sh
chmod +x /home/paulo/scripts/backup.sh
```

Linha no crontab:

```cron
0 2 * * * /home/paulo/scripts/backup.sh >> /home/paulo/logs/backup.log 2>&1
```

### 7.5 Evite execução sobreposta com `flock`

Se uma tarefa roda a cada poucos minutos, ela pode iniciar de novo antes da execução anterior terminar. Para evitar sobreposição, use `flock`.

```cron
*/5 * * * * /usr/bin/flock -n /tmp/check_disk.lock /home/paulo/scripts/check_disk.sh >> /home/paulo/logs/check_disk.log 2>&1
```

Esse cuidado é útil para backups, sincronizações, verificações pesadas e rotinas que mexem em arquivos compartilhados.

### 7.6 Cuidado com `%` dentro da linha do cron

No crontab, o caractere `%` tem tratamento especial. Se ele aparecer sem escape dentro do comando, o cron pode interpretar como quebra de linha e enviar o restante como entrada padrão do comando.

Evite colocar comandos complexos direto no crontab, principalmente quando usam `date`.

Se precisar usar `%` diretamente, escape com `\%`:

```cron
0 2 * * * echo "$(date +\%F)" >> /tmp/teste-cron.log
```

Na prática, para qualquer coisa maior que uma linha simples, prefira colocar a lógica dentro de um script `.sh` e chamar o script pelo cron.

---

<a id="8"></a>
## 8. Checklist de depuração

Se uma tarefa não rodou como esperado, siga esta ordem:

1. Verifique se o serviço está ativo.

   Debian/Ubuntu:
   ```bash
   systemctl status cron --no-pager
   ```

   AlmaLinux/Rocky Linux:
   ```bash
   systemctl status crond --no-pager
   ```

2. Confira se a tarefa está realmente no crontab.

   ```bash
   crontab -l
   ```

3. Rode o script manualmente com o mesmo usuário.

   ```bash
   /home/paulo/scripts/meu_script.sh
   ```

4. Verifique permissões.

   ```bash
   ls -l /home/paulo/scripts/meu_script.sh
   ```

   Se faltar permissão de execução:

   ```bash
   chmod +x /home/paulo/scripts/meu_script.sh
   ```

5. Confira os logs do cron.

   Debian/Ubuntu:
   ```bash
   sudo grep CRON /var/log/syslog
   sudo journalctl -u cron.service --since "1 hour ago"
   ```

   AlmaLinux/Rocky Linux:
   ```bash
   sudo grep CRON /var/log/cron
   sudo journalctl -u crond.service --since "1 hour ago"
   ```

6. Verifique se o script usa caminhos absolutos.

   Dentro do script, prefira:

   ```bash
   /usr/bin/rsync
   /usr/bin/find
   /usr/bin/curl
   ```

   em vez de apenas:

   ```bash
   rsync
   find
   curl
   ```

7. Confira o fuso horário do servidor.

   ```bash
   timedatectl
   ```

8. Em arquivos editados manualmente, confirme se existe uma nova linha no final.

   Isso é mais comum em arquivos em `/etc/cron.d/`. Se a última linha terminar direto no fim do arquivo, sem newline, o cron pode considerar a entrada quebrada ou parcialmente quebrada.

---

<a id="9"></a>
## 9. Crontab do sistema vs. crontab do usuário

Existem dois usos comuns:

- `crontab -e`: edita a crontab do usuário atual;
- `/etc/crontab` e `/etc/cron.d/`: arquivos de agendamento do sistema.

Na crontab do usuário, você não informa o usuário na linha:

```cron
0 2 * * * /home/paulo/scripts/backup.sh
```

Em `/etc/crontab` e `/etc/cron.d/`, existe uma coluna extra informando qual usuário executa o comando:

```cron
# m h dom mon dow user  command
0 2 * * * root /usr/local/sbin/backup-geral.sh
```

No Debian, arquivos em `/etc/cron.d/` devem seguir alguns cuidados:

- pertencer ao usuário `root`;
- não ser graváveis por grupo ou outros;
- ter nome simples, sem ponto no nome;
- não precisam ser executáveis;
- incluir a coluna do usuário que executará o comando.

Arquivos em `/etc/cron.d/` não aparecem no `crontab -l`, porque não fazem parte da crontab pessoal do usuário. Para conferir esses agendamentos, veja o arquivo diretamente:

```bash
sudo ls -l /etc/cron.d/
sudo cat /etc/cron.d/nome-do-arquivo
```

Para tarefas próprias de usuário, `crontab -e` normalmente é suficiente. Para tarefas do sistema, use `/etc/crontab`, `/etc/cron.d/` ou avalie `systemd timers`.

---

<a id="10"></a>
## 10. Quando usar cron ou systemd timer

Use cron quando:

- a tarefa é simples;
- o agendamento é direto;
- não precisa de dependências do `systemd`;
- você quer algo rápido e fácil de revisar com `crontab -l`.

Prefira `systemd timer` quando:

- a tarefa é crítica em produção;
- precisa depender de rede, disco montado ou outro serviço;
- você quer logs integrados no `journald`;
- você quer controlar a execução com `systemctl`;
- precisa de comportamento mais previsível após boot ou falha.

Resumo prático: cron resolve muito bem tarefas simples. Para rotinas críticas de servidor, `systemd timers` costuma ser mais organizado.

---

<a id="11"></a>
## 11. Ferramentas úteis

Se você tem dúvida sobre a sintaxe de tempo, o Crontab Guru ajuda bastante:

https://crontab.guru/

Ele traduz expressões como `*/15 * * * *` para uma descrição em linguagem humana, o que reduz erro de agendamento.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### Cron (manpages Debian)

- `cron(8)`: https://manpages.debian.org/trixie/cron/cron.8.en.html
- `crontab(1)`: https://manpages.debian.org/trixie/cron/crontab.1.en.html
- `crontab(5)`: https://manpages.debian.org/trixie/cron/crontab.5.en.html
- `systemd.timer(5)`: https://manpages.debian.org/trixie/systemd/systemd.timer.5.en.html

### Ferramentas

- Crontab Guru: https://crontab.guru/

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
