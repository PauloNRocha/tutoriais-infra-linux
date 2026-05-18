# Guia Prático: Flameshot no Ubuntu com GNOME Wayland

*Criado em: 09 de março de 2026*  
*Última atualização em: 18 de maio de 2026*

Esse guia registra a forma que funcionou no meu notebook para fazer o Flameshot iniciar junto com a sessão no Ubuntu usando GNOME Wayland e também assumir a tecla `Print`. O foco aqui é resolver esse cenário específico: Flameshot instalado pelo APT, sessão Wayland e atalho de captura do GNOME atrapalhando o uso da tecla `Print`.

Não trato aqui de KDE, Xfce, X11, Flatpak, Snap ou outras combinações. Podem funcionar, mas não foi esse o ambiente que motivou o guia.

---

## Índice rápido

1. [Cenário](#1)
2. [Confirmar o ambiente](#2)
3. [Instalar e testar o Flameshot](#3)
4. [Criar o script de inicialização](#4)
5. [Criar o autostart do usuário](#5)
6. [Configurar a tecla Print](#6)
7. [Validação](#7)
8. [Troubleshooting](#8)
9. [Rollback](#9)
10. [Referências](#referencias)

---

<a id="1"></a>
## 1. Cenário

Aqui o Flameshot não funcionava direito quando eu tentava usar de forma “normal” pelo atalho ou pela inicialização automática comum.

O comportamento era basicamente este:

- abrindo pelo terminal, funcionava;
- usando atalho direto, falhava ou não abria como esperado;
- colocando apenas para iniciar com a sessão, o comportamento ficava inconsistente.

Depois de alguns testes, o que resolveu foi:

- criar um script de inicialização no usuário;
- subir o Flameshot com ajuste para Wayland;
- configurar o autostart no GNOME;
- liberar a tecla `Print` do atalho padrão do Ubuntu;
- criar um atalho personalizado chamando o Flameshot da forma que realmente funcionou aqui.

---

<a id="2"></a>
## 2. Confirmar o ambiente

Antes de aplicar, confirme se você está mesmo em GNOME com Wayland:

```bash
echo "$XDG_SESSION_TYPE"
echo "$XDG_CURRENT_DESKTOP"
```

Saída esperada no cenário deste guia:

```text
wayland
ubuntu:GNOME
```

ou algo próximo envolvendo `GNOME`.

Se aparecer `x11`, este guia provavelmente não é o caminho ideal para o seu caso.

---

<a id="3"></a>
## 3. Instalar e testar o Flameshot

Instale pelo APT:

```bash
sudo apt update
sudo apt install -y flameshot
```

Valide:

```bash
command -v flameshot
flameshot --version
```

Teste manualmente:

```bash
QT_QPA_PLATFORM=wayland flameshot gui
```

Se o Flameshot abrir a interface de captura, o básico está funcionando.

---

<a id="4"></a>
## 4. Criar o script de inicialização

Crie o diretório:

```bash
mkdir -p "$HOME/.local/bin"
```

Crie o script:

```bash
nano "$HOME/.local/bin/flameshot-wayland-autostart.sh"
```

Cole o conteúdo abaixo:

```bash
#!/usr/bin/env bash

sleep 4
export QT_QPA_PLATFORM=wayland
/usr/bin/flameshot &
```

Dê permissão de execução:

```bash
chmod +x "$HOME/.local/bin/flameshot-wayland-autostart.sh"
```

O que esse script faz:

- `sleep 4`: espera alguns segundos para a sessão gráfica terminar de subir;
- `QT_QPA_PLATFORM=wayland`: força o caminho Wayland do Qt;
- `/usr/bin/flameshot &`: inicia o Flameshot em segundo plano.

No meu caso esse pequeno atraso ajudou, porque subir cedo demais fazia o Flameshot não pegar o ambiente gráfico ainda pronto.

Teste manualmente:

```bash
"$HOME/.local/bin/flameshot-wayland-autostart.sh"
```

---

<a id="5"></a>
## 5. Criar o autostart do usuário

Crie o diretório:

```bash
mkdir -p "$HOME/.config/autostart"
```

Crie o arquivo `.desktop`:

```bash
cat > "$HOME/.config/autostart/flameshot-wayland.desktop" <<EOF
[Desktop Entry]
Type=Application
Name=Flameshot Wayland
Exec=$HOME/.local/bin/flameshot-wayland-autostart.sh
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Terminal=false
EOF
```

Valide o caminho configurado em `Exec=`:

```bash
grep '^Exec=' "$HOME/.config/autostart/flameshot-wayland.desktop"
ls -l "$HOME/.local/bin/flameshot-wayland-autostart.sh"
```

O caminho gravado no `Exec=` precisa ser absoluto, como `/home/paulo/.local/bin/flameshot-wayland-autostart.sh`. Em arquivo `.desktop`, eu evito deixar `$USER`, `$HOME` ou qualquer expansão pendente, porque isso nem sempre se comporta como no terminal.

---

<a id="6"></a>
## 6. Configurar a tecla Print

No GNOME, a tecla `Print` normalmente já vem presa no atalho padrão de captura de tela interativa.

Pelo caminho gráfico:

```text
Configurações -> Teclado -> Atalhos de teclado -> Capturas de tela
```

A opção que precisei liberar foi:

```text
Fazer uma captura de tela interativa
```

A ideia é remover dali a tecla `Print`, para ela ficar livre para o Flameshot.

Se pela interface gráfica não der para limpar direito, confira o valor atual:

```bash
gsettings get org.gnome.shell.keybindings show-screenshot-ui
```

Limpe o atalho padrão:

```bash
gsettings set org.gnome.shell.keybindings show-screenshot-ui "[]"
```

Depois crie um atalho personalizado:

```text
Configurações -> Teclado -> Atalhos de teclado -> Atalhos personalizados
```

Use:

```text
Nome: Flameshot
Comando: bash -lc 'QT_QPA_PLATFORM=wayland flameshot gui'
Tecla: Print
```

Aqui o detalhe importante foi justamente chamar via `bash -lc` com `QT_QPA_PLATFORM=wayland`. Chamando só `flameshot gui`, no meu ambiente não ficou confiável.

---

<a id="7"></a>
## 7. Validação

### 7.1 Validar o autostart

Encerre a sessão e entre novamente.

Depois confira se o Flameshot está rodando:

```bash
pgrep -a flameshot
```

### 7.2 Validar o atalho

Pressione:

```text
Print
```

Resultado esperado:

- abrir a interface de captura do Flameshot.

### 7.3 Validar a captura

Faça uma captura de teste.

Resultado esperado:

- captura funcionando normalmente;
- imagem salva ou copiada para a área de transferência, conforme a configuração usada no Flameshot.

---

<a id="8"></a>
## 8. Troubleshooting

### 8.1 Funciona no terminal, mas não funciona no atalho

Use exatamente este comando no atalho personalizado:

```bash
bash -lc 'QT_QPA_PLATFORM=wayland flameshot gui'
```

Evite usar só:

```bash
flameshot gui
```

No meu caso, e também em ambiente Wayland no geral, isso pode falhar.

### 8.2 Não sobe no login

Verifique se os arquivos existem:

```bash
ls -l "$HOME/.config/autostart/flameshot-wayland.desktop"
ls -l "$HOME/.local/bin/flameshot-wayland-autostart.sh"
```

Verifique se o script está executável:

```bash
chmod +x "$HOME/.local/bin/flameshot-wayland-autostart.sh"
```

Teste manualmente:

```bash
"$HOME/.local/bin/flameshot-wayland-autostart.sh"
```

Se ainda assim não subir, aumente o atraso no script. Por exemplo, troque:

```bash
sleep 4
```

por:

```bash
sleep 8
```

### 8.3 A tecla Print continua abrindo a ferramenta nativa do Ubuntu

Confira:

```bash
gsettings get org.gnome.shell.keybindings show-screenshot-ui
```

Limpe novamente:

```bash
gsettings set org.gnome.shell.keybindings show-screenshot-ui "[]"
```

### 8.4 Aparece erro de permissão ou captura não autorizada no Wayland

No Wayland, a captura de tela passa pelo portal de desktop. Se o Flameshot abrir, mas não conseguir capturar, teste o comando recomendado na documentação do Flameshot:

```bash
script --command "QT_QPA_PLATFORM=wayland flameshot gui" /dev/null
```

Se aparecer uma janela pedindo permissão para compartilhar a tela, permita a captura e teste novamente.

---

<a id="9"></a>
## 9. Rollback

### 9.1 Desativar o autostart

Renomeie o arquivo `.desktop`:

```bash
mv "$HOME/.config/autostart/flameshot-wayland.desktop" "$HOME/.config/autostart/flameshot-wayland.desktop.bak"
```

### 9.2 Desativar o script

Renomeie o script:

```bash
mv "$HOME/.local/bin/flameshot-wayland-autostart.sh" "$HOME/.local/bin/flameshot-wayland-autostart.sh.bak"
```

### 9.3 Restaurar o atalho padrão do GNOME

Pelo caminho gráfico:

```text
Configurações -> Teclado -> Atalhos de teclado -> Capturas de tela
```

Ou resetando a chave:

```bash
gsettings reset org.gnome.shell.keybindings show-screenshot-ui
```

Depois remova ou altere o atalho personalizado criado para o Flameshot.

---

<a id="referencias"></a>
## Referências (fontes para consulta)

### Flameshot

- Wayland Help: https://flameshot.org/docs/guide/wayland-help/
- Command Line Options: https://flameshot.org/docs/guide/command-line-options/

### freedesktop.org

- Desktop Entry Specification: https://specifications.freedesktop.org/desktop-entry-spec/latest-single/
- Desktop Application Autostart Specification: https://xdg.pages.freedesktop.org/xdg-specs/autostart-spec/latest/

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha/tutoriais-infra-linux
