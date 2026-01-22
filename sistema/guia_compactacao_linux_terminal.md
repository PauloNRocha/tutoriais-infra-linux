# Guia Prático: Como Compactar e Extrair Arquivos no Terminal Linux

*Criado em 08 de dezembro de 2025*

Compactar e extrair arquivos e pastas pelo terminal é uma das tarefas mais comuns no dia a dia de quem usa Linux. Como existem vários formatos, decidi criar um guia rápido para mostrar como usar os mais populares: `.tar.gz`, `.tar.bz2`, `.tar.xz` e o universal `.zip`.

---

## Índice rápido
1. [Comparando os Formatos de Compressão](#1)
2. [Como Compactar uma Pasta](#2)
3. [Como Listar o Conteúdo (sem extrair)](#3)
4. [Como Extrair Arquivos](#4)
5. [Dica Extra: Extraindo para um Diretório Específico](#5)

---

<a id="1"></a>
## 1. Comparando os Formatos de Compressão

| Formato      | Extensão      | Equilíbrio                                    | Uso Comum                               |
| :----------- | :------------ | :-------------------------------------------- | :-------------------------------------- |
| **gzip**     | `.tar.gz`     | Rápido, boa compressão (o mais comum).        | Pacotes de software, logs, backups rápidos. |
| **bzip2**    | `.tar.bz2`    | Mais lento, compressão melhor que o gzip.     | Distribuições de código-fonte.          |
| **xz**       | `.tar.xz`     | Muito lento, a melhor taxa de compressão.     | Lançamentos de kernel, arquivamento de longo prazo. |
| **zip**      | `.zip`        | Rápido, compressão razoável.                  | Compatibilidade máxima, especialmente com Windows. |

---

<a id="2"></a>
## 2. Como Compactar uma Pasta

Nestes exemplos, vamos compactar uma pasta chamada `meus_arquivos` localizada em `~/documentos/`.

### Compactando com `tar` (`.gz`, `.bz2`, `.xz`)
O `tar` é o "empacotador", ele agrupa os arquivos. As flags `-z`, `-j` e `-J` aplicam a compressão.

-   `-c`: **c**ria um novo arquivo.
-   `-v`: **v**erbaliza o processo (mostra os arquivos sendo adicionados).
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
O `zip` é uma ferramenta separada, ideal para compatibilidade.
- `-r`: **r**ecursivo (inclui subpastas).

```bash
zip -r meus_arquivos.zip ~/documentos/meus_arquivos/
```

---

<a id="3"></a>
## 3. Como Listar o Conteúdo (sem extrair)

Às vezes, você só quer espiar o que tem dentro de um arquivo compactado.

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

<a id="4"></a>
## 4. Como Extrair Arquivos

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

<a id="5"></a>
## 5. Dica Extra: Extraindo para um Diretório Específico

Por padrão, os arquivos são extraídos na pasta atual. Para mudar isso:

- **Para arquivos `.tar.*`**, use a flag `-C` (maiúscula) seguida do caminho de destino:
  ```bash
  tar -xzvf meus_arquivos.tar.gz -C /tmp/destino/
  ```

- **Para arquivos `.zip`**, use a flag `-d` (de **d**iretório):
  ```bash
  unzip meus_arquivos.zip -d /tmp/destino/
  ```
Com estes comandos, você consegue gerenciar a maioria das suas necessidades de compactação e extração diretamente do terminal.

---

## Referências (fontes para consulta)

### Manpages (Debian)

- `tar(1)`: https://manpages.debian.org/bookworm/tar/tar.1.en.html
- `gzip(1)`: https://manpages.debian.org/bookworm/gzip/gzip.1.en.html
- `bzip2(1)`: https://manpages.debian.org/bookworm/bzip2/bzip2.1.en.html
- `xz(1)`: https://manpages.debian.org/bookworm/xz-utils/xz.1.en.html
- `zip(1)` / `unzip(1)`: https://manpages.debian.org/bookworm/zip/zip.1.en.html • https://manpages.debian.org/bookworm/unzip/unzip.1.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
