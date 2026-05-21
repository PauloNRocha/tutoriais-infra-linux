# Guia Prático: Geração de Senhas com pwgen no Linux

*Criado em: 22 de dezembro de 2025*  
*Última atualização em: 21 de maio de 2026*

Senha temporária, usuário novo, acesso de emergência e credencial de laboratório são situações em que muita gente acaba improvisando. O `pwgen` ajuda justamente nisso: gerar senhas rapidamente no terminal, com opções para deixar o resultado mais fácil de digitar ou mais forte para uso real.

O ponto mais importante deste guia é simples: para senha de sistema, serviço, banco, VPN, painel ou qualquer coisa sensível, use o modo seguro com `-s`. O modo padrão do `pwgen` gera senhas pronunciáveis, boas para leitura e digitação, mas não é o melhor caminho para credenciais críticas.

---

## Índice rápido

1. [O que é o pwgen](#1)
2. [Instalação](#2)
3. [Funcionamento básico](#3)
4. [Modo pronunciável e modo seguro](#4)
5. [Opções principais](#5)
6. [Comprimento recomendado](#6)
7. [Exemplos práticos](#7)
8. [Cuidados com histórico, arquivos e prints](#8)
9. [Quando não usar o pwgen](#9)
10. [Comparação com outras ferramentas](#10)
11. [Referências](#referencias)

---

<a id="1"></a>
## 1. O que é o pwgen

O `pwgen` é uma ferramenta de linha de comando para gerar senhas automaticamente no Linux.

Ele pode ser útil para:

- criar senha temporária para usuário novo;
- gerar senha de laboratório;
- gerar senha inicial que será trocada no primeiro login;
- criar credenciais aleatórias para serviços internos;
- gerar várias opções rapidamente durante uma manutenção.

O `pwgen` é apenas um gerador. Ele não guarda senha, não organiza credenciais e não substitui um cofre como Bitwarden, KeePass, Vault ou solução equivalente.

---

<a id="2"></a>
## 2. Instalação

### Debian / Ubuntu

```bash
sudo apt update
sudo apt install -y pwgen
```

### AlmaLinux / Rocky / RHEL

Em distribuições da família RHEL, normalmente o pacote fica no EPEL:

```bash
sudo dnf install -y epel-release
sudo dnf install -y pwgen
```

### Verificar instalação

```bash
pwgen --version
```

---

<a id="3"></a>
## 3. Funcionamento básico

A sintaxe básica é:

```text
pwgen [opções] TAMANHO [QUANTIDADE]
```

Exemplo:

```bash
pwgen 12 5
```

Isso gera 5 senhas com 12 caracteres.

Se você rodar apenas:

```bash
pwgen 12
```

o comportamento pode mudar conforme o terminal. Em uso interativo, ele costuma mostrar uma tela cheia de senhas para você escolher uma. Quando a saída é redirecionada para outro comando ou arquivo, o comportamento é diferente e normalmente gera uma quantidade menor.

Essa diferença é normal e está documentada na manpage do `pwgen`.

---

<a id="4"></a>
## 4. Modo pronunciável e modo seguro

### 4.1 Senhas pronunciáveis

Sem `-s`, o `pwgen` tenta gerar senhas mais fáceis de ler, digitar e memorizar.

```bash
pwgen 12 5
```

Exemplo de saída:

```text
aiYeif3ohs ahcai4IeTh aeCei7quoh rooyooB2ei Eewosu3yee
```

Esse modo pode ser útil para senha temporária que será digitada manualmente e trocada logo depois.

Não use esse modo para senhas de produção, porque ele segue padrões para facilitar leitura humana.

### 4.2 Senhas seguras com `-s`

Para gerar senhas completamente aleatórias, use `-s`:

```bash
pwgen -s 20 3
```

Exemplo de saída:

```text
AdLNHI1pB4iCa0d8uJqa hL1jF0fB0YJ5g4rX bKHe132ftZtb6A8BhtR
```

Para credenciais sensíveis, esse deve ser o modo preferencial.

> [!CAUTION]
> A própria manpage do `pwgen` alerta que senhas geradas sem `-s` não devem ser usadas onde a senha possa sofrer ataque offline de força bruta.

---

<a id="5"></a>
## 5. Opções principais

| Flag | Nome | O que faz | Observação prática |
| :--- | :--- | :--- | :--- |
| `-s` | `--secure` | Gera senha completamente aleatória | Use para produção |
| `-y` | `--symbols` | Inclui pelo menos um caractere especial | Útil para banco, painel, API e serviços |
| `-n` | `--numerals` | Inclui pelo menos um número | Normalmente já vem no uso interativo, mas pode ser explícito |
| `-c` | `--capitalize` | Inclui pelo menos uma letra maiúscula | Normalmente já vem no uso interativo, mas pode ser explícito |
| `-B` | `--ambiguous` | Evita caracteres ambíguos, como `l`, `1`, `0`, `O` | Bom para senha digitada manualmente, mas reduz o espaço de possibilidades |
| `-1` | uma por linha | Imprime uma senha por linha | Bom para copiar, revisar ou usar em scripts |
| `-N` | `--num-passwords` | Define a quantidade de senhas | Alternativa explícita ao último argumento |
| `-v` | `--no-vowels` | Remove vogais e números confundíveis com vogais | Use só se houver motivo específico; não é “remover ambíguos” |

> [!NOTE]
> Se a intenção for evitar caracteres ambíguos, a opção correta é `-B`, não `-v`.

### Referência rápida das opções

Essa tabela resume as opções exibidas por `pwgen --help`, com a tradução prática para consulta durante o uso.

| Opção | Função |
| :--- | :--- |
| `-c` / `--capitalize` | Inclui pelo menos uma letra maiúscula |
| `-A` / `--no-capitalize` | Não inclui letras maiúsculas |
| `-n` / `--numerals` | Inclui pelo menos um número |
| `-0` / `--no-numerals` | Não inclui números |
| `-y` / `--symbols` | Inclui pelo menos um símbolo especial |
| `-r <chars>` / `--remove-chars=<chars>` | Remove caracteres específicos do conjunto usado para gerar a senha |
| `-s` / `--secure` | Gera senhas completamente aleatórias |
| `-B` / `--ambiguous` | Não usa caracteres ambíguos, como `l`, `1`, `0` e `O` |
| `-h` / `--help` | Mostra a ajuda do comando |
| `-H` / `--sha1=arquivo[#seed]` | Usa o hash SHA1 de um arquivo como gerador não tão aleatório |
| `-C` | Mostra as senhas em colunas |
| `-1` | Mostra uma senha por linha, sem colunas |
| `-v` / `--no-vowels` | Não usa vogais, para evitar palavras indesejadas por acidente |

Na prática, para senha sensível, comece por `-s` e só adicione outras opções quando houver necessidade real.

---

<a id="6"></a>
## 6. Comprimento recomendado

Esses tamanhos não são regra absoluta, mas funcionam bem como referência prática:

| Uso | Tamanho mínimo | Exemplo |
| :--- | :--- | :--- |
| Senha temporária de usuário | 14 a 16 caracteres | `pwgen -s 16 1` |
| Servidor, painel ou sistema interno | 20 caracteres | `pwgen -s 20 1` |
| Banco de dados, API ou integração | 24 caracteres | `pwgen -s -y 24 1` |
| Backup, cofre, chave sensível ou credencial muito crítica | 32 caracteres ou mais | `pwgen -s -y 32 1` |

Na prática, comprimento costuma ajudar mais do que tentar enfiar todo tipo de símbolo em senha curta.

---

<a id="7"></a>
## 7. Exemplos práticos

### Senha segura para usuário novo

```bash
pwgen -s 16 1
```

Depois, em Linux, você pode forçar troca no próximo login:

```bash
sudo chage -d 0 nome_usuario
```

### Senha para banco de dados

```bash
pwgen -s -y 24 1
```

### Senha longa para serviço ou API

```bash
pwgen -s -y 32 1
```

### Senha forte sem símbolos

Alguns sistemas antigos, painéis, strings de conexão, arquivos `.env`, YAML ou scripts podem lidar mal com símbolos especiais. Nesses casos, prefira aumentar bastante o tamanho da senha em vez de forçar símbolos.

```bash
pwgen -s 32 1
```

Se o sistema aceita símbolos, mas algum caractere específico causa problema, use `-r` com cuidado. Exemplo removendo apenas o cifrão:

```bash
pwgen -s -y -r '$' 32 1
```

Use isso apenas quando houver uma limitação real do sistema. Quando símbolos forem aceitos sem problema, `-y` continua sendo uma boa opção.

### Senha segura, mas melhor para digitar manualmente

Use `-B` quando você precisa passar a senha por telefone, chat, anotação temporária ou digitação manual e quer evitar confusão entre caracteres parecidos.

```bash
pwgen -s -B 20 1
```

### Lista de senhas, uma por linha

```bash
pwgen -s -1 20 10
```

### Salvar lista temporária em arquivo protegido

Evite salvar senha em arquivo. Se for realmente necessário, ajuste o `umask` antes:

```bash
umask 077
pwgen -s -1 20 10 > senhas.txt
ls -l senhas.txt
```

O arquivo deve ficar legível apenas pelo dono.

---

<a id="8"></a>
## 8. Cuidados com histórico, arquivos e prints

Gerar uma senha forte é só metade do trabalho. O vazamento normalmente acontece depois: print, log, arquivo salvo em local errado, terminal compartilhado ou histórico.

Evite:

- salvar lista de senha em arquivo sem permissão restrita;
- mandar senha em grupo ou canal público;
- tirar print de tela com várias senhas geradas;
- reutilizar a mesma senha em serviços diferentes;
- deixar senha colada em histórico, chamado, wiki ou bloco de notas sem proteção.

Quando for gerar senha para outra pessoa, prefira senha temporária e force troca no primeiro login quando o sistema permitir.

---

<a id="9"></a>
## 9. Quando não usar o pwgen

O `pwgen` não é a melhor ferramenta para tudo.

Não use como substituto de:

- **gerenciador de senhas:** use Bitwarden, KeePass, Vault ou ferramenta equivalente;
- **chave SSH:** use `ssh-keygen`;
- **chave GPG:** use `gpg`;
- **hash de senha para `/etc/shadow`:** use ferramenta própria para hash, como `mkpasswd`, quando aplicável;
- **segredo gerenciado por aplicação:** prefira secret manager, variável protegida, arquivo com permissão restrita ou mecanismo nativo da plataforma.

Também evite `pwgen -H` como método padrão para gerar senhas sensíveis. Esse modo usa o hash SHA1 de um arquivo como base, ou seja, é determinístico, não gera senhas tão aleatórias quanto o modo seguro e ainda pode deixar pistas no histórico do shell.

---

<a id="10"></a>
## 10. Comparação com outras ferramentas

| Ferramenta | Uso principal |
| :--- | :--- |
| `pwgen` | Geração rápida de senhas no terminal |
| `openssl rand` | Geração de bytes ou strings aleatórias, útil em scripts e integrações |
| `mkpasswd` | Geração de hash de senha, não senha em texto puro para uso direto |
| `pass` | Gerenciador de senhas em linha de comando |
| Bitwarden / KeePass | Cofre de senhas para uso pessoal ou equipe |

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### Manpages / Debian

- `pwgen(1)`: https://manpages.debian.org/trixie/pwgen/pwgen.1.en.html
- Pacote `pwgen` no Debian: https://packages.debian.org/trixie/pwgen

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
