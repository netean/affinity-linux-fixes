# affinity-linux-fixes

One command to fix the most annoying bugs in **Affinity (by Canva, v3)** running under Wine on Linux, and one command to update it.

It's for people who already have Affinity installed with the [AffinityOnLinux](https://github.com/seapear/AffinityOnLinux) guides (Lutris, Flatpak Lutris or plain Wine). It doesn't install Wine or Affinity for you.

```bash
git clone https://github.com/netean/affinity-linux-fixes.git
cd affinity-linux-fixes
./affinity-fix install
```

Then launch Affinity as usual. To update Affinity later:

```bash
affinity-fix update
```

## What it fixes

| Problem | Fix | Status |
|---|---|---|
| Menu changes, custom workspaces and preferences aren't saved | [WineFix](https://github.com/noahc3/AffinityPluginLoader) patches the save-on-exit bug | ✅ Confirmed |
| Affinity doesn't fully close, so Lutris still shows it running | Disables Serif's crash reporter (`crashpad_handler.exe`), which kept the process alive after quitting | ✅ Confirmed |
| Canva sign-in: the browser login works but Affinity crashes when the login is passed back | Installs the missing .NET WinRT libraries, [LoginFix](https://github.com/seapear/AffinityOnLinux/tree/main/Guides/Wine/LoginFix), and an `affinity://` link handler that doesn't crash on the long callback URL | 🧪 Testing |
| Vector pen preview line is drawn in the wrong place | WineFix's patched `d2d1.dll` | ✅ Included (upstream fix) |
| Colour picker zoom preview is black on Wayland | WineFix | ✅ Included (upstream fix) |
| Colour picker can't sample outside the canvas | — | ❌ No fix exists yet (Wine limitation) |
| Random crashes at startup | WineFix's font-enumeration fix | ✅ Included (upstream fix) |
| Updating Affinity is painful | `affinity-fix update` | ✅ Confirmed |

"Confirmed" means tested on Nobara 44 (KDE, Wayland) with Flatpak Lutris, the ElementalWarrior Wine runner and Affinity 3.3.3. If it works or breaks on your setup, please open an issue.

## Commands

| Command | What it does |
|---|---|
| `affinity-fix install` | Finds your Affinity setup, backs up the Wine prefix and applies every fix. Safe to re-run. |
| `affinity-fix update` | Downloads the latest Affinity installer if it has changed, backs up, runs the installer (you click through it), then re-applies the fixes. |
| `affinity-fix update --force` | Reinstalls Affinity even if the installer hasn't changed. |
| `affinity-fix plugins` | Updates and re-applies only the plugin fixes. |
| `affinity-fix check` | Shows the installed and latest Affinity versions, and whether each fix is in place. |
| `affinity-fix uninstall` | Removes the fixes. Affinity, your settings and the backups are left alone. |

### Install options

Auto-detection looks for a Lutris game that runs `Affinity.exe` (Flatpak or native Lutris). If there isn't one, it falls back to `~/.affinity`, the AffinityOnLinux default. You can override everything:

```bash
./affinity-fix install --prefix ~/.affinity --wine /usr/bin/wine
./affinity-fix install --lutris-yml ~/.local/share/lutris/games/affinity-123.yml
./affinity-fix install --no-backup
```

Settings are saved to `~/.config/affinity-fix/config`.

## Requirements

- Affinity v3 already installed in a Wine prefix with `dotnet48` (as in the AffinityOnLinux guides)
- `curl`, `python3`, `7z` (p7zip), `flatpak` if you use Flatpak Lutris
- **One of** `podman`, `docker` or the .NET SDK. LoginFix is only published as source code, so it's compiled during install. Using podman or docker, it builds in a throwaway `mcr.microsoft.com/dotnet/sdk:8.0` container and leaves nothing on your system. Without any of them, everything except Canva sign-in still gets fixed.

## What it changes

Everything is downloaded from the original source at install time. Nothing third-party is bundled in this repo.

1. **Backup.** It makes a copy of your Wine prefix at `<prefix>.backup-YYYYMMDD-HHMM`. On btrfs/XFS this is an instant copy-on-write snapshot that uses almost no space. The 3 newest backups are kept, and backups you made yourself are never touched.
2. **[AffinityPluginLoader + WineFix](https://github.com/noahc3/AffinityPluginLoader)** (by noahc3). The latest release is extracted into `drive_c/Program Files/Affinity/Affinity/`. Affinity is then launched through `AffinityHook.exe`, which loads the patches. `Affinity.exe` itself isn't modified, so Affinity updates don't remove the hook.
3. **[LoginFix](https://github.com/seapear/AffinityOnLinux/tree/main/Guides/Wine/LoginFix)** (AffinityOnLinux). Built from source against the matching plugin loader version and copied to `apl/plugins/`.
4. **.NET WinRT libraries.** `System.Runtime.WindowsRuntime.dll` and `System.Runtime.WindowsRuntime.UI.Xaml.dll` are extracted from Microsoft's own .NET Framework 4.8 installer, checked against known SHA-256 hashes, and placed in the prefix's .NET folders. Windows only installs these on Windows 8 or later, so `winetricks dotnet48` under Wine skips them, and the Canva sign-in crashes without them. If winetricks already has the installer cached, that copy is reused.
5. **Crash reporter.** A Wine override (`crashpad_handler.exe` = disabled) stops it starting. It only uploads crash reports to Serif, and under Wine it stops Affinity from exiting.
6. **`affinity://` handler.** `~/.local/share/applications/affinity-fix-url-handler.desktop` is registered with `xdg-mime`, so the browser can hand the Canva login back to Affinity. With Flatpak Lutris, it runs Wine inside the same sandbox so the callback reaches your already-open Affinity window.
7. **Lutris.** The game's `exe:` is pointed at `AffinityHook.exe`, and the original config is saved next to it as `.affinity-fix.bak`.
8. A symlink at `~/.local/bin/affinity-fix`.

Logs, if something goes wrong:

- Plugin loader: `<prefix>/drive_c/Program Files/Affinity/Affinity/apl/logs/apl.latest.log`
- Affinity: `<prefix>/drive_c/users/<you>/AppData/Roaming/Affinity/Affinity/3.0/Log.txt`

## Known issues

- **Colour picker outside the canvas** still doesn't work. WineFix has a fix planned using the desktop's colour-picking API.
- **Wine 11.x.** A user report says the WinRT libraries alone aren't enough for sign-in on Wine 11.12. So far, sign-in has only been confirmed on ElementalWarrior Wine and Wine 10.x.
- **Crash reporting is off.** Under Wine, accepting crash reporting causes repeated crashes anyway (a known WineFix issue).
- Settings tweaks for WineFix (colour picker sampling mode and so on) are in `apl/config/winefix.toml`, or in Affinity's Preferences under WineFix.

## Credits

This is mostly glue around other people's hard work:

- [noahc3/AffinityPluginLoader](https://github.com/noahc3/AffinityPluginLoader): the plugin loader and WineFix
- [seapear/AffinityOnLinux](https://github.com/seapear/AffinityOnLinux): the guides, LoginFix and the community
- The AffinityOnLinux issue tracker ([#83](https://github.com/seapear/AffinityOnLinux/issues/83)), for finding the missing WinRT libraries and the `wine start` long-URL crash
- [ElementalWarrior](https://github.com/ElementalWarrior), for the Wine build that made Affinity usable on Linux

Affinity is a trademark of Canva / Serif. This project isn't affiliated with or endorsed by them.

## License

MIT for the scripts in this repository. The components it downloads keep their own licences.
