# Configuração do Barrier — Guia para Humanos

Compartilha teclado e mouse entre 3 máquinas via rede local.

**Layout:** Linux | Mac | Windows — o Windows é o **server**, Mac e Linux são **clients**. O mouse passa do Windows para a esquerda (Mac) e do Mac para a esquerda (Linux).

**Topologia (IPs fictícios de exemplo):**

| Máquina | Papel | IP fictício | Screen name |
|---------|-------|-------------|-------------|
| Windows | Server | `192.168.1.50` | `desktop` |
| Mac     | Client | `192.168.1.60` | `macbook` |
| Linux   | Client | `192.168.1.70` | `lab` |

> Troque os IPs/screen names pelos reais da sua rede. Todos os exemplos abaixo usam esses fictícios.

---

## Pré-requisitos

- Barrier 2.4+ instalado nas 3 máquinas (repo: https://github.com/debauchee/barrier)
- As 3 máquinas na mesma sub-rede
- Porta **24800/tcp** liberada no firewall de todas (no Windows, liberar de entrada para o server)

---

## 1. Windows — Server

### 1.1 Configuração

Crie o arquivo de config em `C:\Users\<usuario>\AppData\Local\Barrier\Barrier.conf`:

```ini
section: screens
	desktop:
	macbook:
		ctrl = meta
	lab:
end

section: links
	desktop:
		left = macbook
	macbook:
		right = desktop
		left = lab
	lab:
		right = macbook
end
```

**Mapeamento de modificadores por screen:** dentro da seção `screens`, cada screen pode ter opções de modificador na forma `nome = novo` (`shift`, `ctrl`, `alt`, `altgr`, `meta`, `super`). O exemplo acima faz o **Ctrl do Windows virar Command (⌘) no Mac** — útil para atalhos tipo Ctrl+C/V funcionarem como Cmd+C/V. Aplicar apenas no screen do Mac não afeta o Linux.

### 1.2 Iniciar o server

```powershell
Start-Process -FilePath "C:\Program Files\Barrier\barriers.exe" `
  -ArgumentList '--config','C:\Users\<usuario>\AppData\Local\Barrier\Barrier.conf','--disable-crypto','-f'
```

Confira a escuta:

```powershell
netstat -ano | findstr 24800
```

### 1.3 Iniciar automaticamente (sem janela)

O `barriers` é um app de console — se iniciado por um `.cmd`, abre uma janela que, se fechada, **mata o server**. Use um VBScript que roda o server oculto (sem janela):

Crie o arquivo `C:\Users\<usuario>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\barrier-server.vbs`:

```vbs
CreateObject("Wscript.Shell").Run """C:\Program Files\Barrier\barriers.exe"" --config C:\Users\<usuario>\AppData\Local\Barrier\Barrier.conf --disable-crypto -f", 0, False
```

O terceiro argumento `0` oculta a janela; o `False` não espera o processo terminar. O server sobe no login sem nada visível. Teste com `wscript "caminho\barrier-server.vbs"` e confirme com `netstat -ano | findstr 24800`.

> A pasta Startup inicia **no login**. Para iniciar no boot (antes do login), use o Agendador de Tarefas.

---

## 2. Mac — Client

### 2.1 Iniciar o client (teste manual)

```bash
/Applications/Barrier.app/Contents/MacOS/barrierc -f --no-tray --name macbook --disable-crypto 192.168.1.50:24800
```

> Na primeira execução, conceda permissões de **Rede Local** e **Acessibilidade** em Ajustes do Sistema → Privacidade e Segurança. Sem a de Rede Local, o client falha com "No route to host".

### 2.2 Iniciar automaticamente (launchd)

Crie `~/Library/LaunchAgents/com.barrier.client.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.barrier.client</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Applications/Barrier.app/Contents/MacOS/barrierc</string>
        <string>-f</string>
        <string>--no-tray</string>
        <string>--name</string>
        <string>macbook</string>
        <string>--disable-crypto</string>
        <string>192.168.1.50:24800</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Carregar e testar:

```bash
plutil -lint ~/Library/LaunchAgents/com.barrier.client.plist
launchctl load ~/Library/LaunchAgents/com.barrier.client.plist
lsof -i :24800
```

---

## 3. Linux — Client

### 3.1 Instalar

```bash
sudo apt install barrier   # Debian/Ubuntu (outras distros: veja o repo)
```

### 3.2 Iniciar o client (teste manual)

```bash
/usr/bin/barrierc -f --no-tray --name lab --disable-crypto 192.168.1.50:24800
```

### 3.3 Iniciar automaticamente (systemd user)

Crie `~/.config/systemd/user/barrier-client.service`:

```ini
[Unit]
Description=Barrier client
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/barrierc -f --no-tray --name lab --disable-crypto 192.168.1.50:24800
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

Habilitar e iniciar:

```bash
systemctl --user daemon-reload
systemctl --user enable barrier-client.service
loginctl enable-linger schon   # inicia mesmo sem login gráfico
systemctl --user start barrier-client.service
```

Confirmar:

```bash
systemctl --user status barrier-client.service --no-pager
ss -tn | grep 24800   # deve mostrar ESTAB para o IP do Windows
```

---

## 4. Teste final

Em qualquer máquina, confira conexões estabelecidas na porta 24800:

```bash
ss -tn | grep 24800   # Linux
lsof -i :24800        # Mac
netstat -ano | findstr 24800   # Windows
```

Movimentação esperada: mouse no **Windows**, vá à borda esquerda → **Mac**; no **Mac**, borda esquerda → **Linux**.

---

## 5. Layout de teclado entre máquinas

O Barrier transmite a tecla conforme o layout da máquina **de origem** (onde o teclado físico está conectado). Ou seja, o caractere que chega no destino é decidido pelo layout do servidor, não do client. Não existe remapeamento de layout de teclado (letras/caracteres) no Barrier — só de modificadores.

Para o `ç` e acentos funcionarem igual em todas as telas, **todas as máquinas devem usar o mesmo layout**:

- **Windows (ABNT2):** `00010416` — `Get-WinUserLanguageList` deve mostrar `0416:00010416`
- **Linux (br):** `setxkbmap -query` deve mostrar `X11 Layout: br`

> Se o Windows usar US International (`00020409`), num teclado físico ABNT2 a tecla `;` vira `ç` no lugar errado. Alinhe os layouts para o mesmo idioma.

---

## Solução de problemas

| Sintoma | Causa provável | Correção |
|---------|----------------|----------|
| `No route to host` no Mac | Permissão de Rede Local negada | Ajustes → Privacidade → Rede Local → conceder ao Barrier |
| `server already has a connected client with our name` | Cliente órfão conectado | Matar processos antigos (`pkill -f barrierc` / `killall barrierc`) e reiniciar |
| Client antigo apontando para IP errado | Restou processo de config anterior | Matar tudo e subir de novo com o IP correto |
| Porta fechada | Firewall | Liberar `24800/tcp` (entrada, no server) |
| Client volta a cair após matar processos | GUI do Barrier aberta relançando clients | Fechar a GUI (`barrier`), deixar só o systemd gerenciar |
| Server para ao fechar janela | `barriers` é app de console iniciado por `.cmd` | Usar o VBScript oculto (seção 1.3) |
| `key: Ctrl = Meta` dá erro de config | Sintaxe de remapeamento é por screen | Usar `ctrl = meta` dentro da seção `screens` |
| `;` digita `ç` no lugar errado | Layout do Windows não bate com o teclado físico | Alinhar layouts (seção 5) |
