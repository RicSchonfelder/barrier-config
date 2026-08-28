# Barrier — Referência para IA (machine-readable knowledge base)

Documento denso e estruturado para uma IA ler e agir. Complementa `BARRIER-GUIA.md` (humano) e `PROMPT-IA.md` (prompt genérico).

## FATOS

- Barrier = KVM software (teclado/mouse sobre IP, porta `24800/tcp`). Fork do Synergy.
- Modelo: 1 **server** (tem o teclado físico) + N **clients**.
- O server **injeta** eventos no client; o client **interpreta com o próprio layout** de teclado. Não há remapeamento de caracteres entre máquinas — só de modificadores.
- **Criptografia**: esta instalação roda com `--disable-crypto` em TODOS os lados. TLS desencontrado causa quedas. Não reativar cripto sem alinhar todos os lados.
- Build atual **não suporta** `clipboardSharingSize` (erro: `unknown argument "clipboardSharingSize"`). Única opção de clipboard no conf é o booleano `clipboardSharing`.

## TOPOLOGIA REAL DESTA INSTALAÇÃO (referência)

| Máquina | Papel | IP | Screen name |
|---------|-------|----|-------------|
| Windows | Server | `192.168.100.94` | `desktop` |
| Mac | Client | `192.168.100.83` | `macbook` |
| Linux | Client | `192.168.100.102` | `lab` |

Ordem das telas (esquerda→direita): `macbook` | `desktop` | `lab`.
- Do Windows, borda **esquerda** → Mac; borda **direita** → Linux.
- Server inicia com `--name desktop`.

## ARQUIVO Barrier.conf (Windows server)

Caminho: `C:\Users\<user>\AppData\Local\Barrier\Barrier.conf`

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

Regras do conf:
- `section: screens` lista as telas; modificadores (`ctrl`, `super`, `alt`, `altgr`, `meta`, `shift`) remapeiam **por screen**.
- `section: links` define vizinhança (`.left`/`.right`/`.up`/`.down`). A tela do próprio server é referida pelo seu screen name.
- Nomes de screen são case-insensitive no match, mas use minúsculo em `links` para evitar confusão.
- `clipboardSharing = false` elimina lag por clipboards grandes (≠0 até 6,5 MB observados).

## COMANDOS ESSENCIAIS

### Windows server — iniciar manual
```powershell
Start-Process -FilePath "C:\Program Files\Barrier\barriers.exe" -ArgumentList "--name","desktop","--disable-crypto","-c","$env:USERPROFILE\AppData\Local\Barrier\Barrier.conf"
Get-NetTCPConnection -LocalPort 24800 -State Listen   # confirmar
```

### Windows server — autostart (Agendador de Tarefas, recomendado)
```powershell
$action = New-ScheduledTaskAction -Execute "C:\Program Files\Barrier\barriers.exe" -Argument "--name desktop --disable-crypto -c C:\Users\$env:USERNAME\AppData\Local\Barrier\Barrier.conf"
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit 0
Register-ScheduledTask -TaskName "BarrierServer" -Action $action -Trigger $trigger -Settings $settings -Force
```
- Para iniciar ANTES do login: `-AtStartup` + `-Delay "00:00:30"`.
- Não usar `.cmd`/atalho comum: `barriers` é app de console; fechar a janela mata o server.

### Windows server — log de debug (diagnóstico)
```powershell
Stop-Process -Name barriers -Force
Start-Process -FilePath "C:\Program Files\Barrier\barriers.exe" -ArgumentList "--name","desktop","--disable-crypto","-c","$env:USERPROFILE\AppData\Local\Barrier\Barrier.conf","-d","DEBUG","-l","$env:USERPROFILE\barrier_server.log" -WindowStyle Hidden
```
Ler `barrier_server.log`; grep por `<screen>`, `dead`, `slow`, `clipboard ... size=`, `switch from/to`. Voltar à instância normal (sem `-d/-l`) após o diagnóstico.

### Mac client — manual e launchd
```bash
/Applications/Barrier.app/Contents/MacOS/barrierc -f --no-tray --name macbook --disable-crypto 192.168.100.94:24800
```
launchd plist `~/Library/LaunchAgents/com.barrier.client.plist`: Label `com.barrier.client`, ProgramArguments `barrierc -f --no-tray --name macbook --disable-crypto 192.168.100.94:24800`, `RunAtLoad true`, `KeepAlive true`. Validar: `plutil -lint`, `launchctl load`, `lsof -i :24800`.

Permissões obrigatórias no Mac (não automatizáveis): **Rede Local**, **Acessibilidade**, **Input Monitoring** (Ajustes → Privacidade e Segurança). Sem Rede Local → `No route to host`.

### Linux client — systemd user
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
`systemctl --user daemon-reload && systemctl --user enable --now barrier-client.service`; `ss -tn | grep 24800`.

### Linux client — auto-layout ABNT2 ↔ US Intl (X11)
Quando o server é US International e o Linux é ABNT2, os acentos não batem. Script `~/bin/barrier-layout.sh` (loop de 5s):
- exporta `DISPLAY=:0`, `XAUTHORITY=$HOME/.Xauthority`.
- se `pgrep -x barrier|barrierc|barriers` → `setxkbmap -model abnt2 -layout us -variant intl`.
- senão → `setxkbmap -model abnt2 -layout br -option compose:menu`.
Serviço `~/.config/systemd/user/barrier-layout.service` → `ExecStart=%h/bin/barrier-layout.sh`, `WantedBy=default.target`, `enable --now`.
Em **Wayland** trocar `setxkbmap` por `gsettings set org.gnome.desktop.input-sources sources "[('xkb','us+intl')]"`.

## SINTOMAS → CAUSA → CORREÇÃO (decisão rápida)

| Sintoma | Causa | Correção |
|---------|-------|----------|
| `No route to host` (Mac) | falta permissão **Rede Local** | conceder no Mac (manual) |
| input não entra no Mac | falta Acessibilidade/Input Monitoring | conceder no Mac (manual) |
| `server already has a connected client with our name` | client órfão | `pkill -f barrierc` (Linux) / `killall barrierc` (Mac) |
| porta fechada | firewall | liberar 24800/tcp entrada no server |
| server morre ao fechar janela | iniciado por .cmd | Agendador de Tarefas / VBScript oculto |
| travadas curtas ao trocar tela | clipboard ≥6 MB | `clipboardSharing = false` |
| cursor "vibra" na borda / lag na fronteira | edge jitter | deslocar tela no Configure Server; ou Lock to screen (Scroll Lock) |
| `unknown argument "clipboardSharingSize"` | build sem limite | usar `clipboardSharing = false` |
| acentos/ç errados no Linux | layout ≠ server | auto-layout (script acima) |
| queda intermitente no Mac | TLS/cripto desencontrado | `--disable-crypto` em ambos os lados |
| diagnosticar no server | precisa de log | subir com `-d DEBUG -l logfile` |

## GOTCHAS
- Screen name do server vem de `--name`, não do hostname. Deve casar com a entrada em `links`.
- `barrier` (GUI) vs `barriers` (server daemon) vs `barrierc` (client). Rodar `barriers.exe` direto é mais previsível que a GUI para autostart.
- No Mac, o cliente Barrier tem latência de input um pouco maior por natureza; usar build notarizado.
- Log de debug cresce; usar só para diagnóstico.
- `setxkbmap` no Linux exige a sessão gráfica (DISPLAY/XAUTHORITY); falha em SSH puro sem essas vars.
```
