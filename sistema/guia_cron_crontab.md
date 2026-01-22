# Guia Prático do Cron e Crontab: Agendando Tarefas no Linux

*Criado em 04 de dezembro de 2025 e atualizado em 08 de dezembro de 2025*

Este guia explica de forma direta e prática como usar o **Cron**, o agendador de tarefas padrão do Linux. Ao final, você saberá criar e gerenciar `crontabs` para automatizar scripts e comandos.

---

## Índice rápido
1. [O que é o Cron?](#1)
2. [Editando a Crontab do Usuário](#2)
3. [A Sintaxe do Cron (Minuto, Hora, Dia...)](#3)
4. [Operadores Especiais (`*`, `/`, `-`, `,`)](#4)
5. [Exemplos do Mundo Real](#5)
6. [Boas práticas recomendadas](#6)
7. [Checklist de depuração (cron não rodou)](#7)
8. [Crontab do Sistema vs. Crontab do Usuário](#8)
9. [Ferramentas Úteis](#9)

---

<a id="1"></a>
## 1. O que é o Cron?
O Cron é um serviço (daemon) que roda em segundo plano no sistema operacional e executa tarefas agendadas. Cada usuário pode ter sua própria lista de tarefas, que é chamada de **crontab**.

---

<a id="2"></a>
## 2. Editando a Crontab do Usuário
Para editar a sua lista de tarefas, o comando é simples:

```bash
crontab -e
```

Na primeira vez, o sistema pede para escolher um editor de texto (ex.: `nano`). Após escolher, o arquivo da crontab será aberto.

Para apenas listar as tarefas agendadas sem abrir o editor, use:
```bash
crontab -l
```

---

<a id="3"></a>
## 3. A Sintaxe do Cron (Minuto, Hora, Dia...)
Cada linha na crontab representa uma tarefa e segue uma estrutura de tempo muito específica, seguida pelo comando a ser executado.

```
┌───────────── minuto (0 - 59)
│ ┌───────────── hora (0 - 23)
│ │ ┌───────────── dia do mês (1 - 31)
│ │ │ ┌───────────── mês (1 - 12)
│ │ │ │ ┌───────────── dia da semana (0 - 6) (Domingo=0 ou 7)
│ │ │ │ │
* * * * * comando_para_executar
```

- **Minuto**: 0 a 59
- **Hora**: 0 a 23
- **Dia do Mês**: 1 a 31
- **Mês**: 1 a 12 (ou os nomes em inglês `jan`, `feb`, `mar`, etc.)
- **Dia da Semana**: 0 a 6 (onde Domingo pode ser 0 ou 7)

---

<a id="4"></a>
## 4. Operadores Especiais (`*`, `/`, `-`, `,`)
Para não precisar especificar cada valor, usamos alguns operadores para criar regras mais flexíveis.

- `*` **(Asterisco)**: Significa "todos os valores". Um `*` no campo de hora significa "a cada hora".
- `/` **(Barra)**: Usado para definir "passos". Por exemplo, `*/15` no campo de minutos significa "a cada 15 minutos".
- `-` **(Hífen)**: Usado para definir um "intervalo". Por exemplo, `8-17` no campo de hora significa "das 8h às 17h".
- `,` **(Vírgula)**: Usado para listar valores específicos. Por exemplo, `1,15,30` no campo de minutos significa "nos minutos 1, 15 e 30".

---

<a id="5"></a>
## 5. Exemplos do Mundo Real

| Tarefa | Crontab |
|---|---|
| Rodar um script de backup todo dia às 2h da manhã. | `0 2 * * * /home/paulo/scripts/backup.sh` |
| Verificar o espaço em disco a cada 10 minutos. | `*/10 * * * * /home/paulo/scripts/check_disk.sh` |
| Executar uma tarefa de limpeza toda segunda-feira à 1h30. | `30 1 * * 1 /home/paulo/scripts/limpeza.sh` |
| Rodar um comando no primeiro dia de cada mês. | `0 0 1 * * /usr/bin/atualizar_relatorios` |

---

<a id="6"></a>
## 6. Boas práticas recomendadas

### 6.1 Redirecione a Saída (stdout e stderr)
O cron envia um e-mail para o usuário com qualquer saída gerada pelo script. Para evitar isso (especialmente se o comando não gera saídas importantes), redirecione o `stdout` e o `stderr` para o "buraco negro" do Linux.

```bash
# Exemplo: não quero receber e-mail de sucesso ou erro
*/10 * * * * /home/paulo/scripts/check_disk.sh >/dev/null 2>&1
```

- `>/dev/null`: Redireciona a saída padrão (sucesso) para o nada.
- `2>&1`: Redireciona a saída de erro (`stderr`, descritor 2) para o mesmo lugar que a saída padrão (`stdout`, descritor 1).

### 6.2 Use Caminhos Absolutos
O cron não executa no mesmo ambiente do seu terminal. Variáveis de ambiente como `$PATH` podem ser diferentes. Por isso, sempre use o caminho absoluto tanto para seus scripts quanto para os comandos.

- **Ruim**: `* * * * * meu_script.sh`
- **Bom**: `* * * * * /home/paulo/scripts/meu_script.sh`

### 6.3 Use um Script
Se a sua automação precisa de mais de um comando, não coloque tudo na mesma linha do crontab. Crie um arquivo `.sh`, coloque os comandos lá, dê permissão de execução (`chmod +x meu_script.sh`) e chame esse script a partir do cron. Fica muito mais organizado.

---

<a id="7"></a>
## 7. Checklist de depuração (cron não rodou)
Se uma tarefa não executou como esperado, siga estes passos para investigar:

1.  **Verifique os logs do sistema:** O Cron registra suas ações no log do sistema.
    ```bash
    # Procure por 'CRON' no log do sistema (syslog)
    grep CRON /var/log/syslog

    # Ou, em sistemas mais novos com journald:
    journalctl -u cron.service
    ```
    Os logs mostrarão se o cron tentou executar seu comando e se houve algum erro imediato.

2.  **Verifique o e-mail local:** Se você não redirecionou a saída (`>/dev/null 2>&1`), o cron tentará enviar um e-mail com a saída ou erro para o usuário.
    ```bash
    # Verifique a caixa de e-mail local do seu usuário
    mail
    ```

3.  **Permissões:** O script que você está tentando rodar tem permissão de execução?
    ```bash
    ls -l /home/paulo/scripts/meu_script.sh
    # Certifique-se de que ele tem o 'x' (execução) nas permissões.
    # Se não tiver, adicione com: chmod +x /home/paulo/scripts/meu_script.sh
    ```

4.  **Caminhos Absolutos:** Você usou o caminho completo para o script e para os comandos dentro dele? Lembre-se, o `$PATH` do cron é mínimo.

---

<a id="8"></a>
## 8. Crontab do Sistema vs. Crontab do Usuário

- **`crontab -e`**: Edita a crontab do **usuário logado**. Os comandos aqui rodam com a permissão desse usuário.
- **`/etc/crontab` e `/etc/cron.d/`**: São os arquivos de crontab do **sistema**. Para editar, você precisa ser `root`. A sintaxe é um pouco diferente, pois você precisa especificar **qual usuário** vai rodar o comando.

**Exemplo de linha no `/etc/crontab`:**
```
# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
```
Note a coluna `user` (`root` neste caso) antes do comando.

---

<a id="9"></a>
## 9. Ferramentas Úteis

Se você tem dúvida sobre a sintaxe de tempo, o site **Crontab Guru** é uma ferramenta fantástica. Ele traduz a sua regra de tempo para uma linguagem humana, ajudando a evitar erros.

[https://crontab.guru/](https://crontab.guru/)

---

## Referências (fontes para consulta)

### Cron (manpages Debian)

- `cron(8)`: https://manpages.debian.org/bookworm/cron/cron.8.en.html
- `crontab(1)`: https://manpages.debian.org/bookworm/cron/crontab.1.en.html
- `crontab(5)`: https://manpages.debian.org/bookworm/cron/crontab.5.en.html

### Ferramentas

- Crontab Guru: https://crontab.guru/

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
