# Guia Prático: Sincronização de pastas com rsync

*Criado em: 04 de dezembro de 2025*  
*Última atualização em: 18 de maio de 2026*

`rsync` é uma daquelas ferramentas que vale dominar cedo no Linux. Ele serve para copiar, sincronizar e espelhar diretórios locais ou remotos, economizando tempo porque transfere apenas o que mudou. O ponto de atenção é que `rsync` também pode apagar arquivos no destino quando usado como espelho, então este guia prioriza comandos testáveis, com `--dry-run`, logs e exemplos seguros para rotina de servidor.

---

## Índice rápido
1. [Instalando o rsync](#1)
2. [Sintaxe básica](#2)
3. [A barra final na origem](#3)
4. [Opções mais usadas](#4)
5. [Exemplos locais](#5)
6. [Sincronização remota via SSH](#6)
7. [Excluindo arquivos e pastas](#7)
8. [Automatizando com cron](#8)
9. [Cuidados antes de usar em produção](#9)
10. [Referências](#referencias)

---

<a id="1"></a>
## 1. Instalando o rsync

No Debian/Ubuntu:
```bash
sudo apt update
sudo apt install -y rsync
rsync --version
```

No AlmaLinux/Rocky Linux:
```bash
sudo dnf install -y rsync
rsync --version
```

Para sincronização remota via SSH, o `rsync` precisa existir nos dois lados: origem e destino.

---

<a id="2"></a>
## 2. Sintaxe básica

A estrutura geral é:

```bash
rsync [opções] /caminho/da/origem/ /caminho/do/destino/
```

Exemplo:

```bash
rsync -avh /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

Onde:

- `/home/paulo/Documentos/` é a origem;
- `/media/paulo/Backup/Documentos/` é o destino;
- `-a` preserva estrutura e atributos básicos;
- `-v` mostra o que está sendo feito;
- `-h` mostra tamanhos em formato legível.

---

<a id="3"></a>
## 3. A barra final na origem

Essa é a pegadinha mais comum do `rsync`.

Com barra no final:

```bash
rsync -avh /origem/ /destino/
```

Copia o **conteúdo** de `/origem/` para dentro de `/destino/`.

Sem barra no final:

```bash
rsync -avh /origem /destino/
```

Copia a **pasta origem** para dentro de `/destino/`, criando `/destino/origem`.

Na maioria dos backups de pasta, a forma mais previsível é usar a barra no final da origem.

---

<a id="4"></a>
## 4. Opções mais usadas

| Opção | Função |
|---|---|
| `-a`, `--archive` | Modo arquivo; equivale a `-rlptgoD`. Preserva links, permissões, datas, grupo, dono e devices. |
| `-v`, `--verbose` | Mostra mais detalhes da execução. |
| `-h`, `--human-readable` | Mostra tamanhos em formato legível. |
| `-i`, `--itemize-changes` | Mostra um resumo do que mudaria ou mudou em cada item. |
| `-n`, `--dry-run` | Simula a execução sem alterar nada. |
| `--delete` | Apaga no destino arquivos que não existem mais na origem. Use com cuidado. |
| `--exclude=PADRAO` | Ignora arquivos ou diretórios que combinam com o padrão informado. |
| `-z`, `--compress` | Comprime dados durante transferência remota; útil em link lento. |
| `--log-file=ARQUIVO` | Grava log da execução em arquivo. |
| `--progress` | Mostra progresso por arquivo, útil em execução manual. |

Observação importante: `-a` não inclui tudo. Ele não preserva ACLs, atributos estendidos ou hardlinks. Se isso for necessário, avalie opções como `-A`, `-X` e `-H`, testando antes com `--dry-run`.

---

<a id="5"></a>
## 5. Exemplos locais

### 5.1 Copiar uma pasta para outro disco

Primeiro simule:

```bash
rsync -avhi --dry-run /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

Se a lista estiver correta, execute de verdade:

```bash
rsync -avh /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

### 5.2 Espelhar um diretório com `--delete`

Espelhar significa deixar o destino igual à origem. Arquivos removidos da origem também serão removidos do destino.

Teste primeiro:

```bash
rsync -avhi --delete --dry-run /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

Se o teste mostrar as remoções esperadas, rode:

```bash
rsync -avh --delete /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

Antes de usar `--delete`, confira três vezes se origem e destino estão corretos. Um caminho invertido pode causar perda de dados.

---

<a id="6"></a>
## 6. Sincronização remota via SSH

Enviar arquivos para outro servidor:

```bash
rsync -avh -e ssh /home/paulo/Documentos/ usuario@servidor:/backup/paulo/Documentos/
```

Baixar arquivos de outro servidor:

```bash
rsync -avh -e ssh usuario@servidor:/backup/paulo/Documentos/ /home/paulo/Documentos/
```

Com porta SSH personalizada:

```bash
rsync -avh -e "ssh -p 2222" /home/paulo/Documentos/ usuario@servidor:/backup/paulo/Documentos/
```

Se for automatizar via SSH, use chave pública e teste o login antes:

```bash
ssh usuario@servidor
```

Veja também: [Guia de Produção: Acesso SSH por chave pública em servidores Linux](../acesso-remoto/guia_producao_ssh_chave_publica_linux.md)

---

<a id="7"></a>
## 7. Excluindo arquivos e pastas

Exemplo ignorando cache, arquivos temporários e logs:

```bash
rsync -avh \
  --exclude='.cache/' \
  --exclude='*.tmp' \
  --exclude='*.log' \
  /home/paulo/Documentos/ \
  /media/paulo/Backup/Documentos/
```

Quando houver muitos padrões, use um arquivo de exclusões:

```bash
nano /home/paulo/rsync-excludes.txt
```

Exemplo de conteúdo:

```text
.cache/
*.tmp
*.log
node_modules/
```

Usando o arquivo:

```bash
rsync -avh --exclude-from=/home/paulo/rsync-excludes.txt /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

---

<a id="8"></a>
## 8. Automatizando com cron

Para uma rotina simples, cron resolve bem. Antes de automatizar, teste o comando manualmente com `--dry-run`.

Crie uma pasta para logs:

```bash
mkdir -p /home/paulo/logs
```

Edite a crontab:

```bash
crontab -e
```

Exemplo rodando todo dia às 3h, sem apagar arquivos do destino:

```cron
0 3 * * * /usr/bin/rsync -a --log-file=/home/paulo/logs/rsync-documentos.log /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

Para evitar execução sobreposta, use `flock`:

```cron
0 3 * * * /usr/bin/flock -n /tmp/rsync-documentos.lock /usr/bin/rsync -a --log-file=/home/paulo/logs/rsync-documentos.log /home/paulo/Documentos/ /media/paulo/Backup/Documentos/
```

Se a intenção for espelhar e apagar no destino o que foi apagado na origem, adicione `--delete` somente depois de testar manualmente com `--dry-run`.

Veja também: [Guia Prático: Cron e crontab no Linux](guia_cron_crontab.md)

Para rotinas críticas de produção, principalmente quando dependem de rede, discos montados ou serviços, prefira avaliar `systemd timers`.

---

<a id="9"></a>
## 9. Cuidados antes de usar em produção

- Sempre rode com `--dry-run` antes de usar `--delete`.
- Confirme se a barra final na origem está correta.
- Confirme se o destino é realmente o destino.
- Em automações, use caminhos absolutos.
- Guarde logs das execuções importantes.
- Teste restauração, não apenas cópia.
- Não trate `rsync --delete` como backup versionado. Ele espelha o estado atual; se um arquivo for apagado na origem, pode ser apagado no destino.
- Para backup com histórico, avalie snapshots, Proxmox Backup Server, Borg, Restic ou outro método com retenção/versionamento.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### rsync

- `rsync(1)` (manpage Debian Trixie): https://manpages.debian.org/trixie/rsync/rsync.1.en.html
- Documentação oficial do rsync: https://rsync.samba.org/documentation.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
