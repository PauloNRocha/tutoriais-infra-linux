# Tutoriais de Infraestrutura, Redes e Segurança
### Linux • ISP • Produção

## Sobre este repositório

Este repositório reúne tutoriais técnicos que utilizo e mantenho em **ambientes reais de produção**, principalmente em **Provedores de Internet (ISP)**.

Não é um repositório de laboratório, testes acadêmicos ou exemplos “de blog”.
Os materiais aqui refletem **decisões práticas**, limitações reais e preocupações comuns de quem mantém infraestrutura funcionando todos os dias.

O foco não é apenas *como fazer*, mas **por que fazer dessa forma**, incluindo riscos, trade-offs e cuidados necessários em produção.

---

## Escopo e princípios

Este repositório segue alguns princípios claros:

- Conteúdo voltado para **ambientes reais de produção**
- Foco em **infraestrutura de ISP e servidores Linux**
- **Segurança considerada desde o início**, não como correção posterior
- Preferência por **software livre** e soluções abertas
- Exemplos reproduzíveis, organizados e testados
- Explicações técnicas além da simples lista de comandos

---

## Transparência sobre o processo de escrita

Parte do texto utiliza **IA como ferramenta de apoio à escrita, organização e revisão**.
Todo o conteúdo técnico é **validado manualmente**, testado em ambiente real e ajustado conforme experiência prática.

Nada aqui é publicado “no automático”.

---

## Índice de tutoriais

### Acesso remoto

| Título | Descrição |
|------|-----------|
| [Guia de Acesso Remoto Gráfico (XRDP, AnyDesk, TeamViewer)](acesso-remoto/guia_acesso_remoto_grafico_linux.md) | Configuração de acesso remoto gráfico em servidores Linux, incluindo resolução de problemas comuns e boas práticas. |

---

### DNS

| Título | Descrição |
|------|-----------|
| [Bloqueio de Sites com BIND9 e RPZ](dns/guia_bind9_rpz_bloqueio_dns.md) | Bloqueio de domínios via DNS utilizando RPZ no BIND9. |
| [BIND9 no Debian 13 (Master/Slave, TSIG, DNSSEC, Fail2Ban)](dns/guia_producao_bind9_debian13.md) | Implantação de DNS autoritativo em produção com foco em segurança e disponibilidade. |
| [Unbound no Debian 13 (DNS Recursivo para ISP)](dns/guia_producao_unbound_debian13.md) | Servidor DNS recursivo para provedores, com DNSSEC, RPZ e hardening. |

---

### GPU (NVIDIA / CUDA)

| Título | Descrição |
|------|-----------|
| [Instalação NVIDIA + CUDA + PRIME](gpu/guia_nvidia_cuda_prime_ubuntu2404.md) | Drivers NVIDIA, CUDA Toolkit e gráficos híbridos no Linux. |
| [Troubleshooting NVIDIA + CUDA](gpu/guia_nvidia_cuda_troubleshooting_ubuntu2404.md) | Erros comuns e soluções práticas envolvendo NVIDIA e CUDA. |
| [Recuperar e Manter Driver NVIDIA](gpu/guia_recuperar_driver_nvidia_ubuntu2404.md) | Recuperação de drivers, kernels, Secure Boot e módulos NVIDIA. |

---

### Recuperação de sistema

| Título | Descrição |
|------|-----------|
| [Recuperar Kernel e /lib/modules](recuperacao-de-sistema/guia_recuperar_ubuntu_lib_modules.md) | Guia de emergência para restaurar kernel e módulos após falhas graves. |

---

### Redes

| Título | Descrição |
|------|-----------|
| [Configuração de Rede no Debian 12](rede/guia_configurar_rede_debian12.md) | Configuração de interfaces, IP fixo e DHCP no Debian. |
| [Configuração de Wi-Fi no Debian 12](rede/guia_configurar_wifi_debian12.md) | Conexão e troubleshooting de redes Wi-Fi. |

---

### Segurança

| Título | Descrição |
|------|-----------|
| [Krill RPKI no Debian 13](seguranca/guia_producao_krill_rpki_debian13.md) | Autoridade Certificadora RPKI para proteção de roteamento BGP. |
| [Wazuh AIO para Provedores](seguranca/guia_wazuh_aio_provedor_debian13.md) | Monitoramento, segurança e auditoria em ambiente de ISP. |
| [Guia do pwgen](seguranca/guia_pwgen.md) | Geração de senhas seguras via linha de comando. |

---

### Sistema

| Título | Descrição |
|------|-----------|
| [Cron e Crontab](sistema/guia_cron_crontab.md) | Agendamento de tarefas no Linux. |
| [Desabilitar Suspensão e Hibernação](sistema/guia_desabilitar_suspensao_hibernacao_debian.md) | Desativação correta de suspensão em servidores e notebooks. |
| [Formatar Discos no Linux](sistema/guia_formatar_disco_linux.md) | Identificação, particionamento e formatação segura. |
| [Sincronização com rsync](sistema/guia_rsync_sincronizacao_pastas.md) | Backup e espelhamento de diretórios. |
| [Fixar Frequência Máxima da CPU](sistema/guia_fixar_frequencia_cpu_debian12.md) | Ajuste de governor para desempenho máximo. |
| [Compactação no Terminal Linux](sistema/guia_compactacao_linux_terminal.md) | tar, gzip, bzip2, xz e zip. |
| [Usuários, Grupos e Permissões](sistema/guia_usuarios_grupos_permissoes.md) | Usuários, sudo, permissões e ACL. |
| [Servidor NTP Interno com Chrony (Debian 13)](sistema/guia_producao_ntp_chrony_debian13.md) | NTP interno com IPv4/IPv6, nftables, troubleshooting e NTS opcional. |

---

### VPN

| Título | Descrição |
|------|-----------|
| [OpenVPN 2 no Linux](vpn/guia_openvpn2_debian_ubuntu.md) | Cliente OpenVPN no Debian/Ubuntu, com NetworkManager. |

---

## Sobre

Infraestrutura e redes com foco em:

- Servidores Linux (Debian/Ubuntu)
- Ambientes de Provedor de Internet (ISP)
- Segurança e operação em produção

Este repositório funciona como **caderno técnico público**: problemas reais, soluções testadas.

---

## Aviso

Exemplos são genéricos. **Revise e adapte antes de aplicar em produção.**

GitHub: https://github.com/PauloNRocha
