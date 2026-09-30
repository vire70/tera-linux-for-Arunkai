# TERA Linux Launcher — Arun Kai Edition

A native Linux launcher for the **Arun Kai** TERA private server, forked from
[PopusBenedictus/tera-launcher-for-linux](https://github.com/PopusBenedictus/tera-launcher-for-linux)
(archived, WTFPL license). This fork bundles a working `launcher-config.json`
for Arun Kai and fixes several issues that prevented the original from
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

3. Run the AppImage again, log in with your Arun Kai account, and hit **Play**.

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
(`LauncherLoginAction`). Arun Kai's server uses a different, multi-step
flow, so this fork's `gui/main.c` differs from upstream in a few places:

- **Login rewritten** to call `LoginAction`, then `GetAccountInfoAction`,
  `GetAuthKeyAction`, and `GetCharacterCountAction` in sequence on the same
  session cookie, matching what Arun Kai's own web launcher does. Also
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
original project was built for, or the four-call style Arun Kai uses — the
login code here assumes the latter.

## Credit

Built on [PopusBenedictus/tera-launcher-for-linux](https://github.com/PopusBenedictus/tera-launcher-for-linux).
See `COPYING`/`COPYING.WTFPL` for license terms.
