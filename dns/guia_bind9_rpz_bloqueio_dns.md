# Guia de Produção: Bloqueio de Domínios com BIND9 e RPZ

*Criado em: 04 de dezembro de 2025*  
*Última atualização em: 15 de maio de 2026*

RPZ é útil quando o resolvedor DNS interno precisa aplicar uma política simples de bloqueio, sem depender de proxy ou configuração em cada estação. Na prática, eu uso esse recurso para bloquear domínios conhecidos, testar redirecionamento para página de aviso e confirmar rapidamente se o filtro está funcionando.

O ponto mais importante: RPZ atua no **resolvedor recursivo**. Se o BIND estiver rodando apenas como DNS autoritativo público, com `recursion no;`, essa política não vai filtrar a navegação dos clientes da rede.

Pré-requisito:

- você já deve ter um servidor BIND9 funcionando como resolvedor recursivo para sua rede interna;
- a recursão deve estar liberada apenas para redes confiáveis;
- se quiser a implantação completa do BIND9 em produção, veja: [guia de produção do BIND9 no Debian 13](./guia_producao_bind9_debian13.md);
- se sua ideia for um resolvedor dedicado com RPZ mais completo, veja também: [guia de produção do Unbound no Debian 13](./guia_producao_unbound_debian13.md).

---

