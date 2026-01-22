# Guia Operacional de Produção: Krill RPKI com Nginx e nftables no Debian 13

*Criado em 03 de Janeiro de 2026*
*Última atualização em 08 de janeiro de 2026*

Este guia detalha o processo completo para instalar, configurar e proteger uma Autoridade Certificadora (CA) de RPKI usando o **Krill (NLnet Labs)** em um servidor **Debian 13 (Trixie)**. Focamos em um ambiente de produção seguro, incluindo a configuração de Nginx como proxy reverso para acesso via HTTPS e o uso de `nftables` para o firewall.

> **Objetivo:** Ao final deste tutorial, você terá o Krill rodando como um serviço robusto, com acesso administrativo seguro via SSH tunnel ou HTTPS (Nginx + TLS). Sua CA estará configurada com um Parent (como o Registro.br) e um repositório de publicação funcional, permitindo a criação e validação de ROAs (Route Origin Authorizations) e ASPAs para proteger seus prefixos de rede e o trânsito BGP.

---

## Índice
1. [Arquitetura Recomendada (e Portas)](#1)
2. [Pré‑requisitos e Preparação](#2)
3. [Instalação do Krill (APT NLnet Labs)](#3)
4. [Configuração do Krill (`/etc/krill.conf`)](#4)
5. [Configurando o Acesso à Interface do Krill](#5)
    5.1. [Opção A: SSH Tunnel (Mais Seguro)](#5.1)
    5.2. [Opção B: Nginx + TLS (Produção)](#5.2)
6. [Firewall Obrigatório: nftables (Modelo Pronto)](#6)
7. [Configurar CA + Parent (Exemplo: Registro.br)](#7)
8. [Configurar Repository/Publicação (Repository)](#8)
9. [Criar ROAs (Boas Práticas e Validação)](#9)
10. [ASPA (Autonomous System Provider Authorization) — Opcional](#10)
11. [Garantindo a Estabilidade Pós-Reboot (Systemd)](#11)
12. [Opcional: Publicação Própria (RRDP/rsync)](#12)
13. [Monitoramento e Troubleshooting](#13)
14. [Backup e Recuperação](#14)
15. [Opcional: Multi‑user (Usuários Nomeados)](#15)
16. [Checklist Final](#16)
17. [O que NÃO Fazer (Erros Comuns)](#17)
18. [Referências](#18)

---

<a id="1"></a>
## 1) Arquitetura Recomendada e Portas

### 1.1 Dois Modelos Seguros (Escolha 1)

**Modelo A (Recomendado para Começar):**
- O Krill escuta em `localhost:3000`.
- O acesso administrativo é feito **apenas por SSH tunnel**.
- O firewall abre somente a porta `22/tcp` (e o necessário para o servidor em geral, como respostas de requisições de saída).

**Modelo B (Produção com Acesso Remoto por HTTPS):**
- O Krill escuta em `localhost:3000`.
- O Nginx atua como proxy reverso, expondo a porta `443/tcp` com um certificado TLS válido (obtido via Let’s Encrypt).
- O firewall abre as portas `22/tcp`, `80/tcp` e `443/tcp`. A porta `80` é necessária para a renovação HTTP‑01 do Certbot.

> **Dica:** Mesmo no Modelo B, considere fortemente restringir o acesso à porta `443/tcp` por IP ou VPN, pois a interface web do Krill é uma ferramenta **administrativa** sensível.

### 1.2 Portas (o Mínimo Essencial)
- `22/tcp`: Acesso SSH para administração do servidor.
- `80/tcp`: HTTP, **somente** para o desafio de validação do Let’s Encrypt (HTTP‑01) e redirecionamento para HTTPS.
- `443/tcp`: HTTPS, para a interface web do Krill (atrás do Nginx) e/ou para o serviço RRDP, se você optar por publicar seus certificados via HTTPS.
- `3000/tcp`: Porta interna do Krill (**somente local**, não deve ser exposta diretamente no firewall para a internet).
- `873/tcp`: rsync (**apenas se** você optar por publicar seus certificados via `rsync://...`).

---

<a id="2"></a>
## 2) Pré‑requisitos e Preparação

### 2.1 Requisitos Mínimos
- Sistema Operacional: Debian **13 (Trixie)**.
- Acesso `sudo`: Um usuário com privilégios de administrador.
- DNS: Se você for usar um domínio (ex: `rpki.seudominio.com`) para a interface web ou publicação, ele deve estar configurado para apontar para o IP do seu servidor.

### 2.2 Atualizar o Sistema e Instalar Utilitários Essenciais

É fundamental manter o sistema atualizado e instalar ferramentas básicas que serão utilizadas.

```bash
sudo apt update                  # Atualiza a lista de pacotes disponíveis
sudo apt full-upgrade -y         # Realiza uma atualização completa do sistema
sudo apt install -y ca-certificates curl gnupg lsb-release # Instala ferramentas úteis
```

### 2.3 Sincronização de Tempo (Essencial para RPKI)

A precisão do relógio do sistema é **crítica** para a validade dos certificados RPKI. Instale e configure um cliente NTP.

```bash
sudo apt install -y chrony       # Instala o Chrony, um cliente NTP moderno e eficiente
sudo systemctl enable --now chrony # Habilita e inicia o serviço Chrony
timedatectl status               # Verifica o status da sincronização de tempo
```

---

<a id="3"></a>
## 3) Instalação do Krill (APT NLnet Labs)

Vamos adicionar o repositório oficial do NLnet Labs para obter a versão mais recente do Krill.

### 3.1 Adicionar Keyring do Repositório

Primeiro, adicione a chave GPG do repositório para verificar a autenticidade dos pacotes.

```bash
sudo mkdir -p /usr/share/keyrings
curl -fsSL https://packages.nlnetlabs.nl/aptkey.asc | sudo gpg --dearmor -o /usr/share/keyrings/nlnetlabs-archive-keyring.gpg
```

### 3.2 Adicionar o Repositório APT e Instalar o Krill

Em seguida, adicione o repositório à sua lista de sources e instale o pacote `krill`.

Arquivo: `/etc/apt/sources.list.d/nlnetlabs.list`

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/nlnetlabs-archive-keyring.gpg] https://packages.nlnetlabs.nl/linux/debian $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/nlnetlabs.list > /dev/null
sudo apt update                    # Atualiza a lista de pacotes após adicionar o novo repositório
sudo apt install -y krill          # Instala o serviço Krill RPKI
```

### 3.3 Habilitar e Checar o Serviço

Verifique se o Krill foi instalado corretamente e está rodando.

```bash
sudo systemctl enable --now krill  # Garante que o Krill inicie automaticamente com o sistema
sudo systemctl status krill --no-pager # Verifica o status atual do serviço Krill
krill --version                    # Exibe a versão do Krill instalada
```

Para monitorar os logs do Krill:
```bash
sudo journalctl -u krill -n 100 --no-pager # Exibe as últimas 100 linhas de log do serviço Krill
```

---

<a id="4"></a>
## 4) Configuração do Krill (`/etc/krill.conf`)

O arquivo de configuração principal do Krill é `/etc/krill.conf`. Vamos editá-lo para ajustá-lo ao ambiente de produção.

### 4.1 Abrir o Arquivo e Localizar o Token Inicial

Ao instalar, o Krill gera um `admin_token` inicial. É crucial salvá-lo.

Arquivo: `/etc/krill.conf`

```bash
sudo sed -n '1,220p' /etc/krill.conf # Exibe as primeiras linhas do arquivo para contexto
sudo grep -E '^(admin_token|auth_token)\s*=' /etc/krill.conf # Localiza o token de administrador
```

> **Atenção:** **Guarde este token** em um gerenciador de senhas seguro. Ele é a credencial inicial para acesso administrativo à interface do Krill.

### 4.2 Ajustes Recomendados para Produção

> **Importante:** Antes de fazer qualquer alteração, faça um backup do arquivo de configuração.
> 
> ```bash
> sudo cp -av /etc/krill.conf /etc/krill.conf.bak.$(date +%F-%H%M) # Faz um backup do arquivo de configuração
> ```

Edite o arquivo de configuração.

```bash
sudo nano /etc/krill.conf # Abre o arquivo de configuração para edição
```

Exemplo de configuração (ajuste para o seu ambiente e domínio):
```toml
# Guarda dados/estado do Krill (contém chaves criptográficas importantes)
data_dir = "/var/lib/krill/data/"

# O Krill escutará apenas localmente (recomendado para segurança)
ip = "127.0.0.1"
port = 3000

# URL público FINAL da sua interface Krill (DEVE terminar com /)
service_uri = "https://rpki.seudominio.com/"

# Token de administrador (mantenha forte e NÃO compartilhe)
admin_token = "COLE_UM_TOKEN_FORTE_AQUI" # Substitua pelo token gerado ou um novo e forte

# Configuração HTTPS do Krill:
# - padrão: HTTPS com certificado autoassinado (útil para SSH tunnel)
# - existing: usa certificado já existente no data_dir/ssl/
# - disable: HTTP (USE SOMENTE se atrás de um proxy TLS confiável como Nginx)
#
# https_mode = "disable" # Descomente e ajuste conforme seu modelo de acesso
```

> **ATENÇÃO:** Se você estiver usando um proxy TLS (como Nginx com Let\'s Encrypt) para expor a interface do Krill na porta 443, configure `https_mode = "disable"` no Krill. **NUNCA exponha a UI do Krill diretamente com HTTPS interno (autoassinado) para a internet em produção.** Isso evita avisos de segurança e garante que a criptografia seja gerenciada pelo proxy.

> **Reforço sobre `service_uri`:** Este parâmetro é crucial para o correto funcionamento do RPKI.
> - Deve ser um **URL público** (ex: `https://rpki.seudominio.com/`).
> - Deve usar **HTTPS válido** (o Nginx com Let's Encrypt cuidará disso no Modelo B).
> - **DEVE terminar com uma barra (`/`)**. Muitos problemas com Parents e Repositories ocorrem devido a um `service_uri` incorreto.

Para gerar um novo token forte (se você quiser trocar o padrão):
```bash
openssl rand -base64 48 # Gera uma string aleatória base64 de 48 caracteres
```

### 4.3 Permissões do Arquivo de Configuração

O processo do Krill precisa **ler** o arquivo `/etc/krill.conf`. Para segurança, é ideal que apenas o `root` e o grupo `krill` tenham acesso de leitura.

Verifique as permissões atuais:
```bash
ls -l /etc/krill.conf # Lista as permissões do arquivo de configuração
```

Se as permissões estiverem muito abertas (ex.: leitura para "outros"), ajuste-as:
```bash
sudo chown root:krill /etc/krill.conf # Define o proprietário como root e o grupo como krill
sudo chmod 640 /etc/krill.conf   # Define permissões de leitura/escrita para root, leitura para o grupo e nada para outros
```

### 4.4 Reiniciar o Serviço para Aplicar as Mudanças

Após as alterações no `krill.conf`, reinicie o serviço para que elas sejam aplicadas.

```bash
sudo systemctl restart krill       # Reinicia o serviço Krill
sudo journalctl -u krill -n 50 --no-pager # Verifica os logs após o reinício para garantir que não há erros
```

---

<a id="5"></a>
## 5) Configurando o Acesso à Interface do Krill

A interface web do Krill é uma ferramenta administrativa. É fundamental configurá-la para acesso seguro. Apresentamos duas opções, escolha a que melhor se adapta ao seu cenário.

<a id="5.1"></a>
### 5.1 Opção A: SSH Tunnel (Mais Seguro para Administração Pessoal)

Este método é altamente recomendado, especialmente para administração pessoal, pois evita expor a interface web do Krill diretamente na internet.

No seu computador local (cliente), execute o comando SSH para criar um túnel:
```bash
ssh -L 3000:localhost:3000 usuario@IP_OU_HOST_DO_SERVIDOR # Cria um túnel da porta 3000 local para a 3000 do servidor
```
- `usuario`: Seu nome de usuário SSH no servidor.
- `IP_OU_HOST_DO_SERVIDOR`: O endereço IP ou hostname do seu servidor Krill.

Após estabelecer o túnel, acesse a interface do Krill no seu navegador local:
- Se o Krill estiver configurado com HTTPS padrão (autoassinado): `https://localhost:3000`
- Se você configurou `https_mode = "disable"` no `krill.conf`: `http://localhost:3000`

> **Nota:** Se você usar HTTPS padrão, é normal o navegador exibir um aviso de certificado inválido (pois é autoassinado). Basta aceitar para prosseguir.

<a id="5.2"></a>
### 5.2 Opção B: Nginx + TLS (Acesso em Produção via HTTPS)

Para expor a interface do Krill via HTTPS para uma equipe ou para acesso remoto que não seja via SSH tunnel, o uso de um proxy reverso como o Nginx com TLS (Let's Encrypt) é a forma padrão e segura.

#### 5.2.1 Instalar Nginx e Certbot

Instale o servidor web Nginx e o Certbot, a ferramenta para obter certificados SSL/TLS gratuitos do Let's Encrypt.

```bash
sudo apt install -y nginx certbot python3-certbot-nginx # Instala Nginx, Certbot e o plugin do Certbot para Nginx
sudo systemctl enable --now nginx                       # Habilita e inicia o serviço Nginx
```

#### 5.2.2 Preparar Webroot do Let’s Encrypt

O Certbot precisa de um diretório específico para validar seu domínio via HTTP-01.

```bash
sudo mkdir -p /var/www/letsencrypt/.well-known/acme-challenge # Cria o diretório para o desafio do Certbot
sudo chown -R www-data:www-data /var/www/letsencrypt         # Define permissões para o Nginx acessar o diretório
```

#### 5.2.3 Criar um Site HTTP Mínimo para Emitir o Certificado

Vamos configurar o Nginx para responder na porta 80 e permitir a validação do Let's Encrypt, além de redirecionar todo o tráfego para HTTPS.

Arquivo: `/etc/nginx/sites-available/krill`

```bash
sudo nano /etc/nginx/sites-available/krill # Cria o arquivo de configuração do site Nginx
```

Conteúdo do arquivo (ajuste `rpki.seudominio.com` para o seu domínio):
```nginx
server {
    listen 80;
    server_name rpki.seudominio.com; # Seu domínio aqui

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
    }

    location / {
        return 301 https://$host$request_uri; # Redireciona HTTP para HTTPS
    }
}
```

Ativar a configuração e validar:
```bash
sudo rm -f /etc/nginx/sites-enabled/default # Remove a configuração padrão do Nginx
sudo ln -s /etc/nginx/sites-available/krill /etc/nginx/sites-enabled/krill # Ativa a nova configuração
sudo nginx -t                             # Testa a sintaxe da configuração do Nginx
sudo systemctl reload nginx               # Recarrega o Nginx para aplicar as mudanças
```

> **Nota:** Antes de configurar o HTTPS, o redirecionamento para `https://` pode "quebrar" no navegador, pois a porta 443 ainda não estará configurada. Isso é um comportamento esperado.

#### 5.2.4 Emitir o Certificado TLS (HTTP‑01)

Agora, com o Nginx pronto para o desafio, podemos emitir o certificado SSL/TLS.

> **Importante:** Garanta que a porta `80/tcp` esteja liberada no seu firewall e que o registro DNS do seu domínio (`rpki.seudominio.com`) aponte corretamente para o IP deste servidor.

```bash
sudo certbot certonly --webroot -w /var/www/letsencrypt -d rpki.seudominio.com # Obtém o certificado para o domínio
```

Para checar se a renovação automática está funcionando (simulação):
```bash
sudo certbot renew --dry-run # Simula a renovação do certificado
```

#### 5.2.5 Configurar o Proxy HTTPS para o Krill (Interface Web)

Edite o mesmo arquivo de configuração do Nginx (`/etc/nginx/sites-available/krill`) para adicionar a seção HTTPS.

```bash
sudo nano /etc/nginx/sites-available/krill # Edita o arquivo de configuração do site Nginx
```

Conteúdo **completo** do arquivo (ajuste o domínio):
```nginx
server {
    listen 80;
    server_name rpki.seudominio.com;

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name rpki.seudominio.com;

    ssl_certificate     /etc/letsencrypt/live/rpki.seudominio.com/fullchain.pem; # Caminho para o certificado completo
    ssl_certificate_key /etc/letsencrypt/live/rpki.seudominio.com/privkey.pem;  # Caminho para a chave privada

    # Configurações TLS básicas e seguras
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1h;
    ssl_session_tickets off;
    ssl_dhparam /etc/nginx/dhparam.pem; # Gerar com `sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048`

    # HSTS (HTTP Strict Transport Security) 
    # Use includeSubDomains APENAS se TODOS os subdomínios forem HTTPS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    client_max_body_size 100M; # Limite para uploads (se houver)

    # Bloqueia o acesso ao endpoint /metrics do Krill
    location /metrics {
        deny all;
        return 404; # Retorna Not Found para qualquer tentativa de acesso
    }

    location / {
        # Se o Krill estiver configurado com HTTPS padrão (autoassinado no krill.conf):
        # proxy_pass https://127.0.0.1:3000/; 
        # proxy_ssl_verify off; # Necessário para certificados autoassinados

        # Se você configurou o Krill para HTTP no loopback (https_mode = "disable"):
        proxy_pass http://127.0.0.1:3000/; # Encaminha requisições para o Krill

        proxy_set_header Host $host;             # Preserva o cabeçalho Host original
        proxy_set_header X-Real-IP $remote_addr; # Passa o IP real do cliente
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # Passa a cadeia de proxies
        proxy_set_header X-Forwarded-Proto $scheme; # Indica o protocolo original (HTTPS)

        # Configurações para WebSockets (usado pela UI do Krill)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Valide e recarregue o Nginx para aplicar as mudanças:
```bash
sudo nginx -t                      # Testa a sintaxe da configuração do Nginx
sudo systemctl reload nginx        # Recarrega o Nginx para aplicar as mudanças
```

#### 5.2.6 Nota Importante sobre Exposição Pública

Expor a interface web do Krill via internet, mesmo com HTTPS, significa expor um painel **administrativo crítico**. Considere fortemente as seguintes medidas de segurança:
- **Preferir o SSH tunnel (Opção A):** É o método mais seguro para acesso pessoal, pois seu IP não precisa "ver" o servidor diretamente na porta 443.
- **Restringir acesso no Nginx:** Use diretivas `allow` / `deny` no Nginx para limitar o acesso por IP de origem ou configure uma VPN para sua rede de gerenciamento, permitindo que apenas clientes da VPN acessem a interface.

---

<a id="6"></a>
## 6) Firewall Obrigatório: nftables (Modelo Pronto)

Um firewall bem configurado é essencial para a segurança do seu servidor Krill. Utilizaremos o `nftables`, o substituto moderno do `iptables`.

### 6.1 Instalar e Habilitar nftables

```bash
sudo apt install -y nftables       # Instala o pacote nftables
sudo systemctl enable --now nftables # Habilita e inicia o serviço nftables
```

### 6.2 Regras Básicas (SSH + HTTP/HTTPS) — Com Opção de Restringir SSH

O arquivo de configuração do `nftables` é `/etc/nftables.conf`.

```bash
sudo nano /etc/nftables.conf # Abre o arquivo de configuração para edição
```

Conteúdo do arquivo `/etc/nftables.conf` (modelo base seguro):
```nft
#!/usr/sbin/nft -f
# Limpa todas as regras existentes
flush ruleset

# Define a família de tabelas para IPv4 e IPv6
table inet filter {
  # Define a cadeia de entrada (input)
  chain input {
    type filter hook input priority 0; # Garante que esta cadeia seja a primeira a ser processada
    policy drop;                       # Política padrão: derrubar tudo que não for explicitamente permitido

    # 1) Permite tráfego na interface de loopback (comunicação interna do servidor)
    iif "lo" accept

    # 2) Permite pacotes de conexões já estabelecidas ou relacionadas (essencial para tráfego de saída e respostas)
    ct state established,related accept

    # 3) Permite tráfego ICMP/ICMPv6 (útil para diagnóstico de rede, como ping)
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept

    # 4) Permite acesso SSH na porta 22
    # Para MAIOR SEGURANÇA, restrinja por IP ou rede. Exemplo:
    # tcp dport 22 ip saddr 192.0.2.0/24 accept # Apenas a rede 192.0.2.0/24 pode acessar SSH
    tcp dport 22 accept # Permite SSH de qualquer IP (MENOS SEGURO)

    # 5) Permite acesso HTTP e HTTPS (se você estiver usando Nginx/Let’s Encrypt)
    tcp dport { 80, 443 } accept

    # 6) Permite acesso rsync (opcional, se você for publicar rsync://...)
    # Descomente a linha abaixo e ajuste se necessário
    # tcp dport 873 accept
  }

  # Cadeia de encaminhamento (forward) - normalmente usada para roteadores/gateways
  chain forward {
    type filter hook forward priority 0;
    policy drop; # Nenhuma regra para encaminhamento, tudo é derrubado por padrão
  }

  # Cadeia de saída (output) - permite todo o tráfego de saída do servidor
  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
```

Aplique as regras e valide:
```bash
sudo nft -f /etc/nftables.conf # Carrega as regras do arquivo para o kernel
sudo nft list ruleset          # Lista as regras ativas do firewall
sudo ss -tulpn                 # Lista as portas TCP abertas e os processos associados
```

> **Atenção:** **NUNCA** abra a porta `3000/tcp` para a internet. O Krill deve ser acessado apenas via SSH tunnel ou por um proxy reverso seguro como o Nginx.

---

<a id="7"></a>
## 7) Configurar CA + Parent (Exemplo: Registro.br)

Com o Krill rodando e acessível, o próximo passo é configurar sua Autoridade Certificadora (CA) e estabelecer a comunicação com seu Parent (geralmente o seu RIR/NIR, como o Registro.br no Brasil).

1) Acesse a interface web do Krill (via SSH tunnel ou `https://rpki.seudominio.com`).
2) Faça login utilizando o `admin_token` (ou o usuário multi-user configurado).
3) Crie sua **CA Handle** (um nome interno para identificar sua CA, ex.: `isp-exemplo-br`).

### 7.1 Fluxo com Registro.br (Parent)

Se seu Parent for o Registro.br, siga este fluxo:

1)  **No Krill:** Navegue até a seção "Parents" e clique para gerar/copiar o **Child Request XML**.
2)  **No portal do Registro.br:** Acesse "Titularidade → Seu ASN → Configurar RPKI".
3)  Cole o conteúdo do Child Request XML no campo apropriado e habilite o serviço RPKI.
4)  Copie o **Parent Response XML** gerado pelo Registro.br.
5)  **No Krill:** Cole o conteúdo do Parent Response XML na seção "Parents" e confirme a configuração.

---

<a id="8"></a>
## 8) Configurar Repository/Publicação (Repository)

A forma como seus ROAs serão publicados para o mundo é através de um repositório. A melhor prática é utilizar a **publicação remota** (Hosted Publication), se ela for oferecida pelo seu RIR/NIR.

1)  **No Krill:** Navegue até a seção "Repository" e clique para gerar/copiar o **Publisher Request XML**.
2)  **No portal do seu RIR/NIR:** Habilite a publicação remota e cole o conteúdo do Publisher Request XML.
3)  Copie o **Repository Response XML** gerado.
4)  **No Krill:** Cole o conteúdo do Repository Response XML e confirme a configuração.

--- 

<a id="9"></a>
## 9) Criar ROAs (Boas Práticas e Validação)

Os ROAs (Route Origin Authorizations) são a peça central do RPKI, autorizando quais ASNs (Autonomous System Numbers) podem anunciar quais prefixos IP.

Na interface web do Krill, navegue até a seção **ROAs** e clique em **Add ROA**.

**Recomendações importantes ao criar seus ROAs:**
-   Crie ROAs **somente** para os prefixos IP que você **realmente anuncia** na internet.
-   `maxLength`: Use com extrema cautela.
    -   Se você anuncia um prefixo específico, como `/24`, defina `maxLength = 24`.
    -   Se você anuncia um bloco maior, como `/22`, e nunca o desagrega em prefixos menores, defina `maxLength = 22`.
    -   Se você intencionalmente desagrega um bloco maior em prefixos menores (ex: /22 em /24s), defina `maxLength` conscientemente para o maior prefixo desagregado que você anuncia (ex: `maxLength = 24` para um /22 se você anuncia /24s dele).

### 9.1 Validando seus ROAs Externamente (Checklist Pós-Criação)

Após criar seus ROAs no Krill, é **fundamental** verificar se eles estão sendo propagados e validados corretamente na internet. Use estes validadores externos para confirmar:

-   **Routinator Validator:** Uma ferramenta de validação de RPKI de código aberto.
-   **RIPE NCC RPKI Validator:** Validador online fornecido pelo RIPE NCC.
-   **Qrator Radar / Cloudflare Radar / RIPE RIS:** Ferramentas de observabilidade que podem mostrar o status de validação BGP para seus prefixos.

---

<a id="10"></a>
## 10) ASPA (Autonomous System Provider Authorization) — Opcional

#### PRODUÇÃO AVANÇADA

Enquanto os ROAs validam **quem** pode originar um prefixo, o **ASPA** valida **como** esse prefixo transita pela internet. É um objeto RPKI que complementa o ROA, permitindo a validação de relacionamentos cliente-provedor no `AS-Path` para prevenir sequestros de rotas e vazamentos de trânsito (route leaks).

> **Analogia Rápida:**
> - **ROA diz:** "O prefixo `192.0.2.0/24` só pode ser anunciado pelo `AS65530`."
> - **ASPA diz:** "O `AS65530` (ASN do exemplo) só deve ser visto na internet através dos provedores de trânsito `AS65531` e `AS65532`."

### ASPA via Interface Web (Krill 0.15+)

Embora parte da documentação oficial ainda descreva o gerenciamento de ASPA apenas via CLI ou API, o Krill 0.15.x já oferece suporte completo à criação, edição e remoção de ASPAs diretamente pela interface web.

A configuração pode ser feita acessando:
`CA` → `ASPAs` → `Add ASPA`

> **Nota:** A documentação oficial do Krill ainda pode mencionar ASPA apenas via CLI/API. No entanto, versões recentes (0.15+) já incluem suporte nativo via UI.

### Passos Práticos

1.  Acesse a interface web do Krill.
2.  Entre na sua CA principal (ex: `isp-exemplo-br`).
3.  No menu lateral, selecione **ASPA**.
4.  Clique para criar um novo registro ASPA para o seu ASN.
5.  No campo "Providers", liste **apenas os ASNs dos seus provedores de trânsito (upstreams)**, separados por vírgula. Não inclua ASNs de Pontos de Troca de Tráfego (IXPs) ou de seus clientes.

> **Nota de Segurança (IMPORTANTE):**
> **ATENÇÃO:** A inclusão incorreta de ASNs como `providers` (por exemplo, adicionar um peer de IXP) pode causar validações negativas no futuro e impactar a visibilidade dos seus prefixos. Use ASPA apenas se tiver certeza absoluta dos seus relacionamentos de trânsito.

> **ATENÇÃO:** **ASPA e Mitigação DDoS com Túneis (GRE/IPsec/etc.)**
>
> É fundamental compreender que o ASPA (AS Provider Authorization) avalia exclusivamente o ASN imediatamente adjacente no atributo AS_PATH.
>
> A presença de túneis (GRE, GRE6, IPsec, VXLAN), bem como o uso de sessões BGP multihop, não altera o funcionamento do ASPA, pois esses mecanismos fazem parte apenas da camada de transporte e não influenciam a topologia lógica do BGP.
>
> Serviços de mitigação DDoS (scrubbing centers) devem ser incluídos no objeto ASPA sempre que houver uma sessão BGP direta com o mitigador, de forma que o ASN do mitigador possa aparecer imediatamente acima do seu ASN no AS_PATH.
>
> Isso se aplica mesmo quando a comunicação ocorre por túneis ou multihop, e mesmo que os anúncios de prefixos sejam temporários ou realizados apenas durante eventos de ataque.
>
> Por outro lado, mitigadores que não aparecem como ASN adjacente no AS_PATH, por exemplo, quando operam atrás de um upstream ou por mecanismos transparentes, não devem ser incluídos no ASPA.

### Validando seu Objeto ASPA Externamente

Após criar seu objeto ASPA, é fundamental verificar se ele está sendo propagado e validado corretamente na internet. Você pode usar validadores externos para confirmar:

-   **NLnet Labs RPKI Tools:** Ferramentas online para inspecionar objetos RPKI.
-   **RIPE NCC RPKI Validator:** Validador online do RIPE NCC.
-   **Cloudflare Radar:** Permite inspecionar dados de RPKI para ASNs específicos.

---

<a id="11"></a>
## 11) Garantindo a Estabilidade Pós-Reboot (Systemd)

Um problema comum em ambientes de produção é o Krill falhar ao publicar ROAs após uma reinicialização do servidor. Isso acontece porque o serviço do Krill pode iniciar *antes* que a rede, o DNS ou a stack TLS estejam completamente operacionais, mesmo que segundos depois tudo se normalize.

O erro típico que você verá nos logs é:
```log
[ERROR] Failed to publish for 'seu_asn_br'.
Error: HTTP client error: Issue accessing URI
```
Isso cria uma "condição de corrida" clássica. A solução correta e elegante é instruir o `systemd` para garantir que a rede esteja totalmente online antes de sequer tentar iniciar o serviço do Krill.

**1. Editando a Unidade de Serviço do Krill**

Vamos usar o comando `systemctl edit` para criar um "override" (uma sobreposição de configuração) para o serviço do Krill. Esta é a forma correta, pois evita modificar o arquivo de serviço original, que poderia ser sobrescrito em uma atualização futura.

```bash
sudo systemctl edit krill
```
Este comando abrirá um editor de texto para um novo arquivo de configuração.

**2. Adicionando as Diretivas de Dependência de Rede**

No editor, insira o seguinte conteúdo. Estas linhas instruem o `systemd` a esperar a rede ficar online (`Wants=`/`After=`) antes de iniciar o serviço.

```ini
[Unit]
Wants=network-online.target
After=network-online.target
```
Salve e feche o arquivo.

**3. Recarregando o Systemd e Reiniciando o Krill**

Após salvar, o `systemd` precisa ser informado sobre a nova configuração. Em seguida, reiniciamos o Krill para garantir que tudo está funcionando.

```bash
sudo systemctl daemon-reload
sudo systemctl restart krill
```

Com essa configuração, o Krill agora iniciará de forma robusta após cada reboot, evitando falsos erros de publicação e garantindo a consistência da sua validação RPKI.

---

<a id="12"></a>
## 12) Opcional: Publicação Própria (RRDP/rsync)

Esta seção é para casos específicos onde a publicação remota não está disponível ou sua arquitetura exige que você hospede seu próprio repositório de RPKI.

**Conceitos importantes:**
-   `/rfc8181`: É o endpoint do **Publication Server** do Krill, onde outras CAs e publicadores se conectam para trocar informações.
-   RRDP (RRDPH): Um protocolo HTTP para distribuição de dados RPKI (arquivos `notification.xml`, snapshots, deltas).
-   rsync: Um protocolo alternativo para distribuição de dados RPKI, mais antigo, mas ainda utilizado.

**Boas Práticas:**
-   Considere usar **hostnames separados** para a interface administrativa e para a publicação de dados.
    -   Ex: Interface admin: `rpki-admin.seudominio.com`
    -   Ex: Publicação (RRDP/rsync): `rpki-pub.seudominio.com`
-   **Não** adicione autenticação HTTP extra no caminho `/rfc8181` ou nos endpoints RRDP, pois o protocolo RPKI já usa mensagens criptograficamente assinadas.

**Checklist (Alto Nível):**
1)  Defina os URIs públicos para RRDP e, opcionalmente, rsync.
    -   Ex: RRDP: `https://rpki-pub.seudominio.com/rrdp/`
    -   Ex: rsync: `rsync://rpki-pub.seudominio.com/repo/`
2)  Inicialize o Publication Server do Krill com esses URIs.
    -   `krillc pubserver server init --rrdp https://rpki-pub.seudominio.com/rrdp/ --rsync rsync://rpki-pub.seudominio.com/repo/`
3)  Sirva os arquivos RRDP via web server (Nginx):
    -   Publique o diretório `$DATA_DIR/repo/rrdp` (ex: `/var/lib/krill/data/repo/rrdp`) no Nginx.
    -   Configure o Nginx para evitar cache longo para o arquivo `notification.xml`, pois ele muda com frequência.
4)  Se você for publicar via rsync, libere a porta `873/tcp` no seu firewall `nftables`.

---

<a id="13"></a>
## 13) Monitoramento e Troubleshooting

### 13.1 Logs do Serviço Krill
Os logs são a primeira fonte de informação para diagnosticar problemas.

```bash
sudo journalctl -u krill -f # Acompanha os logs do serviço Krill em tempo real
```

### 13.2 Métricas (Prometheus)

O Krill expõe métricas no formato Prometheus no endpoint `/metrics`.

```bash
# Se o Krill estiver no HTTPS padrão (autoassinado):
curl -k https://127.0.0.1:3000/metrics | head # Ignora o certificado autoassinado para obter as métricas

# Se o Krill estiver em HTTP (https_mode = "disable"):
# curl http://127.0.0.1:3000/metrics | head
```

> **Atenção:** O endpoint `/metrics` do Krill **não** possui autenticação. Ele **NÃO** deve ser exposto publicamente na internet. Se você estiver usando Nginx como proxy, certifique-se de que o Nginx está configurado para **bloquear** o acesso externo a `/metrics`, como mostrado na seção [5.2.5 Configurar o Proxy HTTPS para o Krill](#5.2.5).

### 13.3 Problemas Comuns e Soluções

-   **Nginx 502 / Interface web não abre:**
    -   Verifique o status do serviço Krill: `sudo systemctl status krill --no-pager`
    -   Analise os logs do Krill: `sudo journalctl -u krill -n 200 --no-pager`
    -   Verifique os logs de erro do Nginx: `sudo tail -n 200 /var/log/nginx/error.log`
    -   Confirme se a diretiva `proxy_pass` no Nginx corresponde ao `https_mode` configurado no Krill:
        -   Krill HTTPS padrão (autoassinado): `proxy_pass https://127.0.0.1:3000/` e `proxy_ssl_verify off;`
        -   `https_mode="disable"` (HTTP): `proxy_pass http://127.0.0.1:3000/`

-   **Erros na configuração de Parent/Repository:**
    -   Verifique o `service_uri` em `/etc/krill.conf`. Ele deve ser um URL público HTTPS válido e **terminar com uma barra (`/`)**.
    -   Confirme se o DNS do seu domínio está correto e se o certificado TLS do Nginx é válido.

---

<a id="14"></a>
## 14) Backup e Recuperação

A perda dos dados do Krill pode resultar na perda da sua CA RPKI e, consequentemente, na invalidação dos seus ROAs. **Um backup regular e seguro é indispensável.**

> **ATENÇÃO:** Nunca migre um Krill para outro servidor sem levar **TODO** o diretório `data_dir` (`/var/lib/krill/data/`) e o arquivo `/etc/krill.conf`. A perda ou inconsistência desses dados pode causar a perda irrecuperável da sua CA.

### 14.1 O que Deve Entrar no Backup
-   **Configuração:** O arquivo `/etc/krill.conf`.
-   **Dados/Estado (Crítico):** O diretório `data_dir` (geralmente `/var/lib/krill/data/`). Este diretório contém as chaves criptográficas da sua CA.

> Faça backup **criptografado** e armazene-o **fora do servidor**. A segurança do backup é tão importante quanto a do sistema em produção.

### 14.2 Script Simples de Backup com Retenção

Crie um script para automatizar o backup diário com retenção de 30 dias.

Arquivo: `/usr/local/sbin/backup-krill.sh`

```bash
sudo nano /usr/local/sbin/backup-krill.sh # Cria o script de backup
```

Conteúdo do script:
```bash
#!/usr/bin/env bash
set -euo pipefail # Sai imediatamente se um comando falhar, evita variáveis não definidas
umask 077         # Garante que arquivos criados tenham permissões restritivas

BACKUP_DIR="/var/backups/krill"       # Diretório onde os backups serão salvos
DATE="$(date +%Y%m%d-%H%M)"           # Formato da data para o nome do arquivo de backup
RETENTION_DAYS="30"                   # Número de dias para manter os backups

mkdir -p "$BACKUP_DIR"                # Cria o diretório de backup se não existir
chmod 700 "$BACKUP_DIR"               # Define permissões restritivas para o diretório de backup

# Cria um arquivo tar.gz com os arquivos e diretórios essenciais
tar -czf "$BACKUP_DIR/krill-backup-$DATE.tar.gz" \
  /etc/krill.conf \
  /var/lib/krill/data/

# Remove backups antigos baseados na RETENTION_DAYS
find "$BACKUP_DIR" -type f -name "krill-backup-*.tar.gz" -mtime +"$RETENTION_DAYS" -delete
echo "OK: Backup do Krill criado em $BACKUP_DIR/krill-backup-$DATE.tar.gz"
```

Defina as permissões de execução para o script:
```bash
sudo chmod 700 /usr/local/sbin/backup-krill.sh # Torna o script executável
```

Agende a execução diária com cron (exemplo: 02:00 da manhã):
```bash
sudo crontab -e # Abre o editor do crontab
```
Adicione a seguinte linha ao final do arquivo crontab e salve:
```cron
0 2 * * * /usr/local/sbin/backup-krill.sh >/dev/null 2>&1 # Executa o script diariamente às 02:00
```

### 14.3 Restauração do backup

Em caso de desastre, siga estes passos para restaurar:

1)  Pare o serviço Krill:
    ```bash
    sudo systemctl stop krill
    ```
2)  Restaure o backup mais recente (ajuste o caminho e nome do arquivo `.tar.gz`):
    ```bash
    sudo tar -xzf /caminho/para/seu/backup/krill-backup-YYYYMMDD-HHMM.tar.gz -C /
    ```
3)  Ajuste as permissões do diretório `data_dir` (o proprietário deve ser `krill:krill`):
    ```bash
    sudo chown -R krill:krill /var/lib/krill/
    ```
4)  Reforce as permissões do arquivo de configuração (se necessário):
    ```bash
    sudo chown root:krill /etc/krill.conf
    sudo chmod 640 /etc/krill.conf
    ```
5)  Inicie o serviço e valide o funcionamento:
    ```bash
    sudo systemctl start krill
    sudo systemctl status krill --no-pager
    ```

---

<a id="15"></a>
## 15) Opcional: Multi‑user (Usuários Nomeados)

Para ambientes maiores, onde múltiplos administradores precisam gerenciar o Krill, é preferível usar autenticação com **usuários nomeados** em vez de compartilhar um único `admin_token`.

> **Nota:** Em ambientes pequenos, um único `admin_token` é aceitável, desde que o acesso à interface do Krill seja estritamente protegido por mecanismos como VPN ou SSH tunnel.

### 15.1 Gerar Hash e Salt para um Novo Usuário

O Krill CLI pode gerar o hash e salt de senha para um novo usuário.
```bash
krillc config user --id admin@example.com # Substitua pelo e-mail/ID do usuário
```
O comando irá solicitar uma senha e gerar as informações necessárias.

### 15.2 Configurar o Usuário em `/etc/krill.conf`

Edite o arquivo de configuração para adicionar as seções de autenticação e o novo usuário.

```bash
sudo nano /etc/krill.conf # Edita o arquivo de configuração do Krill
```

Adicione ou modifique as seguintes linhas:
```toml
auth_type = "config-file" # Habilita autenticação via arquivo de configuração

[auth_users]
# Substitua o e-mail e as credenciais geradas
"admin@example.com" = { role="admin", password_hash="SEU_HASH_AQUI", salt="SEU_SALT_AQUI" }
```

### 15.3 Reiniciar o Serviço

Após as alterações, reinicie o Krill.
```bash
sudo systemctl restart krill # Reinicia o serviço Krill
```

> **Atenção:** Nunca sirva a interface web do Krill em HTTP sem criptografia, especialmente com múltiplos usuários. Use sempre TLS (Nginx) ou SSH tunnel para proteger as credenciais.

---

<a id="16"></a>
## 16) Checklist Final

Antes de considerar sua instalação completa, revise estes pontos:

-   `systemctl status krill`: O serviço Krill está `active (running)`?
-   `ip = "127.0.0.1"` e `port = 3000`: O Krill está escutando apenas localmente, na porta correta?
-   Firewall `nftables` ativo (`sudo nft list ruleset`): As regras de firewall estão corretamente aplicadas e bloqueando a porta `3000/tcp` da internet?
-   Acesso à Interface Web: Você consegue acessar a interface web do Krill via SSH tunnel **ou** por HTTPS (Nginx + TLS) de forma segura?
-   CA, Parent e Repository: Sua CA está criada, e a comunicação com o Parent e o Repositório de Publicação está funcionando?
-   ROAs: Seus ROAs estão criados no Krill e foram validados com sucesso por ferramentas externas (ex: RIPE Validator)?
-   ASPA: Seus objetos ASPA estão criados no Krill e foram validados com sucesso por ferramentas externas?
-   Backup: O script de backup está configurado e funcionando, salvando o `krill.conf` e o `data_dir` regularmente em local seguro?

---

<a id="17"></a>
## 17) O que NÃO Fazer (Erros Comuns a Evitar)

Este é um resumo rápido de práticas que **DEVEM SER EVITADAS** para garantir a segurança e funcionalidade da sua infraestrutura RPKI com Krill.

-   **Não Expor a Porta 3000/tcp Diretamente:** A porta padrão do Krill (`3000/tcp`) é para acesso local. Nunca abra esta porta diretamente no firewall para a internet.
-   **Não Usar HTTP Público para a UI:** Jamais exponha a interface web do Krill via HTTP (porta 80) publicamente. Sempre use HTTPS (porta 443) e, idealmente, um proxy reverso.
-   **Não Esquecer o Backup:** A perda do diretório `data_dir` ou do `krill.conf` significa a perda da sua CA RPKI. Faça backups criptografados e fora do servidor.
-   **Não Definir `maxLength` Incorretamente:** Um `maxLength` muito amplo pode diluir a proteção do seu ROA. Se você anuncia um `/24`, use `maxLength = 24`.
-   **Não Expor o Endpoint `/metrics`:** O endpoint `/metrics` não tem autenticação. Bloqueie o acesso público a ele via Nginx ou firewall.
-   **Não Migrar o Servidor sem o `data_dir` e `krill.conf`:** A migração de um Krill requer que todo o estado (configuração e dados) seja movido. Não negligencie isso.
-   **Não Ignorar a Sincronização de Tempo:** A validade dos certificados RPKI depende de um relógio de sistema preciso. Garanta que o Chrony (ou outro serviço NTP) esteja funcionando.
-   **Não Incluir ASNs Incorretos no ASPA:** Adicionar ASNs que não são seus provedores de trânsito (ex: IXPs, clientes) na configuração ASPA pode causar validações negativas.

---

<a id="18"></a>
## 18) Referências (fontes para consulta)

### Krill (NLnet Labs)

- Documentação (stable): https://krill.docs.nlnetlabs.nl/en/stable/
- Instalar e executar: https://krill.docs.nlnetlabs.nl/en/stable/install-and-run.html
- Interações com Parent/RIR/NIR: https://krill.docs.nlnetlabs.nl/en/stable/parent-interactions.html
- Múltiplos usuários: https://krill.docs.nlnetlabs.nl/en/stable/multi-user.html

### RFCs (RPKI)

- ROA (RFC 6482): https://www.rfc-editor.org/rfc/rfc6482.html
- Provisioning Protocol (RFC 6492): https://www.rfc-editor.org/rfc/rfc6492.html
- Publication Protocol (RFC 8181): https://www.rfc-editor.org/rfc/rfc8181.html
- ASPA (RFC 9319): https://www.rfc-editor.org/rfc/rfc9319.html

---

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
