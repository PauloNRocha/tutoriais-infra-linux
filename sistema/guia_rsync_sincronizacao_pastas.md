# Guia de Produção: Sincronização de Pastas com `rsync`

*Criado em 04 de dezembro de 2025 e atualizado em 08 de dezembro de 2025*

Se a necessidade é manter uma cópia de segurança (backup) de uma pasta ou espelhar diretórios no Linux, o `rsync` é uma das ferramentas mais usadas para sincronização eficiente e segura. Este guia mostra exemplos práticos de uso, do básico ao caso comum de automação com `cron`.

---

## Índice rápido
1. [Por que usar o `rsync`?](#1)
2. [A Sintaxe Básica](#2)
3. [As Flags Mais Importantes (Opções)](#3)
4. [Exemplos Práticos](#4)
5. [Automatizando com Cron](#5)

---

<a id="1"></a>
## 1. Por que usar o `rsync`?
O `rsync` é superior a um simples `cp` (comando de copiar) porque ele usa um algoritmo de "delta-transfer". Isso significa que ele só copia as partes dos arquivos que foram alteradas, tornando as sincronizações subsequentes extremamente rápidas e eficientes, economizando tempo e largura de banda.

---

<a id="2"></a>
## 2. A Sintaxe Básica
A estrutura de um comando `rsync` é muito simples:

```bash
rsync [opções] /caminho/da/origem/ /caminho/do/destino/
```
> **Dica de ouro:** Preste atenção na barra `/` no final do caminho de origem.
> - `/origem/`: Copia o **conteúdo** da pasta de origem para o destino.
> - `/origem`: Copia a **pasta de origem em si** para dentro do destino.
> Na maioria dos casos de backup, você vai querer usar a barra no final (`/origem/`).

---

<a id="3"></a>
## 3. As Flags Mais Importantes (Opções)
As opções (ou flags) mudam o comportamento do `rsync`. Estas são algumas das mais úteis no dia a dia:

- `-a` (archive): É um modo "tudo em um". Ele preserva permissões, dono, datas e copia os diretórios recursivamente. É a opção mais comum para backups.
- `-v` (verbose): Mostra no terminal quais arquivos estão sendo transferidos. Ótimo para saber o que está acontecendo.
- `-h` (human-readable): Exibe os tamanhos dos arquivos em um formato legível (KB, MB, GB).
- `--delete`: **(Use com cuidado!)** Esta opção faz com que arquivos que foram **apagados na origem** também sejam **apagados no destino**. É o que transforma a sincronização em um verdadeiro "espelho".
- `--dry-run`: **(Sua melhor amiga!)** Simula a execução do comando sem de fato copiar ou apagar nada. Perfeito para testar seu comando e ver o que ele *iria* fazer.
- `--progress`: Mostra o progresso da transferência de arquivos grandes.

### Tabela de Parâmetros Comuns

| Parâmetro        | Descrição                                                                                               |
| :--------------- | :------------------------------------------------------------------------------------------------------ |
| `-a`, `--archive`    | Modo arquivo; equivalente a `-rlptgoD` (recursivo, links, permissões, tempos, grupo, dono, devices).    |
| `-v`, `--verbose`    | Aumenta a quantidade de informações mostradas durante a execução.                                     |
| `-h`, `--human-readable` | Exibe os tamanhos dos arquivos em um formato fácil de ler (KB, MB, GB).                              |
| `--delete`       | Deleta arquivos no destino que não existem mais na origem. **Use com extrema cautela.**                 |
| `--dry-run`      | Simula a operação sem realizar nenhuma alteração. **Sempre use para testar!**                         |
| `--progress`     | Mostra o progresso da transferência de cada arquivo.                                                  |
| `--exclude=PADRÃO` | Exclui arquivos ou diretórios que correspondem ao padrão especificado. Ex: `--exclude='*.tmp'`        |
| `-z`, `--compress` | Comprime os dados do arquivo durante a transferência (útil para links lentos).                         |
| `-u`, `--update` | Pula arquivos que são mais novos no destino.                                                           |
| `--chown=USER:GROUP` | Altera o dono e grupo dos arquivos no destino.                                                        |

---

<a id="4"></a>
## 4. Exemplos Práticos

### Exemplo 1: Backup para um HD Externo
Vamos supor que você queira fazer um backup da sua pasta `Documentos` para um HD externo montado em `/media/usuario/Backup`.

Primeiro, vamos simular a operação com `--dry-run`:
```bash
rsync -avh --dry-run /home/usuario/Documentos/ /media/usuario/Backup/Documentos/
```
O comando vai listar todos os arquivos que *seriam* copiados. Se a lista parecer correta, podemos rodar o comando de verdade:
```bash
rsync -avh /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

### Exemplo 2: Espelhar um diretório (com `--delete`)
Agora, queremos que a pasta de backup seja um espelho exato, apagando do destino os arquivos que não existem mais na origem.

Vamos testar primeiro:
```bash
rsync -avh --delete --dry-run /home/usuario/Documentos/ /media/usuario/Backup/Documentos/
```
Se o teste mostrar que os arquivos corretos seriam deletados, rode o comando final:
```bash
rsync -avh --delete /home/usuario/Documentos/ /media/usuario/Backup/Documentos/
```

---

<a id="5"></a>
## 5. Automatizando com Cron
Fazer backup manual é bom, mas automatizar é melhor ainda. Podemos usar o `cron` para rodar nosso comando `rsync` todo dia.

1.  Abra o editor do crontab:
    ```bash
    crontab -e
    ```
2.  Adicione a seguinte linha no final do arquivo para rodar o backup todo dia às 3h da manhã:

    ```crontab
    # Rodar backup diário da pasta Documentos
    0 3 * * * rsync -a --delete /home/paulo/Documentos/ /media/paulo/Backup/Documentos/ >/dev/null 2>&1
    ```
- `-a` é suficiente para scripts (sem `-vh` para não gerar saída).
- `>/dev/null 2>&1` impede que o cron te envie um e-mail a cada execução.

Agora, seu backup está automatizado e você pode ficar tranquilo sabendo que seus arquivos importantes estão seguros.

---

## Referências (fontes para consulta)

### rsync

- `rsync(1)` (manpage Debian): https://manpages.debian.org/bookworm/rsync/rsync.1.en.html
- Projeto rsync (site oficial): https://rsync.samba.org/

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
