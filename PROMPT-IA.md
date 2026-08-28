# Prompt para IA — Configurar e diagnosticar Barrier (Windows como server)

Cole este prompt em uma IA com acesso shell às 3 máquinas (SSH para Mac/Linux, PowerShell para Windows).
Substitua os valores fictícios antes de rodar. Use os IPs/screen names reais da sua rede.

```
Configure e torne automático o Barrier (compartilhamento de teclado/mouse via rede) em 3 máquinas.
Windows = SERVER; Mac e Linux = CLIENTS.

TOPOLOGIA (esquerda→direita): <SCREEN_MAC> | <SCREEN_WIN> | <SCREEN_LINUX>
IPs fictícios:
- Windows (server): <IP_WIN>  (ex: 192.168.100.94)  screen: <SCREEN_WIN>
- Mac (client):     <IP_MAC>  (ex: 192.168.100.83)  screen: <SCREEN_MAC>
- Linux (client):   <IP_LINUX>(ex: 192.168.100.102) screen: <SCREEN_LINUX>

REGRAS GERAIS (não negociáveis):
- NÃO use criptografia: --disable-crypto em server E clients. TLS desencontrado = quedas.
- O Barrier NÃO remapeia layout de caracteres, só modificadores. O destino interpreta com o próprio layout; portanto o layout do client deve casar com o do server (origem).
- Remapeamento de modificador é POR SCREEN dentro de `section: screens` (ex.: <SCREEN_MAC>: "ctrl = meta"). NUNCA use "key: X = Y" (erro de config).
- Esta build NÃO suporta limite de tamanho de clipboard (clipboardSharingSize dá "unknown argument"). Para eliminar lag por clipboard grande, use clipboardSharing = false na section: options.

PASSOS:

1) SERVER (Windows, local)
   - Escreva C:\Users\<USER_WIN>\AppData\Local\Barrier\Barrier.conf com:
     section: screens — <SCREEN_WIN>:, <SCREEN_MAC>: (modificadores se quiser), <SCREEN_LINUX>:
     section: links — <SCREEN_WIN>.left=<SCREEN_MAC>; <SCREEN_WIN>.right=<SCREEN_LINUX>; <SCREEN_MAC>.right=<SCREEN_WIN>; <SCREEN_LINUX>.left=<SCREEN_WIN>
     section: options — clipboardSharing = false
   - Libere 24800/tcp (entrada) no firewall do Windows.
   - Autostart robusto via Agendador de Tarefas (não use .cmd que abre janela e mata o server):
     $action = New-ScheduledTaskAction -Execute "C:\Program Files\Barrier\barriers.exe" -Argument "--name <SCREEN_WIN> --disable-crypto -c C:\Users\<USER_WIN>\AppData\Local\Barrier\Barrier.conf"
     $trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME
     $settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit 0
     Register-ScheduledTask -TaskName "BarrierServer" -Action $action -Trigger $trigger -Settings $settings -Force
   - Teste manual se quiser: Start-Process "C:\Program Files\Barrier\barriers.exe" -ArgumentList "--name","<SCREEN_WIN>","--disable-crypto","-c","C:\Users\<USER_WIN>\AppData\Local\Barrier\Barrier.conf"
   - Confirme: Get-NetTCPConnection -LocalPort 24800 -State Listen

2) CLIENT (Mac, via SSH <USER_MAC>@<IP_MAC>)
   - Teste: /Applications/Barrier.app/Contents/MacOS/barrierc -f --no-tray --name <SCREEN_MAC> --disable-crypto <IP_WIN>:24800
   - Se "No route to host" mesmo com porta ok, AVISE o usuário que ele precisa conceder Rede Local + Acessibilidade + Input Monitoring em Ajustes→Privacidade (não automatizável).
   - launchd: ~/Library/LaunchAgents/com.barrier.client.plist com Label com.barrier.client, ProgramArguments=[barrierc, -f, --no-tray, --name, <SCREEN_MAC>, --disable-crypto, <IP_WIN>:24800], RunAtLoad true, KeepAlive true. plutil -lint; launchctl load. lsof -i :24800 → ESTAB para <IP_WIN>.

3) CLIENT (Linux, via SSH <USER_LINUX>@<IP_LINUX>)
   - sudo apt install barrier (se faltar).
   - systemd user: ~/.config/systemd/user/barrier-client.service com ExecStart=/usr/bin/barrierc -f --no-tray --name <SCREEN_LINUX> --disable-crypto <IP_WIN>:24800, Restart=always, RestartSec=5, WantedBy=default.target. systemctl --user daemon-reload && enable --now. ss -tn | grep 24800 → ESTAB para <IP_WIN>.
   - AUTO-LAYOUT (quando server é US International e Linux é ABNT2): criar ~/bin/barrier-layout.sh que, num loop, faz setxkbmap para "us intl" se pgrep barrier/barrierc/barriers estiver ativo, senão volta para "br" (model abnt2, option compose:menu). Criar ~/.config/systemd/user/barrier-layout.service (ExecStart=%h/bin/barrier-layout.sh, WantedBy=default.target) e enable --now. O script deve exportar DISPLAY=:0 e XAUTHORITY=$HOME/.Xauthority. Em Wayland troque setxkbmap por gsettings.

4) VALIDAÇÃO
   - Em cada máquina: conexão ESTABLISHED na 24800.
   - Matar client órfão (pkill -f barrierc no Linux / killall barrierc no Mac) que cause "server already has a connected client with our name".
   - Conceder permissões no Mac se faltarem.
   - Conferir layouts: server US Intl → Linux deve estar em us intl (script 3); server ABNT2 → Linux br.

5) DIAGNÓSTICO DE LAG/QUEDA NO MAC (se necessário)
   - No Windows: Stop-Process -Name barriers -Force; depois Start-Process barriers.exe -ArgumentList "--name","<SCREEN_WIN>","--disable-crypto","-c","<conf>","-d","DEBUG","-l","$env:USERPROFILE\barrier_server.log" -WindowStyle Hidden
   - Reproduza a falha e leia barrier_server.log (grep por <SCREEN_MAC>, dead, slow, "clipboard ... size=", switch). Volte à instância normal (sem -d/-l) depois.
   - Edge jitter (cursor vibrando na borda): desloque a tela do Mac no Configure Server ou use Lock to screen (Scroll Lock).
```

> Veja também `AI-REFERENCE.md` (fatos estruturados) e `SKILL.md` (skill para IAs).
