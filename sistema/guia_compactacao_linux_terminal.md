# Guia Prático: Compactar e extrair arquivos no terminal Linux

*Criado em: 08 de dezembro de 2025*  
*Última atualização em: 18 de maio de 2026*

Compactar e extrair arquivos pelo terminal é uma tarefa simples, mas os formatos confundem um pouco no começo. Este guia mostra os comandos mais usados no Linux para trabalhar com `.tar.gz`, `.tar.bz2`, `.tar.xz` e `.zip`, sem entrar em ferramentas gráficas ou casos muito específicos.

---

## Índice rápido
1. [Instalando as ferramentas](#1)
2. [Comparando os formatos de compressão](#2)
3. [Como compactar uma pasta](#3)
4. [Como listar o conteúdo sem extrair](#4)
5. [Como testar um arquivo compactado](#5)
6. [Como extrair arquivos](#6)
7. [Extraindo para um diretório específico](#7)
8. [Compactando sem incluir alguns arquivos](#8)
9. [Referências](#referencias)

---

<a id="1"></a>
## 1. Instalando as ferramentas

Em muitas instalações Linux, `tar`, `gzip`, `bzip2` e `xz` já vêm instalados. Mesmo assim, quando o sistema é minimal ou recém-instalado, é melhor garantir os pacotes antes de seguir.

No Debian/Ubuntu:
```bash
sudo apt update
sudo apt install -y tar gzip bzip2 xz-utils zip unzip
```

No AlmaLinux/Rocky Linux:
```bash
sudo dnf install -y tar gzip bzip2 xz zip unzip
```

---

<a id="2"></a>
## 2. Comparando os formatos de compressão

| Formato      | Extensão      | Equilíbrio                                    | Uso Comum                               |
| :----------- | :------------ | :-------------------------------------------- | :-------------------------------------- |
| **gzip**     | `.tar.gz`     | Rápido, boa compressão e muito comum.         | Pacotes de software, logs, backups rápidos. |
| **bzip2**    | `.tar.bz2`    | Mais lento, compressão melhor que o gzip.     | Distribuições de código-fonte.          |
| **xz**       | `.tar.xz`     | Muito lento, a melhor taxa de compressão.     | Lançamentos de kernel, arquivamento de longo prazo. |
| **zip**      | `.zip`        | Rápido, compressão razoável.                  | Compatibilidade máxima, especialmente com Windows. |

---

<a id="3"></a>
## 3. Como compactar uma pasta

Nestes exemplos, vamos compactar uma pasta chamada `meus_arquivos` localizada em `~/documentos/`.

### Compactando com `tar` (`.gz`, `.bz2`, `.xz`)
O `tar` é o empacotador: ele junta arquivos e pastas em um único arquivo. As opções `-z`, `-j` e `-J` definem qual compressor será usado.

-   `-c`: **c**ria um novo arquivo.
-   `-v`: modo **v**erboso, mostrando os arquivos sendo adicionados.
-   `-f`: especi**f**ica o nome do arquivo de saída.

**1) gzip (`.tar.gz`)**
```bash
tar -czvf meus_arquivos.tar.gz ~/documentos/meus_arquivos/
```
- `-z`: Usa o compressor **g(z)ip**.

**2) bzip2 (`.tar.bz2`)**
```bash
tar -cjvf meus_arquivos.tar.bz2 ~/documentos/meus_arquivos/
```
- `-j`: Usa o compressor **bzip2**.

**3) xz (`.tar.xz`)**
```bash
tar -cJvf meus_arquivos.tar.xz ~/documentos/meus_arquivos/
```
- `-J` (maiúsculo): Usa o compressor **xz**.

### 4) Compactando com `zip`
O `zip` é uma ferramenta separada, útil quando o arquivo precisa ser aberto também em Windows ou por pessoas menos acostumadas com `.tar.*`.
- `-r`: **r**ecursivo (inclui subpastas).

```bash
zip -r meus_arquivos.zip ~/documentos/meus_arquivos/
```

---

<a id="4"></a>
## 4. Como listar o conteúdo sem extrair

Antes de extrair um arquivo recebido de outra pessoa ou baixado da internet, vale listar o conteúdo. Isso evita jogar arquivos soltos na pasta errada.

- **Para arquivos `.tar.*`**, use a flag `-t` (de **t**este ou lis**t**ar):
  ```bash
  tar -tf meus_arquivos.tar.gz
  tar -tf meus_arquivos.tar.bz2
  tar -tf meus_arquivos.tar.xz
  ```

- **Para arquivos `.zip`**, use a flag `-l` (de **l**istar) no `unzip`:
  ```bash
  unzip -l meus_arquivos.zip
  ```

---

<a id="5"></a>
## 5. Como testar um arquivo compactado

Quando o arquivo veio de backup, download ou transferência pela rede, vale testar a integridade antes de extrair.

```bash
gzip -t meus_arquivos.tar.gz
bzip2 -t meus_arquivos.tar.bz2
xz -t meus_arquivos.tar.xz
unzip -t meus_arquivos.zip
```

Se o comando não retornar erro, o arquivo passou no teste básico de integridade.

---

<a id="6"></a>
## 6. Como extrair arquivos

### Extraindo com `tar`
A flag principal aqui é `-x` (de e**x**trair).

**1) gzip (`.tar.gz`)**
```bash
tar -xzvf meus_arquivos.tar.gz
```

**2) bzip2 (`.tar.bz2`)**
```bash
tar -xjvf meus_arquivos.tar.bz2
```

**3) xz (`.tar.xz`)**
```bash
tar -xJvf meus_arquivos.tar.xz
```

### 4) Extraindo com `unzip`
```bash
unzip meus_arquivos.zip
```

---

<a id="7"></a>
## 7. Extraindo para um diretório específico

Por padrão, os arquivos são extraídos na pasta atual. Para mudar isso:

- **Para arquivos `.tar.*`**, use a flag `-C` (maiúscula) seguida do caminho de destino:
  ```bash
  tar -xzvf meus_arquivos.tar.gz -C /tmp/destino/
  ```

- **Para arquivos `.zip`**, use a flag `-d` (de **d**iretório):
  ```bash
  unzip meus_arquivos.zip -d /tmp/destino/
  ```

---

<a id="8"></a>
## 8. Compactando sem incluir alguns arquivos

Em backups manuais, é comum querer compactar uma pasta sem levar cache, logs ou arquivos temporários.

Exemplo com `tar.gz`:
```bash
tar --exclude='cache' --exclude='*.log' --exclude='tmp' -czvf meus_arquivos.tar.gz ~/documentos/meus_arquivos/
```

Neste exemplo:

- `--exclude='cache'`: ignora arquivos ou pastas chamados `cache`;
- `--exclude='*.log'`: ignora arquivos terminados em `.log`;
- `--exclude='tmp'`: ignora arquivos ou pastas chamados `tmp`.

Com estes comandos, você consegue resolver a maioria das necessidades de compactação e extração diretamente pelo terminal.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### Manpages (Debian)

- `tar(1)`: https://manpages.debian.org/bookworm/tar/tar.1.en.html
- `gzip(1)`: https://manpages.debian.org/bookworm/gzip/gzip.1.en.html
- `bzip2(1)`: https://manpages.debian.org/bookworm/bzip2/bzip2.1.en.html
- `xz(1)`: https://manpages.debian.org/bookworm/xz-utils/xz.1.en.html
- `zip(1)` / `unzip(1)`: https://manpages.debian.org/bookworm/zip/zip.1.en.html • https://manpages.debian.org/bookworm/unzip/unzip.1.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
