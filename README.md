# TERA Linux Launcher — Arunkai Edition

A native Linux launcher for the **Arunkai** TERA private server, forked from
[PopusBenedictus/tera-launcher-for-linux](https://github.com/PopusBenedictus/tera-launcher-for-linux)
(archived, WTFPL license). This fork bundles a working `launcher-config.json`
for Arunkai and fixes several issues that prevented the original from
working against this specific server.

## What this gets you

Login, patch-checking, and launching TERA (v100.02 client) through Wine —
no need to run the Windows `ARK TERA.exe` launcher under Wine, which does
not work reliably.

## Requirements

- A working install of **[podman](https://podman.io/)** (or Docker — swap
  `podman` for `docker` in the commands below)
- The unpacked TERA v100.02 client game files somewhere on disk (~67GB)
- Roughly 500MB free for the built AppImage (it bundles GE-Proton)

## Building

```bash
git clone https://github.com/vire70/tera-linux-for-Arunkai.git
cd tera-linux-for-Arunkai/appimage

podman build -t tera-builder .
podman run --rm -it -v "$(pwd)/..:/src:Z" -w /src/appimage tera-builder bash -lc "./build-appimage.sh"
```

This will take a while the first time (downloads a toolchain and GE-Proton
inside the container). When it finishes, `TERA_Launcher_for_Linux-x86_64.AppImage`
will be sitting in the repo root.

## First-time setup

1. Make it executable and run it once, so it creates its config folder:
```bash
   chmod +x TERA_Launcher_for_Linux-x86_64.AppImage
   ./TERA_Launcher_for_Linux-x86_64.AppImage
```
   Close it after it opens — this step just generates
   `~/.arunkai-tera/config/tera-launcher-config.ini`.

2. Edit that file and point `gameprefix` at the folder **containing**
   `Binaries/` in your TERA install — for example:
```ini
   gameprefix=/path/to/your/TERA
```
   (Not the `Binaries` folder itself — the folder that
   directly contains `Binaries`.)

3. Run the AppImage again, log in with your Arunkai account, and hit **Play**.

## Notes

- **Every launch runs a dependency-install pass** (winetricks, vcrun2022, etc.) 
  before starting the game — this is normal and can take
  up to a minute, even after the prefix is fully set up, since winetricks
  re-checks each time.
- The prefix and game files live under `~/.arunkai-tera/` by default
  (`wineprefix/`, `files/` — unused since `gameprefix` overrides it, and
  `config/`).
- If the game exits immediately after "Launching the Game" the very first
  time you press Play, try Play again — the first run sometimes finishes
  installing a required runtime file right as the game tries to start.
  Second attempts onward should be consistent.

## Changes from upstream

The original project assumes a single-call login endpoint
(`LauncherLoginAction`). Arunkai's server uses a different, multi-step
flow, so this fork's `gui/main.c` differs from upstream in a few places:

- **Login rewritten** to call `LoginAction`, then `GetAccountInfoAction`,
  `GetAuthKeyAction`, and `GetCharacterCountAction` in sequence on the same
  session cookie, matching what Arunkai's own web launcher does. Also
  URL-encodes the submitted password (the original sent it raw).
- **Game path conversion fixed** — the path handed to the game process now
  gets its Unix-style slashes converted to Windows-style backslashes before
  being passed to `CreateProcessA` inside Wine; without this, the game
  failed to launch with a "file not found" error despite the file existing.
- **`vkd3d` and `corefonts` removed** from the winetricks verb list in
  `prepare_wineprefix` — both verbs fail against currently-available
  download sources, and winetricks aborts its entire batch on the first
  failed verb, which was silently preventing `vcrun2022`, `ucrtbase2019`,
  and `dxvk` from ever installing. TERA doesn't need vkd3d (it's a DX9
  game), so dropping it is harmless.
- **Restored `WINEDEBUG` output** (was hardcoded to `-all`, i.e. fully
  silenced) so Wine-side errors are visible in the terminal if something
  goes wrong.

If you're adapting this fork for a *different* TERA private server, check
whether it uses the single-endpoint `LauncherLoginAction` style the
original project was built for, or the four-call style Arunkai uses — the
login code here assumes the latter.

## TERA Toolbox (optional)

[TERA Toolbox](https://github.com/tera-private-toolbox/tera-toolbox) adds
client-side mods (auto-loot, bugfix, FPS tweaks, etc.) via a local network
proxy. This launcher can start it automatically before the game.

**Important:** use the **private-server fork**
(`tera-private-toolbox/tera-toolbox`), not the official upstream one — the
official release doesn't have an opcode map for Arun Kai's protocol
version and every mod will fail silently with "unmapped packet" errors in
Toolbox's own log.

### Installing Toolbox

1. Download the **Setup** release from the
   [private-toolbox releases page](https://github.com/tera-private-toolbox/tera-toolbox/releases).
2. Rename it from `TeraToolboxSetup.exe` to `TeraToolbox.exe` (the launcher
   looks for this exact filename) and place it in its own folder, e.g.
   `/path/to/TERA-Toolbox/TeraToolbox.exe`.
3. Edit `~/.arunkai-tera/config/tera-launcher-config.ini` and set:
```ini
   use_tera_toolbox=true
   tera_toolbox_path=/path/to/TERA-Toolbox
```
   (point at the **folder**, not the `.exe`. **Note**: You will have to add
   "tera_toolbox_path=" as a line yourself.)
   
4. Launch as normal. On first run, Toolbox will install itself into that
   folder and may prompt that it's "already installed, continue anyway?"
   — choose **yes** (declining cancels the install and it won't run). This
   install prompt currently reappears on every launch; it's cosmetic and
   safe to click through, just make sure you always pick default location and to launch after install.
5. Once installed, Toolbox opens its own window where you can enable/
   disable individual mods before playing.

### Known quirks

- **Startup can pause for up to a minute or two** with the terminal
  repeating `RtlpWaitForCriticalSection ... wait timed out`. This is two
  Wine processes (Toolbox and the game) briefly contending for the same
  prefix on startup — it resolves on its own; just wait it out.
- Server select will show both the normal server and a second
  `(Toolbox)` entry — pick the `(Toolbox)` one to actually route through
  the proxy and get mod functionality.

## Credit

Built on [PopusBenedictus/tera-launcher-for-linux](https://github.com/PopusBenedictus/tera-launcher-for-linux).
See `COPYING`/`COPYING.WTFPL` for license terms.
