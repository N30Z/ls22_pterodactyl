# Farming Simulator 22 – Pterodactyl Egg

Ein Pterodactyl-Egg zum Betrieb eines **Farming Simulator 22 Dedicated Servers** unter Linux via **Wine** und **SteamCMD**.

FS22 liefert keinen nativen Linux-Server – Giants stellt nur eine Windows-`dedicatedServer.exe` bereit. Dieses Egg führt sie unter Wine aus, lädt die Serverdateien per SteamCMD und hält im Hintergrund einen echten Steam-Client eingeloggt, damit der von der GIANTS-Engine genutzte `steam://`-Startmechanismus funktioniert.

## Voraussetzungen

- Eine **zweite, lizenzierte Steam-Kopie** von Farming Simulator 22 (App-ID `1248130`). Ein anonymer SteamCMD-Login ist nicht möglich.
- Auf dem Steam-Account muss **Steam Guard über den Mobile Authenticator** aktiv sein (nicht Email). Der Mobile Authenticator erzeugt ein persistentes Refresh-Token, das über Neustarts hinweg gültig bleibt. Email-Guard funktioniert nicht für den Dauerbetrieb, weil bei jedem Login ein neuer, manuell zu holender Code verlangt würde.
- Docker-Registry-Zugriff auf `ghcr.io` (Docker Hub / `docker.io` wird nicht benötigt).

## Docker-Images

- **Laufzeit:** `ghcr.io/ptero-eggs/yolks:wine_latest`
- **Installation:** `ghcr.io/parkervcp/installers:debian`

Das Wine-Staging-Image (`wine_staging`) hat sich in der Entwicklung als stabiler erwiesen; falls `wine_latest` Probleme macht, kann testweise darauf gewechselt werden.

## Variablen

| Variable | Beschreibung | Editierbar |
|----------|--------------|------------|
| `STEAM_USER` | Steam-Account mit gültiger FS22-Lizenz | ja |
| `STEAM_PASS` | Passwort dieses Accounts | ja |
| `STEAM_AUTH` | Nicht genutzt (Mobile Authenticator vorausgesetzt) | nein |
| `SERVER_NAME` | Anzeigename des Servers | ja |
| `GAME_PORT` | Port des Gameservers (Standard `10823`) – muss zusätzlich im Webpanel gesetzt werden | ja |
| `WEB_PORT` | Port des Webinterfaces (Standard `8080`) | ja |
| `WEB_USER` | Login für das Webinterface | ja |
| `WEB_PASS` | Passwort für das Webinterface | ja |
| `WINETRICKS_RUN` | Wine-Komponenten (`vcrun2019 corefonts d3dcompiler_47`) | nein |
| `SRCDS_APPID` | Steam App-ID (`1248130`) | nein |
| `WINEDEBUG`, `WINEARCH`, `WINEPATH` | Wine-Umgebung | nein |

> **Hinweis:** `GAME_PORT`/`SERVER_PORT` darf nicht als eigene Variable heißen – `SERVER_PORT` und `SERVER_IP` sind in Pterodactyl reserviert und werden automatisch aus der primären Allocation gesetzt.

## Installation

1. Egg im Pterodactyl-Panel importieren (Nests → Import Egg).
2. Neuen Server anlegen, dabei `STEAM_USER`, `STEAM_PASS`, `WEB_PASS` etc. auf echte Werte setzen.
3. Während der Installation lädt SteamCMD die Serverdateien. Der Login erfolgt in getrennten Sessions mit Retry; der `Waiting for client config...`-Segfault von SteamCMD ist harmlos (der Download läuft trotzdem durch, der Erfolg wird an der heruntergeladenen `dedicatedServer.exe` geprüft).
4. Bei der ersten Steam-Anmeldung eine Bestätigung im **Steam Mobile Authenticator** durchführen.

## Serverstart

1. Server im Panel starten – `start.sh` läuft:
   - startet ein virtuelles Display (Xvfb),
   - installiert einmalig den echten Steam-Client unter Wine,
   - loggt den Client ein (Bestätigung im Mobile Authenticator),
   - startet das Webinterface (`dedicatedServer.exe`).
2. Webinterface im Browser öffnen: `http://<Server-IP>:<WEB_PORT>` und mit `WEB_USER` / `WEB_PASS` einloggen.
3. Dort Karte, Mods, Passwort etc. konfigurieren und den **Spielserver über den Start-Button starten**. Der eigentliche Spielprozess wird vom Webpanel gestartet, nicht direkt vom Script.
4. Sobald das Spiel läuft und `Started network game` ins Log schreibt, wechselt der Pterodactyl-Status auf **Running**. Das GIANTS-Serverlog wird live in die Panel-Konsole gespiegelt.

## Wie es funktioniert

Der Ablauf umschifft mehrere Wine/Steam-Eigenheiten, die einen FS22-Server unter Linux normalerweise blockieren:

- **32-bit-Kompatibilität:** SteamCMD ist 32-bit; das Install-Script installiert `libc6:i386` & Co, um Segfaults vor dem Download zu vermeiden.
- **Getrennte Login-/Update-Sessions:** verhindert einen reproduzierbaren SteamCMD-Segfault, der sonst den Download abbricht.
- **Echter Steam-Client statt nur SteamCMD:** FS22 startet das Spiel intern über das `steam://`-Protokoll. Ohne laufenden Steam-Client meldet `SteamAPI_IsSteamRunning()` „nein" und der Start scheitert. Das Script hält daher einen echten, eingeloggten Client im Hintergrund.
- **Ein einzelnes Xvfb-Display:** alle Wine-Prozesse teilen sich dasselbe virtuelle Display; parallele `xvfb-run`-Instanzen würden sich gegenseitig abschießen.
- **Log-Spiegelung:** alte Logs werden weggeräumt, das frische GIANTS-`log_*.txt` wird erkannt und live in die Panel-Konsole getailt.

## Bekannte Einschränkungen

- **Doppelte Login-Bestätigung beim Start:** Es laufen zwei Steam-Anmeldungen (SteamCMD beim Boot + echter Client). Falls das Image eine `AUTO_UPDATE`-Variable bietet, kann sie auf `0` gesetzt werden, um den SteamCMD-Login beim reinen Restart zu vermeiden.
- **Manueller Start im Webpanel nötig:** Der eigentliche Spielserver startet nicht vollautomatisch beim Container-Start, sondern erst nach einem Klick im Webinterface.
- **Steam Guard per Email wird nicht unterstützt** – nur der Mobile Authenticator ermöglicht persistente Sessions.

## Dateien im Container

| Pfad | Inhalt |
|------|--------|
| `/home/container/start.sh` | Start-Wrapper (Xvfb, Steam-Client, Webpanel, Log-Spiegelung) |
| `/home/container/webpanel.log` | stdout/stderr des Webinterfaces |
| `/home/container/steamclient.log` | Ausgabe des Steam-Clients |
| `/home/container/xvfb.log` | Ausgabe des virtuellen Displays |
| `.../Documents/My Games/FarmingSimulator2022/log_*.txt` | GIANTS-Serverlog (aktueller Lauf) |
| `.../Documents/My Games/FarmingSimulator2022/old_logs/` | Ältere Serverlogs |
| `/home/container/dedicatedServer.xml` | Webpanel-Konfiguration (Port, Zugangsdaten, TLS) |