## Índice rápido
1. [O que é RPZ?](#1)
2. [Passo 1: Habilitar a Response Policy no BIND9](#2)
3. [Passo 2: Declarar a nossa Zona de Bloqueio](#3)
4. [Passo 3: Criar o Arquivo da Lista de Bloqueio](#4)
5. [Passo 4: Validar e Aplicar as Configurações](#5)
6. [Passo 5: Testando o Bloqueio](#6)
7. [Operação diária](#7)
8. [Checklist final](#8)

---

<a id="1"></a>
## 1. O que é RPZ?

De forma simples, RPZ funciona como uma camada de política em cima da recursão DNS. Quando um cliente consulta um domínio que está na sua lista de bloqueio, o BIND não devolve a resposta real da internet. Em vez disso, ele devolve a ação que você definiu:

- `NXDOMAIN`, como se o domínio não existisse;
- um endereço IP interno, se você quiser redirecionar para página de aviso;
- ou outra política suportada pelo RPZ.

Por padrão, as ações RPZ são aplicadas em consultas recursivas. Por isso, não trate este procedimento como receita para servidor autoritativo público. Em provedor, o desenho mais seguro costuma ser separar as funções:

- autoritativo público: responde apenas suas zonas públicas;
- recursivo interno: atende clientes autorizados e aplica RPZ, quando necessário.

---

<a id="2"></a>
## 2. Passo 1: Habilitar a Response Policy no BIND9

A primeira coisa a fazer é dizer ao BIND para usar a zona de bloqueio que você vai criar.

Edite o arquivo de opções principal do BIND:
```bash
sudo nano /etc/bind/named.conf.options
```

Dentro do bloco `options { ... }`, adicione a diretiva abaixo.

Se você usa `view`, coloque o `response-policy` dentro da **view recursiva** que atende seus clientes internos.

```conf
options {
    // ... suas outras opções aqui ...

    response-policy {
        zone "rpz.local" log yes;
    };

    // ... restante das suas opções ...
};
```

Isso habilita a Response Policy, usa a zona `rpz.local` como fonte das regras e deixa o log de reescrita explícito.

---

<a id="3"></a>
## 3. Passo 2: Declarar a nossa Zona de Bloqueio

Agora o BIND precisa saber onde está o arquivo da zona `rpz.local`.

Edite o arquivo de zonas locais:
```bash
sudo nano /etc/bind/named.conf.local
```

Adicione a declaração da zona no final do arquivo:
```conf
zone "rpz.local" {
    type primary;
    file "/etc/bind/db.rpz.local";
    allow-query { localhost; }; // Apenas o servidor precisa consultar isso
};
```

O que esse bloco faz:

- `type primary`: a política será mantida localmente nesta máquina;
- `file "/etc/bind/db.rpz.local"`: define o arquivo onde ficarão as regras;
- `allow-query { localhost; }`: evita consulta direta da zona RPZ por clientes externos.

Observação: em BIND mais antigo, você pode encontrar exemplos usando `type master;`. Em versões atuais, prefira `type primary;`.

---

<a id="4"></a>
## 4. Passo 3: Criar o Arquivo da Lista de Bloqueio

Aqui fica a parte principal da política. É nesse arquivo que você vai colocar os domínios que quer bloquear ou redirecionar.

```bash
sudo nano /etc/bind/db.rpz.local
```

Adicione o seguinte conteúdo.

Os domínios abaixo são exemplos de documentação. Em produção, troque pelos domínios reais da sua política.

```dns
; Arquivo de Zona de Política de Resposta (RPZ)
;
$TTL 60
$ORIGIN rpz.local.
@       IN      SOA     localhost. root.localhost. (
                        2026051501 ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        60 )      ; Negative Cache TTL
;
@       IN      NS      localhost.

; --- REGRAS DE BLOQUEIO ---

; Exemplo 1: bloquear 'example.com' completamente.
; A resposta será NXDOMAIN (domínio não existe).
example.com             CNAME   .
www.example.com         CNAME   .

; Exemplo 2: bloquear 'example.net' e todos os seus subdomínios.
example.net             CNAME   .
*.example.net           CNAME   .

; Exemplo 3: redirecionar 'example.org' para uma página de aviso.
; Troque 192.0.2.100 pelo IP real do seu servidor web de aviso.
example.org             A       192.0.2.100
www.example.org         A       192.0.2.100
```

Explicação das regras:
- `CNAME .`: Esta é a "punição" padrão do RPZ para um bloqueio total. Ela instrui o BIND a responder com `NXDOMAIN`, que para o usuário final é como se o domínio não existisse. É a forma mais limpa de bloquear.
- `A 192.0.2.100`: esta regra responde com um registro `A` (um endereço IPv4). Você pode usar isso para redirecionar o usuário para um servidor web interno que exibe uma mensagem de "Acesso bloqueado".

Quando alterar esse arquivo depois, incremente o `Serial`. Isso facilita a operação, evita confusão em reload e é obrigatório se essa zona algum dia for transferida para outro servidor.

---

<a id="5"></a>
## 5. Passo 4: Validar e Aplicar as Configurações

Antes de recarregar o serviço, vale validar tudo com calma para não colocar erro de sintaxe na produção.

```bash
# Verificar a sintaxe dos arquivos de configuração
sudo named-checkconf

# Verificar a sintaxe do nosso arquivo de zona RPZ
sudo named-checkzone rpz.local /etc/bind/db.rpz.local

# Verificar configuração carregando as zonas declaradas
sudo named-checkconf -z
```
Se os comandos não retornarem nenhuma mensagem de erro, podemos recarregar o BIND9 para aplicar as mudanças:

```bash
sudo systemctl reload bind9
```

Se no seu Debian/Ubuntu o serviço estiver como `named`, use:

```bash
sudo systemctl reload named
```

Depois confira rapidamente o serviço:

```bash
systemctl status bind9 --no-pager
sudo journalctl -u bind9 -n 80 --no-pager
```

Se o serviço estiver como `named`, troque `bind9` por `named` nos comandos acima.

---

<a id="6"></a>
## 6. Passo 5: Testando o Bloqueio

Agora vem a parte que realmente interessa: confirmar se a política entrou em funcionamento.

### Teste 1: domínio bloqueado
```bash
dig @127.0.0.1 example.com A +noall +comments +answer
```
A resposta deve ser `status: NXDOMAIN`.

### Teste 2: domínio redirecionado
```bash
dig @127.0.0.1 example.org A +noall +comments +answer
```
A resposta deve mostrar que o `ANSWER SECTION` contém o registro `A` apontando para `192.0.2.100`.

### Teste 3: domínio normal
```bash
dig @127.0.0.1 debian.org A +short
```
A resposta deve ser normal, com IPs reais. Isso prova que não quebramos a resolução de nomes legítima.

De outro equipamento da rede, troque `@127.0.0.1` pelo IP do seu resolvedor BIND:

```bash
dig @192.0.2.53 example.com A +noall +comments +answer
```

Substitua `192.0.2.53` pelo IP real do seu DNS.

Se esses três testes passarem, o RPZ entrou no ar do jeito esperado.

---

<a id="7"></a>
## 7. Operação diária

Para adicionar novos domínios depois:

1. Edite `/etc/bind/db.rpz.local`.
2. Incremente o `Serial`.
3. Valide a zona.
4. Recarregue apenas a zona RPZ, quando possível.
5. Teste com `dig` apontando explicitamente para o resolvedor.

Comandos:

```bash
sudo named-checkzone rpz.local /etc/bind/db.rpz.local
sudo rndc reload rpz.local
dig @127.0.0.1 DOMINIO_BLOQUEADO A +noall +comments +answer
```

Troque `DOMINIO_BLOQUEADO` pelo domínio que você acabou de inserir na RPZ.

Se o `rndc` não estiver disponível ou não estiver configurado, use o reload do serviço:

```bash
sudo systemctl reload bind9
```

Rollback rápido:

- remova ou comente a regra problemática;
- incremente o `Serial`;
- rode `named-checkzone`;
- recarregue a zona ou o serviço.

---

<a id="8"></a>
## 8. Checklist final

- [ ] O BIND está atuando como resolvedor recursivo apenas para redes confiáveis.
- [ ] O servidor autoritativo público não foi misturado com política RPZ sem necessidade.
- [ ] `response-policy` foi colocado no bloco correto (`options` ou `view` recursiva).
- [ ] A zona `rpz.local` está declarada em `named.conf.local`.
- [ ] `named-checkconf`, `named-checkzone` e `named-checkconf -z` passam sem erro.
- [ ] O teste com `dig @127.0.0.1` confirma `NXDOMAIN` no domínio bloqueado.
- [ ] O teste com domínio normal continua resolvendo.
- [ ] O procedimento de rollback está claro antes de aplicar em produção.

---

## Referências

### BIND / ISC (oficial)

- BIND 9 Reference: https://bind9.readthedocs.io/en/latest/reference.html
- RPZ (Response Policy Zone): https://bind9.readthedocs.io/en/latest/reference.html#response-policy-zone-rpz

### Ferramentas / manpages (Debian)

- `named-checkconf(1)`: https://manpages.debian.org/trixie/bind9-utils/named-checkconf.1.en.html
- `named-checkzone(1)`: https://manpages.debian.org/trixie/bind9-utils/named-checkzone.1.en.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
