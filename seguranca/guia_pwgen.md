# Guia Completo do pwgen

*Criado em 22 de dezembro de 2025*
*Geração de senhas seguras no Linux (passo a passo)*

---

## Índice rápido
1. [O que é o pwgen?](#1)
2. [Instalação](#2)
3. [Funcionamento básico](#3)
4. [Tipos de senhas geradas](#4)
5. [Opções principais (explicadas uma por uma)](#5)
6. [Comprimento recomendado para senhas](#6)
7. [Exemplos práticos de uso](#7)
8. [Erros comuns (o que evitar)](#8)
9. [Quando NÃO usar o pwgen](#9)
10. [Comparação com outras ferramentas](#10)
11. [Boas práticas finais](#11)

---

<a id="1"></a>
## 1. O que é o pwgen?

O **pwgen** é uma ferramenta de linha de comando, rápida e simples, usada para **gerar senhas automaticamente** no Linux.

Ela é amplamente utilizada por administradores de sistema para:
- Criação rápida de senhas para novos usuários.
- Geração de senhas temporárias seguras.
- Automação de scripts que necessitam de credenciais aleatórias.
- Uso em ambientes de laboratório e testes.

O `pwgen` se destaca por gerar dois tipos de senha:
-   **Pronunciáveis:** Mais fáceis de digitar e memorizar (padrão).
-   **Totalmente aleatórias:** Criptograficamente mais fortes e seguras.

> **Importante:** O `pwgen` é um **gerador**, não um gerenciador de senhas (como Bitwarden ou KeePass). Ele não armazena nada.

---

<a id="2"></a>
## 2. Instalação

### Debian / Ubuntu
```bash
sudo apt update
sudo apt install pwgen
```

### AlmaLinux / Rocky / RHEL
O `pwgen` está no repositório EPEL (Extra Packages for Enterprise Linux).
```bash
sudo dnf install epel-release -y
sudo dnf install pwgen -y
```

### Verificando a instalação
```bash
pwgen --version
```
Se o comando retornar a versão, a instalação foi bem-sucedida.

---

<a id="3"></a>
## 3. Funcionamento básico

A sintaxe básica é:
`pwgen [opções] <tamanho_da_senha> [quantidade_de_senhas]`

Se você executar sem nenhuma opção, ele gera uma tela cheia de senhas pronunciáveis para você escolher uma.

```bash
pwgen 12
```
Este comando gera várias senhas de **12 caracteres**. A ideia é que você escolha uma visualmente e as outras sejam descartadas, evitando que a senha fique no histórico do seu terminal. Isso reduz a chance de reutilização acidental e exposição desnecessária.

---

<a id="4"></a>
## 4. Tipos de senhas geradas

### Senhas Pronunciáveis (Padrão)
São senhas que alternam entre consoantes e vogais para serem mais fáceis de ler e digitar.
```bash
pwgen 10 5
```
Saída típica:
```
aiYeif3ohs ahcai4IeTh aeCei7quoh rooyooB2ei Eewosu3yee
```
- **Prós:** Mais fáceis de memorizar e digitar para logins temporários.
- **Contras:** Menos seguras para ambientes críticos por terem um padrão.

### Senhas Totalmente Aleatórias (Modo Seguro)
Para gerar senhas criptograficamente mais fortes, use a flag `-s`.
```bash
pwgen -s 16 5
```
Saída típica:
```
AdLNHI1pB4iCa0d8 uJqahL1jF0fB0YJ5 g4rXbKHe132ftZtH b6A8BhtRj7lqfqi3 6v5XwB08Zl8s0C3y
```
- **Prós:** Muito mais seguras, ideais para produção.
- **Contras:** Difíceis de memorizar e digitar. Perfeitas para copiar e colar.

> **Recomendação:** **Sempre use `-s` para senhas em produção.**

---

<a id="5"></a>
## 5. Opções principais (explicadas uma por uma)

| Flag | Opção                  | Descrição                                                                         |
| :--- | :--------------------- | :-------------------------------------------------------------------------------- |
| `-s` | **Secure**             | Gera senhas totalmente aleatórias, sem padrão. **O mais importante.**              |
| `-y` | **Symbols**            | Inclui pelo menos um caractere especial (como `!@#$%*()`).                           |
| `-n` | **Numbers**            | Inclui pelo menos um número.                                                      |
| `-c` | **Capitalize**         | Inclui pelo menos uma letra maiúscula.                                            |
| `-v` | **No Ambiguous**         | Remove caracteres que podem ser ambíguos (ex: `1` e `l`, `0` e `O`).             |
| `-1` | **One Line**           | Imprime as senhas em uma única coluna, uma por linha. Muito útil para scripts.      |

**Exemplos combinados:**

-   **Segura, com símbolos:**
    ```bash
    pwgen -s -y 16 1
    ```
    Exemplo de saída: `J$8f@Q!2M#kA9zPs`
-   **Segura, sem caracteres ambíguos:**
    ```bash
    pwgen -s -v 16 1
    ```
-   **Para scripts, uma senha por linha:**
    ```bash
    pwgen -s -1 20 5
    ```

---

<a id="6"></a>
## 6. Comprimento recomendado para senhas

| Uso                     | Tamanho Mínimo | Exemplo de Comando (`pwgen`)          |
| :---------------------- | :------------- | :------------------------------------ |
| Usuário comum           | 12 caracteres  | `pwgen -s 12 1`                       |
| Servidor / Sistema      | 16 caracteres  | `pwgen -s -c -n -v 16 1`              |
| Banco de Dados / API    | 20 caracteres  | `pwgen -s -y 20 1`                    |
| Backup / Chave sensível | 24+ caracteres | `pwgen -s -y 32 1`                    |

> **Lembre-se:** O comprimento da senha costuma ser mais importante que a complexidade para resistir a ataques de força bruta.

---

<a id="7"></a>
## 7. Exemplos práticos de uso

### Senha para um novo usuário Linux
*Segura, com maiúsculas, números e sem caracteres ambíguos.*
```bash
pwgen -s -c -n -v 16 1
```

### Senha para um banco de dados
*Bem longa e com símbolos.*
```bash
pwgen -s -y 24 1
```

### Gerar uma lista de 10 senhas para importar em um script
```bash
pwgen -s -1 16 10 > senhas.txt
```

---

<a id="8"></a>
## 8. Erros comuns (o que evitar)

- **Usar `pwgen` sem a flag `-s` em produção.**
- Gerar senhas curtas (menos de 12 caracteres).
- Salvar a senha diretamente no histórico do shell (use `pwgen -1` e copie, ou use `read -s`).
- Expor a senha em logs, prints de tela ou arquivos de configuração não protegidos.
- Reutilizar a mesma senha gerada para múltiplos serviços.

---

<a id="9"></a>
## 9. Quando NÃO usar o pwgen

O `pwgen` é excelente para o que faz, mas **não é a ferramenta certa** para:
-   **Armazenar senhas:** Ele apenas gera, não guarda.
-   **Gerenciamento de credenciais em equipe:** Use um cofre de senhas como Vault ou Bitwarden.
-   **Geração de chaves criptográficas (SSH, GPG):** Use as ferramentas nativas (`ssh-keygen`, `gpg`).

---

<a id="10"></a>
## 10. 🆚 Comparação com outras ferramentas

| Ferramenta   | Uso Principal                                   |
| :----------- | :---------------------------------------------- |
| **pwgen**    | Senhas rápidas, legíveis ou aleatórias. Ótimo para admin. |
| `openssl rand` | Geração de dados criptograficamente fortes. Mais complexo. |
| `mkpasswd`   | Geração de hashes de senhas para uso em sistemas. |
| `pass`       | Gerenciador de senhas de linha de comando seguro. |

---

<a id="11"></a>
## 11. Boas práticas finais

- Use `-s` sempre que a senha for para um sistema em produção.
- Prefira senhas longas (16+ caracteres).
- Gere uma senha diferente e única para cada serviço.
- Use o `pwgen` como um **gerador**, não como um cofre de senhas.
- Em sistemas Linux, combine a geração de senha com políticas de expiração (`sudo chage -d 0 nome_usuario`).

---

## Referências (fontes para consulta)

### Manpages / Debian

- `pwgen(1)`: https://manpages.debian.org/bookworm/pwgen/pwgen.1.en.html
- Pacote no Debian (bookworm): https://packages.debian.org/bookworm/pwgen

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
