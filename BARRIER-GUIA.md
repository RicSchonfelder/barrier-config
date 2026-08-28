# Configuração do Barrier — Guia para Humanos

Compartilha teclado e mouse entre 3 máquinas via rede local.

**Layout (esquerda → direita):** `macbook` (Mac) | `desktop` (Windows, **server**) | `lab` (Linux).
Mouse no Windows: borda **esquerda** → Mac; borda **direita** → Linux.

> **Exemplo real desta instalação** (use estes valores como referência; adapte à sua rede):
> | Máquina | Papel | IP | Screen name |
> |---------|-------|----|-------------|
> | Windows | Server | `192.168.100.94` | `desktop` |
> | Mac | Client | `192.168.100.83` | `macbook` |
> | Linux | Client | `192.168.100.102` | `lab` |

---

## Pré-requisitos

- Barrier 2.4+ em todas (repo: https://github.com/debauchee/barrier)
- Mesma sub-rede; porta **24800/tcp** liberada (entrada no server Windows)
- **NÃO usar criptografia** em nenhum lado: server e clients com `--disable-crypto`

---

## 1. Windows — Server

### 1.1 Arquivo de config (`Barrier.conf`)

Caminho: `C:\Users\<usuario>\AppData\Local\Barrier\Barrier.conf`

```ini
section: screens
	Desktop:
	macbook:
		ctrl = ctrl
		super = alt
		alt = meta
	lab:
end

section: links
	desktop:
		left = macbook
		right = lab
	macbook:
		right = desktop
	lab:
		left = desktop
end

section: options
	clipboardSharing = false
end
```

- O nome da tela do server (`desktop`) é passado em `--name desktop` ao iniciar.
- Remapeamento de modificadores é **por screen** (ex.: `macbook` troca `super`/`alt` para os atalhos ficarem nativos no Mac). Não use `key: X = Y` (dá erro de config).
- `clipboardSharing = false` desliga o compartilhamento de área de transferência. **Esta build não aceita limite de tamanho** (`clipboardSharingSize` dá *unknown argument*). Se precisar de clipboard, remova a `section: options` ou use outra build.

### 1.2 Iniciar manualmente (teste)

```powershell
Start-Process -FilePath "C:\Program Files\Barrier\barriers.exe" `
  -ArgumentList "--name","desktop","--disable-crypto","-c","$env:USERPROFILE\AppData\Local\Barrier\Barrier.conf"
```

Verificar escuta:

```powershell
Get-NetTCPConnection -LocalPort 24800 -State Listen
```

### 1.3 Autostart recomendado — Agendador de Tarefas

Mais robusto que o atalho de Inicialização (não depende de abrir o GUI e sobe no login). Cria a tarefa `BarrierServer`:

```powershell
$action = New-ScheduledTaskAction -Execute "C:\Program Files\Barrier\barriers.exe" `
  -Argument "--name desktop --disable-crypto -c C:\Users\$env:USERNAME\AppData\Local\Barrier\Barrier.conf"
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit 0
Register-ScheduledTask -TaskName "BarrierServer" -Action $action -Trigger $trigger -Settings $settings -Force
```

Gerenciar: *Gerenciador de Tarefas → Biblioteca do Agendador de Tarefas → BarrierServer*.
Para iniciar **antes do login** (clientes conectam já na tela de boas-vindas), troque o trigger para `New-ScheduledTaskTrigger -AtStartup` e adicione um atraso (`-Delay "00:00:30"`) para a rede subir.

> **Alternativa (Inicialização, abre GUI oculto):** VBScript em
> `AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\barrier-server.vbs`:
> ```vbs
> CreateObject("Wscript.Shell").Run """C:\Program Files\Barrier\barriers.exe"" --name desktop --disable-crypto -c C:\Users\<user>\AppData\Local\Barrier\Barrier.conf", 0, False
> ```

---

## 2. Mac — Client

### 2.1 Teste manual

```bash
/Applications/Barrier.app/Contents/MacOS/barrierc -f --no-tray --name macbook --disable-crypto 192.168.100.94:24800
```

### 2.2 Autostart (launchd)

`~/Library/LaunchAgents/com.barrier.client.plist`:

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
        <string>192.168.100.94:24800</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

```bash
plutil -lint ~/Library/LaunchAgents/com.barrier.client.plist
launchctl load ~/Library/LaunchAgents/com.barrier.client.plist
lsof -i :24800   # ESTABLISHED para 192.168.100.94
```

### 2.3 Permissões (obrigatórias)

*Ajustes do Sistema → Privacidade e Segurança*:
- **Rede Local** — sem ela o client falha com `No route to host`
- **Acessibilidade**
- **Monitoramento de Entrada/Saída** (Input Monitoring — macOS 11+)

Não dá para automatizar; o usuário precisa conceder.

### 2.4 Latência do cliente Mac

O cliente Barrier no macOS tem latência de input um pouco maior por natureza. Use um build notarizado compatível com seu macOS (a release 2.4.0 do GitHub costuma abrir; se o Gatekeeper bloquear, use a build citada nas issues do projeto).

---

## 3. Linux — Client

### 3.1 Instalar e testar

```bash
sudo apt install barrier
/usr/bin/barrierc -f --no-tray --name lab --disable-crypto 192.168.100.94:24800
```

### 3.2 Autostart (systemd user)

`~/.config/systemd/user/barrier-client.service`:

```ini
[Unit]
Description=Barrier client
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/barrierc -f --no-tray --name lab --disable-crypto 192.168.100.94:24800
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now barrier-client.service
ss -tn | grep 24800   # ESTAB para 192.168.100.94
```

### 3.3 Auto-layout: ABNT2 ↔ US International

Quando o **server (Windows) usa US International** e o **client Linux é ABNT2**, os acentos não batem. Solução: o Linux troca para `us intl` enquanto o Barrier estiver rodando e volta ao ABNT2 quando parar. (X11/Xorg; em Wayland use `gsettings` no GNOME ou equivalente.)

`~/bin/barrier-layout.sh`:

```bash
#!/usr/bin/env bash
export DISPLAY="${DISPLAY:-:0}"
export XAUTHORITY="${XAUTHORITY:-$HOME/.Xauthority}"

NATIVE="-model abnt2 -layout br -option compose:menu"
USINTL="-model abnt2 -layout us -variant intl"

last=""
while true; do
  if pgrep -x "barrier" >/dev/null 2>&1 \
  || pgrep -x "barrierc" >/dev/null 2>&1 \
  || pgrep -x "barriers" >/dev/null 2>&1; then
    cur="barrier"
  else
    cur="native"
  fi
  if [ "$cur" != "$last" ]; then
    if [ "$cur" = "barrier" ]; then setxkbmap $USINTL; else setxkbmap $NATIVE; fi
    last="$cur"
  fi
  sleep 5
done
```

`~/.config/systemd/user/barrier-layout.service`:

```ini
[Unit]
Description=Alterna layout de teclado conforme o Barrier

[Service]
ExecStart=%h/bin/barrier-layout.sh

[Install]
WantedBy=default.target
```

```bash
mkdir -p ~/bin ~/.config/systemd/user
chmod +x ~/bin/barrier-layout.sh
systemctl --user daemon-reload
systemctl --user enable --now barrier-layout.service
```

> O `setxkbmap` precisa do `DISPLAY`/`XAUTHORITY` da sessão gráfica (definidos no script). Funciona em XFCE/X11 como o desta instalação.

---

## 4. Layout de teclado entre máquinas

O Barrier injeta os eventos e o **destino interpreta com o próprio layout** (não há remapeamento de caracteres, só de modificadores). Portanto, para `ç`/acentos coincidirem, o layout do destino deve casar com o do **server (origem)**:

- Server Windows em **US International** → client Linux deve estar em `us intl` (seção 3.3 resolve isso automaticamente).
- Server Windows em **ABNT2** (`0416:00010416`) → client Linux em `br` (`setxkbmap -query` → `layout: br`).
- Client Mac: defina o mesmo layout do server nas Preferências de Teclado.

---

## 5. Solução de problemas

| Sintoma | Causa provável | Correção |
|---------|----------------|----------|
| `No route to host` (Mac) | Permissão **Rede Local** negada | Conceder em Ajustes → Privacidade → Rede Local |
| Entrada não funciona no Mac | Faltam **Acessibilidade** / **Input Monitoring** | Conceder nas Preferências |
| `server already has a connected client with our name` | Cliente órfão | `pkill -f barrierc` (Linux) / `killall barrierc` (Mac) e reiniciar |
| Porta fechada | Firewall | Liberar `24800/tcp` (entrada no server) |
| Server morre ao fechar janela | `barriers` iniciado por `.cmd` | Usar Agendador de Tarefas (1.3) ou VBScript oculto |
| **Travadas curtas ao trocar de tela** | Clipboard gigante (≥6 MB) sendo enviado | `clipboardSharing = false` no `Barrier.conf` (veja 1.1) |
| **Cursor "vibrando" na borda / lag na fronteira** | *Edge jitter*: oscilação entre telas na emenda | Deslocar a tela do Mac no *Configure Server* (não alinhar 100% as bordas) ou travar com `Scroll Lock` (*Lock to screen*) |
| `unknown argument "clipboardSharingSize"` | Build não suporta limite de tamanho | Usar `clipboardSharing = false` |
| Acentos/ç errados no Linux | Layout do Linux ≠ do server | Script de auto-layout (3.3) ou alinhar layouts (seção 4) |
| Conexão cai/intermitente no Mac | TLS/cripto desencontrado | Garantir `--disable-crypto` em **ambos** os lados |
| Quer diagnosticar no server | Precisa de log de debug | Veja abaixo |

### 5.1 Capturar log de debug do server (Windows)

Útil para lags/quedas do Mac — mostra handshake, switches de tela e tamanhos de clipboard:

```powershell
# mata a instância atual
Stop-Process -Name barriers -Force
# sobe com debug + arquivo de log
Start-Process -FilePath "C:\Program Files\Barrier\barriers.exe" `
  -ArgumentList "--name","desktop","--disable-crypto","-c","$env:USERPROFILE\AppData\Local\Barrier\Barrier.conf","-d","DEBUG","-l","$env:USERPROFILE\barrier_server.log" -WindowStyle Hidden
```

Depois reproduza a falha e leia `barrier_server.log` (procure `macbook`, `dead`, `slow`, `clipboard ... size=`, `switch from/to`). Para voltar ao normal, rode a instância sem `-d/-l` (ex.: pela tarefa do Agendador).

> O log de debug cresce; use só para diagnóstico e volte à instância normal depois.

---

## 6. Teste final

Em cada máquina, confirme `ESTABLISHED` na 24800:

```bash
ss -tn | grep 24800       # Linux
lsof -i :24800            # Mac
Get-NetTCPConnection -LocalPort 24800 -State Established   # Windows
```

Movimento: Windows → borda esquerda = Mac; Windows → borda direita = Linux.
