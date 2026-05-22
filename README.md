# Tutoriais de Infraestrutura, Redes e Segurança
### Linux • Redes • Segurança • ISP • Produção

![Linux](https://img.shields.io/badge/Linux-Debian%20%7C%20Ubuntu-2b3137?style=flat-square&logo=linux)
![Foco](https://img.shields.io/badge/Foco-ISP%20%7C%20Produ%C3%A7%C3%A3o-0a7ea4?style=flat-square)
![Idioma](https://img.shields.io/badge/Idioma-PT--BR-22863a?style=flat-square)
![Status](https://img.shields.io/badge/Status-Em%20uso%20real-1f6feb?style=flat-square)

## Sobre este repositório

Criei este repositório para deixar documentado, do jeito mais detalhado que consigo, o que faço no dia a dia com servidores Linux, redes, segurança e infraestrutura, principalmente em cenário de ISP.

Ele funciona como consulta para mim e também para quem quiser aproveitar alguma configuração, sequência de comandos ou troubleshooting que eu já precisei resolver antes.

Muita coisa aqui existe por um motivo simples: eu também esqueço. Então prefiro registrar direito, com contexto, ordem de execução e observações práticas, para não precisar toda vez voltar a pesquisar tudo do zero quando precisar repetir algo.

A ideia não é transformar cada arquivo em uma documentação oficial do projeto citado. O foco é registrar procedimentos práticos, com o máximo de contexto possível, para ajudar em implantação, manutenção, recuperação e diagnóstico.

## Como esses guias são escritos

Os guias daqui tentam ser detalhados porque eu realmente uso esse material como referência de trabalho, não como texto de vitrine.

Uso IA como apoio para revisar escrita, organizar ideias e melhorar acabamento, mas a base dos guias vem de uso real, teste, erro, manutenção e rotina de infraestrutura.

Pode acontecer de algo não funcionar exatamente da mesma forma em outro ambiente, outra versão de sistema ou outra distribuição. Use os guias como base prática, mas revise e adapte conforme necessário antes de aplicar.

## Para quem este repositório pode servir

Este repositório pode ser útil para administradores de sistemas, técnicos de provedores, analistas de redes, estudantes de infraestrutura Linux e profissionais que precisam lidar com servidores em produção.

Os guias assumem alguma familiaridade com terminal Linux, edição de arquivos de configuração e conceitos básicos de rede.

## Antes de aplicar em produção

Boa parte do conteúdo mexe com serviços sensíveis: SSH, DNS, VPN, firewall, RPKI, monitoramento, disco, usuários e automações.

Antes de executar qualquer procedimento em produção:

- leia o guia inteiro antes de começar;
- ajuste nomes, IPs, domínios, portas e caminhos para o seu ambiente;
- faça backup, snapshot ou tenha um plano de rollback quando o impacto justificar;
- valide primeiro em laboratório ou VM quando houver risco de indisponibilidade;
- revise comandos destrutivos com calma, principalmente em disco, usuário, firewall e serviços remotos.

Os procedimentos são compartilhados como referência prática. O uso em produção precisa ser revisado por quem administra o ambiente, porque cada rede tem suas particularidades.

## O que você vai encontrar aqui

A maior parte dos guias é voltada para ambiente de produção, com foco em Linux, redes e ISP. Sempre que dá, prefiro usar software livre e de código aberto.

Você vai encontrar procedimentos de instalação, configuração, hardening, recuperação e troubleshooting. Os comandos aparecem prontos para copiar, mas tento sempre explicar o motivo de cada etapa, porque comando sem contexto vira armadilha fácil.

## Guias disponíveis

### Acesso remoto

| Título | Descrição |
|------|-----------|
| [Guia de Acesso Remoto Gráfico (XRDP, AnyDesk, TeamViewer)](acesso-remoto/guia_acesso_remoto_grafico_linux.md) | Configuração de acesso remoto gráfico em servidores Linux, incluindo resolução de problemas comuns e boas práticas. |
| [Acesso SSH por chave pública (porta customizada)](acesso-remoto/guia_producao_ssh_chave_publica_linux.md) | Padronização de acesso SSH em produção com login somente por chave, porta customizada e aliases para múltiplos servidores. |

---

### DNS

| Título | Descrição |
|------|-----------|
| [Bloqueio de Domínios com BIND9 e RPZ](dns/guia_bind9_rpz_bloqueio_dns.md) | Bloqueio de domínios via DNS utilizando RPZ no BIND9. |
| [BIND9 no Debian 13 (Primary/Secondary, TSIG, DNSSEC, Fail2Ban)](dns/guia_producao_bind9_debian13.md) | Implantação de DNS autoritativo em produção com foco em segurança e disponibilidade. |
| [Recuperação de zona BIND9 com DNSSEC após erro de sintaxe](dns/guia_producao_bind9_recuperacao_zona_dnssec.md) | Procedimento de recuperação após SERVFAIL em zona assinada, com impacto em DKIM/DMARC. |
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
| [Configuração de Rede no Debian 12/13](rede/guia_configurar_rede_debian12_13.md) | Configuração de interfaces, IP fixo e DHCP no Debian. |
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

Se algum guia te ajudar, melhor ainda. Se não encaixar exatamente no seu ambiente, use como base e adapte para a sua realidade.

## Licença

O conteúdo deste repositório está licenciado sob a Creative Commons Attribution 4.0 International (CC BY 4.0).

Na prática: você pode usar, copiar, adaptar e compartilhar os guias, inclusive em ambiente profissional, desde que mantenha a atribuição ao autor.

GitHub: https://github.com/PauloNRocha/tutoriais-infra-linux
