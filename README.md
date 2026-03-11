# Tutoriais de Infraestrutura, Redes e Segurança
### Linux • ISP • Produção

![Linux](https://img.shields.io/badge/Linux-Debian%20%7C%20Ubuntu-2b3137?style=flat-square&logo=linux)
![Foco](https://img.shields.io/badge/Foco-ISP%20%7C%20Produ%C3%A7%C3%A3o-0a7ea4?style=flat-square)
![Idioma](https://img.shields.io/badge/Idioma-PT--BR-22863a?style=flat-square)
![Status](https://img.shields.io/badge/Status-Em%20uso%20real-1f6feb?style=flat-square)

## Sobre este repositório

Criei este repositório para deixar documentado, do jeito mais detalhado que consigo, o que faço no dia a dia com servidores Linux, redes, segurança e infraestrutura, principalmente em cenário de ISP.

Ele funciona como consulta para mim e também para quem quiser aproveitar alguma configuração, sequência de comandos ou troubleshooting que eu já precisei resolver antes.

Muita coisa aqui existe por um motivo simples: eu também esqueço. Então prefiro registrar direito, com contexto, ordem de execução e observações práticas, para não precisar toda vez voltar a pesquisar tudo do zero quando precisar repetir algo.

## Como esses guias são escritos

Os guias daqui tentam ser detalhados porque eu realmente uso esse material como referência de trabalho, não como texto de vitrine.

Uso IA para revisão de escrita, organização e acabamento dos guias.

Pode acontecer de algo não funcionar para você exatamente da mesma forma que funcionou no meu cenário, caso opte por replicar em outra distro. Então use os guias como base prática, mas revise e adapte conforme necessário antes de aplicar.

## O que você vai encontrar aqui

A maior parte dos guias aqui é voltada para ambiente de produção, com foco em Linux, redes e ISP. Sempre que dá, prefiro usar software livre e de código aberto.
Também tento deixar os passos bem detalhados, porque muita coisa depois vira consulta para mim mesmo.

## Índice de tutoriais

### Acesso remoto

| Título | Descrição |
|------|-----------|
| [Guia de Acesso Remoto Gráfico (XRDP, AnyDesk, TeamViewer)](acesso-remoto/guia_acesso_remoto_grafico_linux.md) | Configuração de acesso remoto gráfico em servidores Linux, incluindo resolução de problemas comuns e boas práticas. |
| [Acesso SSH por chave pública (porta customizada)](acesso-remoto/guia_producao_ssh_chave_publica_linux.md) | Padronização de acesso SSH em produção com login somente por chave, porta customizada e aliases para múltiplos servidores. |

---

### DNS

| Título | Descrição |
|------|-----------|
| [Bloqueio de Sites com BIND9 e RPZ](dns/guia_bind9_rpz_bloqueio_dns.md) | Bloqueio de domínios via DNS utilizando RPZ no BIND9. |
| [BIND9 no Debian 13 (Master/Slave, TSIG, DNSSEC, Fail2Ban)](dns/guia_producao_bind9_debian13.md) | Implantação de DNS autoritativo em produção com foco em segurança e disponibilidade. |
| [Unbound no Debian 13 (DNS Recursivo para ISP)](dns/guia_producao_unbound_debian13.md) | Servidor DNS recursivo para provedores, com DNSSEC, RPZ e hardening. |

---

### GPU (NVIDIA / CUDA)

| Título                                                                              | Descrição                                                      |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| [Instalação NVIDIA + CUDA + PRIME](gpu/guia_nvidia_cuda_prime_ubuntu2404.md)        | Drivers NVIDIA, CUDA Toolkit e gráficos híbridos no Linux.     |
| [Troubleshooting NVIDIA + CUDA](gpu/guia_nvidia_cuda_troubleshooting_ubuntu2404.md) | Erros comuns e soluções práticas envolvendo NVIDIA e CUDA.     |
| [Recuperar e Manter Driver NVIDIA](gpu/guia_recuperar_driver_nvidia_ubuntu2404.md)  | Recuperação de drivers, kernels, Secure Boot e módulos NVIDIA. |

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
| [Flameshot no Ubuntu Wayland iniciando com o sistema](sistema/guia_producao_flameshot_wayland_autostart_ubuntu.md) | Autostart do Flameshot no GNOME com Wayland e uso da tecla Print. |
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

## Observação final

Esse repositório é público porque, além de me ajudar a manter tudo centralizado e documentado, pode servir para outras pessoas que lidam com problemas parecidos.

Se algum guia te ajudar, ótimo. 

GitHub: https://github.com/PauloNRocha
