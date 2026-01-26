# Guia Prático de Bloqueio de Sites com BIND9 e RPZ (Response Policy Zones)

*Criado em 04 de dezembro de 2025*

Este guia mostra como configurar um filtro de DNS usando **Response Policy Zones (RPZ)** no BIND9 para bloquear domínios maliciosos ou indesejados.

**Pré-requisito:** Você já deve ter um servidor BIND9 instalado e funcionando como um servidor de DNS recursivo para sua rede.

Se você quer uma implantação completa (produção) do BIND9 no Debian 13, veja: **[BIND 9 no Debian 13 — Master/Slave + TSIG + DNSSEC](../dns/guia_producao_bind9_debian13.md)**.

---

## Índice rápido
1. [O que é RPZ?](#1)
2. [Passo 1: Habilitar a Response Policy no BIND9](#2)
3. [Passo 2: Declarar a nossa Zona de Bloqueio](#3)
4. [Passo 3: Criar o Arquivo da Lista de Bloqueio](#4)
5. [Passo 4: Validar e Aplicar as Configurações](#5)
6. [Passo 5: Testando o Bloqueio](#6)

---

<a id="1"></a>
## 1. O que é RPZ?
De forma simples, RPZ é uma "lista de exceções" para o seu DNS. Quando um cliente consulta um domínio que está na sua lista de RPZ, em vez de buscar o endereço real na internet, o BIND9 responde com a "punição" que você definiu: pode ser um erro de "não encontrado" (`NXDOMAIN`), o IP de uma página de aviso, ou outra ação.

---

<a id="2"></a>
## 2. Passo 1: Habilitar a Response Policy no BIND9
A primeira coisa a fazer é dizer ao BIND9 para usar a nossa futura zona de bloqueio.

Edite o arquivo de opções principal do BIND:
```bash
sudo nano /etc/bind/named.conf.options
```

Dentro do bloco `options { ... }`, adicione a seguinte diretiva:

```conf
options {
    // ... suas outras opções aqui ...

    response-policy {
        zone "rpz.local";
    };

    // ... restante das suas opções ...
};
```
Isso habilita a Response Policy e usa a zona `rpz.local` como fonte das regras.

---

<a id="3"></a>
## 3. Passo 2: Declarar a nossa Zona de Bloqueio
Agora, precisamos dizer ao BIND onde encontrar os arquivos dessa zona `rpz.local`.

Edite o arquivo de zonas locais:
```bash
sudo nano /etc/bind/named.conf.local
```

Adicione a declaração da zona no final do arquivo:
```conf
zone "rpz.local" {
    type master;
    file "/etc/bind/db.rpz.local";
    allow-query { localhost; }; // Apenas o servidor precisa consultar isso
};
```
- `type master`: Estamos criando as regras nesta máquina.
- `file "/etc/bind/db.rpz.local"`: Este é o arquivo que conterá a nossa "lista negra" de domínios.
- `allow-query { localhost; }`: Medida de segurança. Ninguém de fora precisa consultar essa zona diretamente.

---

<a id="4"></a>
## 4. Passo 3: Criar o Arquivo da Lista de Bloqueio
Este é o coração do nosso sistema. Vamos criar o arquivo que definimos no passo anterior e adicionar as regras de bloqueio.

```bash
sudo nano /etc/bind/db.rpz.local
```

Adicione o seguinte conteúdo (explicado logo abaixo):

```dns
; Arquivo de Zona de Política de Resposta (RPZ)
;
$TTL 60
@       IN      SOA     localhost. root.localhost. (
                        1         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        60 )      ; Negative Cache TTL
;
@       IN      NS      localhost.

; --- REGRAS DE BLOQUEIO ---

; Exemplo 1: Bloquear 'site-malicioso.com' completamente.
; A resposta será NXDOMAIN (domínio não existe).
site-malicioso.com      CNAME   .

; Exemplo 2: Bloquear 'outro-site-ruim.net' e todos os seus subdomínios.
*.outro-site-ruim.net   CNAME   .

; Exemplo 3: Redirecionar 'site-de-phishing.org' para uma página de aviso.
; (Assumindo que você tem um servidor web em 192.168.1.100 com um aviso)
site-de-phishing.org    A       192.168.1.100
*.site-de-phishing.org  A       192.168.1.100
```

**Explicação das regras:**
- `CNAME .`: Esta é a "punição" padrão do RPZ para um bloqueio total. Ela instrui o BIND a responder com `NXDOMAIN`, que para o usuário final é como se o site não existisse. É a forma mais limpa de bloquear.
- `A 192.168.1.100`: Esta regra responde com um registro `A` (um endereço IPv4). Você pode usar isso para redirecionar o usuário para um servidor web interno que exibe uma mensagem de "Acesso bloqueado".

---

<a id="5"></a>
## 5. Passo 4: Validar e Aplicar as Configurações
Antes de reiniciar o serviço, é crucial verificar se não cometemos nenhum erro de sintaxe.

```bash
# Verificar a sintaxe dos arquivos de configuração
sudo named-checkconf

# Verificar a sintaxe do nosso arquivo de zona RPZ
sudo named-checkzone rpz.local /etc/bind/db.rpz.local
```
Se ambos os comandos não retornarem nenhuma mensagem de erro, podemos recarregar o BIND9 para aplicar as mudanças:

```bash
sudo systemctl reload bind9
```

---

<a id="6"></a>
## 6. Passo 5: Testando o Bloqueio
Agora, a prova final. De uma máquina cliente que usa este servidor DNS, ou do próprio servidor, vamos usar o comando `dig`.

**Teste 1: Domínio bloqueado**
```bash
dig site-malicioso.com
```
A resposta deve ser `status: NXDOMAIN`.

**Teste 2: Domínio redirecionado**
```bash
dig site-de-phishing.org
```
A resposta deve mostrar que o `ANSWER SECTION` contém o registro `A` apontando para `192.168.1.100`.

**Teste 3: Domínio normal**
```bash
dig www.google.com
```
A resposta deve ser a normal, com os IPs do Google. Isso prova que não quebramos a resolução de nomes legítima.

Com isso, seu filtro de DNS está ativo e funcionando. Agora é só adicionar mais domínios ao arquivo `db.rpz.local` conforme a necessidade e recarregar o serviço (`sudo systemctl reload bind9`).

---

## Referências (fontes para consulta)

### BIND / ISC (oficial)

- BIND 9 Reference: https://bind9.readthedocs.io/en/latest/reference.html
- RPZ (Response Policy Zone): https://bind9.readthedocs.io/en/v9.18.31/reference.html#response-policy-zone-rpz

### Ferramentas / manpages (Debian)

- `named-checkconf(1)`: https://manpages.debian.org/trixie/bind9-utils/named-checkconf.1.en.html
- `named-checkzone(1)`: https://manpages.debian.org/trixie/bind9-utils/named-checkzone.1.en.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
