# Prompt para IA — Configurar Barrier (Windows como server)

Cole este prompt em uma IA com acesso shell às 3 máquinas via SSH/PowerShell. Substitua `<IP_WINDOWS>`, `<IP_MAC>`, `<IP_LINUX>`, `<USER_WIN>`, `<USER_MAC>`, `<USER_LINUX>`, `<SCREEN_WIN>`, `<SCREEN_MAC>`, `<SCREEN_LINUX>` antes de executar. Os valores em destaque são fictícios.

```
Configure o Barrier (compartilhamento de teclado/mouse via rede) em 3 máquinas.
Windows é o SERVER; Mac e Linux são CLIENTS.

TOPOLOGIA (layout da esquerda para a direita):
Linux (<SCREEN_LINUX>) | Mac (<SCREEN_MAC>) | Windows (<SCREEN_WIN>)

IPs (fictícios):
- Windows (server): <IP_WINDOWS>  ex: 192.168.1.50  screen: <SCREEN_WIN>
- Mac (client):     <IP_MAC>      ex: 192.168.1.60  screen: <SCREEN_MAC>
- Linux (client):   <IP_LINUX>    ex: 192.168.1.70  screen: <SCREEN_LINUX>

Passos:

1. SERVER (Windows, máquina local)
   - Criar o arquivo de config em C:\Users\<USER_WIN>\AppData\Local\Barrier\Barrier.conf com:
     section: screens — <SCREEN_WIN>, <SCREEN_MAC>, <SCREEN_LINUX>
     section: links — <SCREEN_WIN>.left = <SCREEN_MAC>; <SCREEN_MAC>.right = <SCREEN_WIN> e .left = <SCREEN_LINUX>; <SCREEN_LINUX>.right = <SCREEN_MAC>
   - Garantir que a porta 24800/tcp está liberada no firewall (inbound).
   - Iniciar o server headless:
     Start-Process -FilePath "C:\Program Files\Barrier\barriers.exe" -ArgumentList '--config','C:\Users\<USER_WIN>\AppData\Local\Barrier\Barrier.conf','--disable-crypto','-f'
   - Verificar escuta: netstat -ano | findstr 24800
   - Criar autostart: arquivo C:\Users\<USER_WIN>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\barrier-server.cmd contendo:
     @echo off
     start "" "C:\Program Files\Barrier\barriers.exe" --config C:\Users\<USER_WIN>\AppData\Local\Barrier\Barrier.conf --disable-crypto -f

2. CLIENT (Mac, via SSH user <USER_MAC>@<IP_MAC>)
   - Testar manualmente:
     /Applications/Barrier.app/Contents/MacOS/barrierc -f --no-tray --name <SCREEN_MAC> --disable-crypto <IP_WINDOWS>:24800
   - Se der erro "No route to host" mesmo com nc ok na porta, avisar o usuário que precisa conceder permissão de Rede Local + Acessibilidade (Ajustes > Privacidade e Segurança) — não dá para automatizar.
   - Criar autostart com launchd: ~/Library/LaunchAgents/com.barrier.client.plist com Label com.barrier.client, ProgramArguments apontando para o barrierc com --name <SCREEN_MAC> --disable-crypto <IP_WINDOWS>:24800, RunAtLoad true, KeepAlive true.
   - plutil -lint, depois launchctl load. Confirmar com lsof -i :24800 (deve ter ESTABLISHED para <IP_WINDOWS>).

3. CLIENT (Linux, via SSH user <USER_LINUX>@<IP_LINUX>)
   - Instalar se necessário: sudo apt install barrier
   - Criar ~/.config/systemd/user/barrier-client.service com ExecStart=/usr/bin/barrierc -f --no-tray --name <SCREEN_LINUX> --disable-crypto <IP_WINDOWS>:24800, Restart=always, RestartSec=5.
   - systemctl --user daemon-reload && systemctl --user enable barrier-client.service
   - loginctl enable-linger <USER_LINUX>
   - systemctl --user start barrier-client.service
   - Confirmar: ss -tn | grep 24800 → ESTAB para <IP_WINDOWS>.

4. VALIDAÇÃO FINAL
   - Em cada máquina confirmar conexão ESTABLISHED na porta 24800 com o Windows.
   - Matar qualquer client órfão (pkill -f barrierc no Linux, killall barrierc no Mac) que aponte para IP errado ou cause "server already has a connected client with our name".
   - NÃO habilitar SSL/crypto em nenhum lado (usar --disable-crypto em todos os clients e server).
```
