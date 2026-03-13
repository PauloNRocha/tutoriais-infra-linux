# Guia de Produção: Recuperação de zona BIND9 com DNSSEC após erro de sintaxe e impacto em DKIM/DMARC

*Criado em: 13 de março de 2026*

Resolvi registrar este guia depois de um incidente real em produção. O problema começou com o cPanel reclamando de divergência em registros de e-mail, especialmente DMARC e DKIM. Ao corrigir isso manualmente no servidor DNS, a zona acabou sendo salva com erro de sintaxe, o BIND deixou de carregar a zona assinada com DNSSEC e o domínio passou a responder `SERVFAIL`.

O objetivo aqui é documentar um procedimento de recuperação para esse cenário. Não é um guia de implantação inicial de BIND ou de DNSSEC. É um roteiro para quando a zona já existe, usa `inline-signing` e para de carregar.

Veja como implantar o BIND9: ./guia_producao_bind9_debian13.md

Nos exemplos abaixo, vou usar um domínio fictício para facilitar reaproveitamento:

- domínio: `exemplo.com.br`
- arquivo principal da zona: `/var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts`

---

## Índice rápido

1. [O que realmente aconteceu](#1-o-que-realmente-aconteceu)
2. [Sintoma observado](#2-sintoma-observado)
3. [Diagnóstico inicial](#3-diagnóstico-inicial)
4. [O que o DNSSEC com inline-signing muda](#4-o-que-o-dnssec-com-inline-signing-muda)
5. [Procedimento de recuperação](#5-procedimento-de-recuperação)
6. [Verificações pós-recuperação](#6-verificações-pós-recuperação)
7. [O que esse incidente ensina](#7-o-que-esse-incidente-ensina)
8. [Linha do tempo resumida do incidente](#8-linha-do-tempo-resumida-do-incidente)
9. [Referências](#9-referências)

---

## 1. O que realmente aconteceu

A sequência do incidente foi esta:

1. o cPanel começou reclamando de divergência em registros ligados a e-mail;
2. ao validar, ficou claro que o DMARC publicado no DNS estava diferente do que o cPanel esperava;
3. a zona foi editada manualmente no servidor autoritativo;
4. nessa edição, o conteúdo do registro entrou de forma errada, em uma linha só;
5. quando o BIND releu a zona, ele deixou de carregá-la;
6. o `ns1` passou a responder `SERVFAIL`;
7. o `ns2` continuou servindo estado antigo da zona assinada;
8. depois da limpeza dos arquivos auxiliares do DNSSEC e do reinício do serviço, o BIND recriou os arquivos, o `ns2` recebeu a cópia nova e o cPanel voltou a reconhecer a configuração.

Em outras palavras: o incidente começou com DMARC/DKIM, mas o problema final virou carregamento de zona com DNSSEC.

---

## 2. Sintoma observado

O sintoma mais claro foi `SERVFAIL` na zona.

Exemplo:

```bash
dig exemplo.com.br SOA @ns1.exemplo.com.br
```

Resposta típica:

```text
status: SERVFAIL
```

Como o incidente começou na camada de e-mail, também faz sentido testar logo os registros críticos:

```bash
dig default._domainkey.exemplo.com.br TXT @ns1.exemplo.com.br
dig _dmarc.exemplo.com.br TXT @ns1.exemplo.com.br
```

Se houver mais de um nameserver autoritativo, teste os dois separadamente:

```bash
dig exemplo.com.br SOA @ns1.exemplo.com.br
dig exemplo.com.br SOA @ns2.exemplo.com.br
```

No caso que motivou este guia, o `ns1` já estava com `SERVFAIL`, enquanto o `ns2` ainda aparecia com estado antigo da zona assinada.

---

## 3. Diagnóstico inicial

O primeiro passo é validar o arquivo principal da zona com o utilitário do próprio BIND:

```bash
named-checkzone exemplo.com.br \
/var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts
```

Se houver erro de sintaxe, a saída costuma ser parecida com esta:

```text
syntax error
zone exemplo.com.br/IN: loading from master file failed
```

Isso já confirma que o BIND não conseguiu carregar a zona por causa do arquivo principal.

Observação:

- segundo a documentação oficial do BIND, o `named-checkzone` faz as mesmas verificações básicas de carregamento que o `named`;
- por isso ele deve vir antes de qualquer restart, reload ou tentativa de “ver se voltou”.

---

## 4. O que o DNSSEC com inline-signing muda

Quando a zona usa `inline-signing`, o BIND mantém arquivos auxiliares além do arquivo principal.

Estrutura típica:

```text
exemplo.com.br.hosts
exemplo.com.br.hosts.jnl
exemplo.com.br.hosts.signed
exemplo.com.br.hosts.signed.jnl
exemplo.com.br.hosts.jbk
```

Função prática:

| Arquivo | Função |
| --- | --- |
| `exemplo.com.br.hosts` | arquivo principal da zona |
| `exemplo.com.br.hosts.jnl` | journal da zona principal |
| `exemplo.com.br.hosts.signed` | cópia assinada da zona |
| `exemplo.com.br.hosts.signed.jnl` | journal da zona assinada |
| `exemplo.com.br.hosts.jbk` | arquivo auxiliar/backup temporário |

Quando o arquivo principal quebra, esses arquivos podem ficar inconsistentes com o novo estado da zona. A correção da sintaxe, sozinha, nem sempre basta. Em muitos casos você precisa tirar esses arquivos auxiliares do caminho para forçar a reconstrução limpa da zona assinada.

---

## 5. Procedimento de recuperação

### 5.1 Criar diretório de segurança para o incidente

```bash
mkdir -p /root/dns-recovery-example
```

### 5.2 Parar o BIND

```bash
systemctl stop bind9
```

### 5.3 Guardar uma cópia do arquivo principal da zona

```bash
cp -a /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts \
/root/dns-recovery-example/
```

Isso é importante porque o arquivo principal foi justamente o ponto onde o erro entrou.

### 5.4 Mover os arquivos auxiliares da zona assinada

```bash
mv -v /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts.jnl /root/dns-recovery-example/
mv -v /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts.signed /root/dns-recovery-example/
mv -v /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts.signed.jnl /root/dns-recovery-example/
mv -v /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts.jbk /root/dns-recovery-example/
```

Motivo:

- esses arquivos serão recriados automaticamente;
- mover é mais seguro do que apagar direto;
- se algo der errado, você ainda consegue comparar com o estado anterior.

### 5.5 Corrigir o erro de sintaxe na zona

```bash
nano /var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts
```

### 5.6 Validar novamente a zona antes de subir o serviço

```bash
named-checkzone exemplo.com.br \
/var/lib/bind/master-aut/exemplo.com.br/exemplo.com.br.hosts
```

Saída esperada:

```text
zone exemplo.com.br/IN: loaded serial XXXXX
OK
```

Se ainda falhar aqui, não adianta subir o serviço. Corrija a zona primeiro.

### 5.7 Validar a configuração geral do BIND

```bash
named-checkconf
```

Se aparecer algo como:

```text
option 'dnssec-update-mode' is obsolete
```

isso não costuma ser a causa direta do `SERVFAIL`, mas é sinal de configuração antiga que vale limpar depois do incidente.

### 5.8 Iniciar novamente o BIND

```bash
systemctl start bind9
```

### 5.9 Confirmar se a zona voltou a responder no primário

```bash
dig exemplo.com.br SOA @ns1.exemplo.com.br
```

Resposta esperada:

```text
exemplo.com.br. 86400 IN SOA ns1.exemplo.com.br. hostmaster.exemplo.com.br.
```

### 5.10 Confirmar se o secundário recebeu a zona nova

```bash
dig exemplo.com.br SOA @ns1.exemplo.com.br
dig exemplo.com.br SOA @ns2.exemplo.com.br
```

Aqui o objetivo é comparar:

- se os dois nameservers respondem sem `SERVFAIL`;
- se o serial da SOA no `ns2` já acompanha o `ns1`.

Se o `ns2` continuar servindo serial antigo, chave antiga ou estado antigo da zona assinada, o incidente ainda não terminou.

---

## 6. Verificações pós-recuperação

### 6.1 Verificar registros críticos

Como o incidente começou por causa de e-mail, vale checar primeiro DKIM e DMARC:

```bash
dig default._domainkey.exemplo.com.br TXT @ns1.exemplo.com.br
dig _dmarc.exemplo.com.br TXT @ns1.exemplo.com.br
```

### 6.2 Verificar status da zona no `rndc`

```bash
rndc zonestatus exemplo.com.br
```

Exemplo de saída:

```text
name: exemplo.com.br
type: primary
secure: yes
inline signing: yes
```

Isso ajuda a confirmar:

- zona carregada;
- DNSSEC ainda ativo;
- `inline-signing` funcionando.

### 6.3 Verificar DNSSEC

```bash
dig +dnssec exemplo.com.br SOA
dig DS exemplo.com.br @a.dns.br
dig DNSKEY exemplo.com.br @ns1.exemplo.com.br
dig DNSKEY exemplo.com.br @ns2.exemplo.com.br
```

Se o DS do pai continuar compatível com a DNSKEY da zona, a cadeia DNSSEC permanece íntegra.

Se `ns1` e `ns2` responderem chaves diferentes, ainda existe divergência entre os autoritativos.

---

## 7. O que esse incidente ensina

Esse incidente deixa três lições bem claras:

1. problema “de e-mail” pode rapidamente virar problema de zona inteira;
2. em zona com DNSSEC e `inline-signing`, corrigir a sintaxe pode não bastar sem limpar os arquivos auxiliares;
3. sempre que mexer em zona manualmente, valide antes com `named-checkzone`.

---

## 8. Linha do tempo resumida do incidente

1. o cPanel começou reclamando de divergência em DMARC/DKIM;
2. ao comparar cPanel e DNS, ficou clara a diferença no conteúdo publicado;
3. a zona foi editada manualmente no servidor DNS;
4. a correção entrou com sintaxe errada;
5. no reload/restart do BIND, a zona deixou de carregar;
6. o `ns1` passou a responder `SERVFAIL`;
7. depois de corrigir a zona, o `named-checkzone` validou o arquivo;
8. ainda assim o `ns2` continuava com estado antigo da zona assinada;
9. os arquivos auxiliares do DNSSEC foram movidos para fora;
10. o BIND recriou os arquivos, o `ns2` recebeu a cópia nova e o cPanel voltou a reconhecer a configuração.

---

## 9. Referências

- BIND 9 Administrator Reference Manual: https://bind9.readthedocs.io
- `named-checkzone` / `named-compilezone`: https://bind9.readthedocs.io/en/stable/manpages.html#named-checkzone-zone-file-validation-tool
- BIND 9 Configuration Reference (`inline-signing`): https://bind9.readthedocs.io/en/v9.16.44/reference.html
- `rndc` / `zonestatus`: https://bind9.readthedocs.io/en/stable/manpages.html

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
