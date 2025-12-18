# Sonic Pi + VS Code (macOS)

Quick setup to edit and run Sonic Pi code from VS Code with the **Sonic Pi** extension (publisher `jakearl`).

## Prerequisites
- macOS with the Sonic Pi app installed (keep the app running while coding).
- VS Code installed.
- Extensions: `jakearl.vscode-sonic-pi` (required). Optional: `rebornix.ruby` for lightweight Ruby editing niceties.

## Install the VS Code extension
1. Open VS Code → Extensions view.
2. Search for **Sonic Pi** by Jackson Kearl (ID `jakearl.vscode-sonic-pi`) and install.
3. Keep Sonic Pi app open; the extension sends OSC to its server.
4. Optional CLI install: `code --install-extension jakearl.vscode-sonic-pi rebornix.ruby`
5. Sonic Pi app install (macOS): `brew install --cask sonic-pi` then open it once so `~/.sonic-pi/` and logs are created.

## What’s already set up in this repo
- `.vscode/settings.json` and `.vscode/keybindings.json` wired for Sonic Pi (see snippets below).
- Sample project: `sonic-pi-projects/demo-track/`
  - `main.pi` (entry point)
  - `live_loops/beat.pi` (drums)
  - `notes.md` (track notes placeholder)

### How to run the demo
1. Open `sonic-pi-projects/demo-track/main.pi` in VS Code.
2. Ensure the Sonic Pi app is open.
3. Press `Cmd+Enter` (custom binding) or `Alt+R`/`F5` to run. The beat comes from `live_loops/beat.pi`; the melody lives in `main.pi`.
4. To stop all sound, press `Cmd+.` (custom binding) or run `Sonic Pi: Stop`.

## Commands to run/stop from VS Code
- Command Palette → `Sonic Pi: Run` (runs selection if highlighted, otherwise the whole buffer). Default keys: `F5` or `Alt+R`.
- Command Palette → `Sonic Pi: Run Selected` (forces selection only). Default key: `Alt+T`.
- Command Palette → `Sonic Pi: Stop` (panic/stop all). Default key: `Alt+S`.
- Tip: Add faster bindings in `keybindings.json` (see below).

## Recommended project layout
```
sonic-pi-projects/
  track-name/
    main.pi              # entry buffer you run
    live_loops/          # small loops loaded from main
    samples/             # custom WAV/AIFF
    notes.md             # cues/tempos/structure
```

## VS Code settings (Settings JSON)
Paste into your user or workspace `settings.json`:
```json
{
  "files.associations": {
    "*.pi": "sonic-pi",
    "*.spi": "sonic-pi",
    "*.sonicpi": "sonic-pi",
    "*.rb": "ruby"
  },
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.snippetSuggestions": "inline",
  "editor.wordBasedSuggestions": false,
  "[sonic-pi]": {
    "editor.formatOnSave": false
  },
  "vscode-sonic-pi.launchSonicPiServerAutomatically": "never",  // keep Sonic Pi app in control
  "vscode-sonic-pi.runFileWhenRunSelectedIsEmpty": "always"
}
```

## Keybinding suggestions (keybindings.json)
```json
[
  { "key": "cmd+enter", "command": "vscode-sonic-pi.run", "when": "editorLangId == \"sonic-pi\"" },
  { "key": "cmd+.",     "command": "vscode-sonic-pi.stop", "when": "editorLangId == \"sonic-pi\"" }
]
```

## Fallback OSC runner (if the extension cannot trigger playback)
1. Find the port/token:
   - If `~/.sonic-pi/log/server-output.log` exists, use its API port/token (often `4557`).
   - On Sonic Pi 4.x macOS builds, check `~/.sonic-pi/log/gui.log` for a line like `Setting up OSC sender to Spider on port 35715` (that number is the port to use).
2. Send code via Python (standard library only):
   ```bash
   PORT=4557
   TOKEN=""
   CODE='live_loop :foo do; sample :bd_haus; sleep 0.5; end'

   python3 - <<'PY'
   import os, socket
   port = int(os.environ["PORT"])
   token = os.environ["TOKEN"]
   code = os.environ["CODE"]

   def pad(b): return b + (b'\0' * ((4 - len(b) % 4) % 4))
   def osc(address, args):
       msg = pad(address.encode() + b'\0')
       tags = ',' + ''.join('s' for _ in args)
       msg += pad(tags.encode() + b'\0')
       for a in args: msg += pad(a.encode() + b'\0')
       return msg

   sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
   sock.sendto(osc('/run-code', [token, code, 'vscode-fallback']), ('127.0.0.1', port))
   PY
   ```
3. Stop all jobs:
   ```bash
   PORT=4557
   TOKEN=""
   python3 - <<'PY'
   import os, socket
   port = int(os.environ["PORT"])
   token = os.environ["TOKEN"]

   def pad(b): return b + (b'\0' * ((4 - len(b) % 4) % 4))
   def osc(address, args):
       msg = pad(address.encode() + b'\0')
       tags = ',' + ''.join('s' for _ in args)
       msg += pad(tags.encode() + b'\0')
       for a in args: msg += pad(a.encode() + b'\0')
       return msg

   sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
   sock.sendto(osc('/stop-all-jobs', [token]), ('127.0.0.1', port))
   PY
   ```

## Troubleshooting when playback fails
- Sonic Pi app must be open; test audio in the GUI first.
- Check port/token in `~/.sonic-pi/log/server-output.log` and match VS Code settings if needed. Defaults: host `127.0.0.1`, port `4557`.
- macOS Firewall: allow VS Code incoming connections/UDP localhost (System Settings → Network → Firewall → Options).
- VPN/security tools can block localhost UDP; pause them and retry.
- If code runs but is silent, inspect `~/.sonic-pi/log/scsynth.log` and `~/.sonic-pi/log/server-errors.log` for sample path or synth errors.
- If the extension still cannot reach Sonic Pi, try the fallback OSC scripts above. If those also fail, restart Sonic Pi to reset ports.

## Notes about VS Code extension compatibility (Sonic Pi 4.x on macOS)
- Sonic Pi 4.x no longer ships `port-discovery.rb` (the extension relied on it). We patched the local extension at `~/.vscode/extensions/jakearl.vscode-sonic-pi-0.0.10/out/source-runner/main.js` to read ports from `~/.sonic-pi/log/gui.log` (it auto-detects lines like “Setting up OSC sender to Spider on port …”).
- If you reinstall or update the extension, reapply that patch so port detection keeps working on macOS.
