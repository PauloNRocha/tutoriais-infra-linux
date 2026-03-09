# Guia de Produção — Flameshot no Ubuntu Wayland iniciando com o sistema

*Criado em: 09 de março de 2026*

Esse guia registra a forma que funcionou no meu notebook para fazer o Flameshot iniciar junto com a sessão no Ubuntu com Wayland e também assumir a tecla `Print`. Ele cobre o autostart no usuário, o ajuste do atalho no GNOME e a validação básica do funcionamento. Não é uma receita universal para qualquer ambiente fora de Ubuntu + GNOME + Wayland.

---

## Cenário

Aqui o Flameshot não funcionava direito quando eu tentava usar de forma “normal” pelo atalho ou pela inicialização automática comum.

O comportamento era basicamente este:

- se eu abrisse pelo terminal, funcionava;
- se eu tentasse usar por atalho direto ou só colocar para iniciar com o sistema, não funcionava como esperado.

Depois de alguns testes, o que resolveu foi:

- criar um script de inicialização no usuário;
- subir o Flameshot com ajuste para Wayland;
- configurar o autostart no GNOME;
- liberar a tecla `Print` do atalho padrão do Ubuntu;
- criar um atalho personalizado chamando o Flameshot da forma que realmente funcionou aqui.

---

## Ambiente

- Ubuntu
- Wayland
- GNOME
- Flameshot

---

## Pré-requisito

O guia parte do pressuposto de que o Flameshot já está instalado.

Se ainda não estiver:

```bash
sudo apt install -y flameshot
```

Validação rápida:

```bash
command -v flameshot
flameshot --version
```

---

## Arquivos usados

### Script de inicialização
`~/.local/bin/flameshot-wayland-autostart.sh`

### Arquivo de autostart
`~/.config/autostart/flameshot-wayland.desktop`

---

## Implementação

### 1. Criar o script de inicialização

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/flameshot-wayland-autostart.sh <<'EOF'
#!/usr/bin/env bash
sleep 4
export QT_QPA_PLATFORM=wayland
/usr/bin/flameshot &
EOF

chmod +x ~/.local/bin/flameshot-wayland-autostart.sh
```

### O que esse script faz

- `sleep 4`
  - espera alguns segundos para a sessão gráfica terminar de subir;
- `export QT_QPA_PLATFORM=wayland`
  - força o uso do Wayland;
- `/usr/bin/flameshot &`
  - inicia o Flameshot em segundo plano.

No meu caso esse pequeno atraso ajudou, porque se ele subir cedo demais pode não pegar o ambiente gráfico ainda pronto.

---

### 2. Criar o autostart do usuário

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/flameshot-wayland.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Flameshot Wayland
Exec=/home/$USER/.local/bin/flameshot-wayland-autostart.sh
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Terminal=false
EOF
```

Depois ajustei o arquivo porque o `$USER` dentro do `.desktop` pode não expandir direito:

```bash
sed -i "s|\$USER|$USER|g" ~/.config/autostart/flameshot-wayland.desktop
```

Esse cuidado faz sentido porque, no GNOME, expansão de variável em atalho/comando nem sempre acontece da forma que a gente espera.

---

### 3. Testar manualmente antes de reiniciar a sessão

```bash
~/.local/bin/flameshot-wayland-autostart.sh
```

Aqui apareceu a seguinte saída:

```text
flameshot: info: Captura salva na área de transferência.
```

No meu caso isso já mostrou que o comando estava funcionando no ambiente atual.

---

## Configuração da tecla Print

### 4. Liberar o atalho padrão do Ubuntu

No GNOME, a tecla `Print` normalmente já vem presa no atalho padrão de captura de tela interativa.

Caminho:

```text
Configurações → Teclado → Atalhos de teclado → Capturas de tela
```

A opção que precisei mexer foi:

```text
Fazer uma captura de tela interativa
```

A ideia é remover dali a tecla `Print`, para ela ficar livre para o Flameshot.

Se pela interface gráfica não der para limpar direito, este comando resolve:

```bash
gsettings set org.gnome.shell.keybindings show-screenshot-ui "[]"
```

Isso limpa o atalho padrão do GNOME para a captura interativa.

---

### 5. Criar um atalho personalizado para o Flameshot

Caminho:

```text
Configurações → Teclado → Atalhos de teclado → Atalhos personalizados
```

Criar um novo atalho com:

**Nome**
```text
Flameshot
```

**Comando**
```bash
bash -lc 'QT_QPA_PLATFORM=wayland flameshot gui'
```

Depois associar a tecla:

```text
Print
```

Aqui o detalhe importante foi justamente esse comando. Chamando dessa forma funcionou. Chamando de forma mais simples, não.

---

## Resultado esperado

Depois disso, o comportamento esperado é este:

- o Flameshot sobe sozinho quando eu entro na sessão;
- a tecla `Print` abre o Flameshot;
- eu não fico mais dependente de abrir pelo terminal toda vez.

---

## Validação

### Validar o autostart
Encerrar a sessão e entrar novamente.

Verificar se o Flameshot subiu sozinho.

### Validar o atalho
Pressionar:

```text
Print
```

Resultado esperado:

- abrir a interface de captura do Flameshot.

### Validar a captura
Fazer uma captura de teste.

Resultado esperado:

- captura funcionando normalmente;
- imagem salva ou copiada para a área de transferência, conforme a configuração usada no Flameshot.

---

## Troubleshooting

### 1. Funciona no terminal, mas não funciona no atalho
Usar exatamente:

```bash
bash -lc 'QT_QPA_PLATFORM=wayland flameshot gui'
```

Evitar usar só:

```bash
flameshot gui
```

No meu caso, e também em ambiente Wayland no geral, isso pode falhar.

---

### 2. Não sobe no login
Verificar se os arquivos existem:

```bash
ls -l ~/.config/autostart/flameshot-wayland.desktop
ls -l ~/.local/bin/flameshot-wayland-autostart.sh
```

Verificar se o script está executável:

```bash
chmod +x ~/.local/bin/flameshot-wayland-autostart.sh
```

Testar manualmente:

```bash
~/.local/bin/flameshot-wayland-autostart.sh
```

Se ainda assim não subir, uma opção é aumentar o atraso. Por exemplo, trocar:

```bash
sleep 4
```

por:

```bash
sleep 8
```

---

### 3. A tecla Print continua abrindo a ferramenta nativa do Ubuntu
Limpar novamente o atalho padrão do GNOME:

```bash
gsettings set org.gnome.shell.keybindings show-screenshot-ui "[]"
```

---

## Rollback

Se eu quiser desfazer tudo:

### Remover autostart
```bash
rm -f ~/.config/autostart/flameshot-wayland.desktop
```

### Remover script
```bash
rm -f ~/.local/bin/flameshot-wayland-autostart.sh
```

### Restaurar o atalho padrão do GNOME
Reconfigurar manualmente em:

```text
Configurações → Teclado → Atalhos de teclado → Capturas de tela
```

ou redefinir os atalhos pelo próprio GNOME.

---

## Conclusão

No meu caso, no Ubuntu com Wayland, a forma que funcionou de verdade foi esta:

- subir o Flameshot por script do usuário;
- exportar `QT_QPA_PLATFORM=wayland`;
- usar autostart do GNOME;
- tirar o `Print` do atalho padrão do sistema;
- criar um atalho personalizado chamando o Flameshot via shell.

Foi essa abordagem que resolveu o problema real aqui, onde o Flameshot só funcionava quando eu abria pelo terminal.

---

## Créditos

Autor: Paulo Rocha  
Repositório: https://github.com/PauloNRocha
