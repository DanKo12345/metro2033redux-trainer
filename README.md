# Metro 2033 Redux Trainer

A trainer for the Steam version of Metro 2033 Redux (1.0.0.4, Steam build 26162904), with a per-user installer.

## How it works

The trainer finds `metro.exe` and locates the game code it needs by byte signatures, so it never relies on fixed addresses. It changes the game only in memory, never on disk, and closing the trainer undoes every change. It does not connect to the internet.

| Kind of change | Used for | Where in the source |
|---|---|---|
| Engine debug flags | Infinite ammo, invisibility | `Catalog`, `Game.Sync` in `src/Trainer.cs` |
| Engine setting | Fast movement (`g_player_speed_scale`) | `Game.Sync` |
| Small code patches, restored on exit | Medkits, filters, gas mask, reload, air pressure, Volt Driver, freeze, money | `Catalog.Build` (signature, offset, bytes) |

Military-grade ammo is also the game's money. The engine's ammo flag on its own would make the shop free too, so the ammo feature also masks the flag out of the two money functions, and money is a separate feature that patches those same functions.
| Code cave next to the game module | Noclip, position save and teleport, god mode for the player only | `Game.BuildMoveCave`, `Game.BuildGodCave` |

To keep the game from running half-written code, the process is paused for a few microseconds while code is patched (`NtSuspendProcess`).

The Windows APIs it uses on the game process are `OpenProcess`, `ReadProcessMemory`, `WriteProcessMemory`, `VirtualProtectEx`, `VirtualAllocEx` and `FlushInstructionCache`. Hotkeys are read with `GetAsyncKeyState` and only count while the game or the trainer window is in front.

The installer (`src/Setup.cs`) embeds the trainer, copies it to a folder the user picks (by default `%LOCALAPPDATA%\Programs\Metro 2033 Redux Trainer`), creates shortcuts and registers an uninstall entry under `HKCU`. No administrator rights are needed. To uninstall, `Uninstall.exe` copies itself to `%TEMP%` so it can remove the install folder, then deletes that copy.

Settings (language, sound, always on top, hotkeys) are stored in `HKCU\Software\Metro2033ReduxTrainer`.

## Building

Requirements: Windows 10 or 11 with .NET Framework 4.x, which is part of Windows. No other tools are needed: the build uses the C# compiler that ships with .NET Framework (`csc.exe`).

```
powershell -ExecutionPolicy Bypass -File build.ps1
```

Output:

- `dist\Metro2033ReduxTrainer.exe`: the trainer on its own
- `dist\Metro2033ReduxTrainerSetup.exe`: the installer, with the trainer embedded as a resource

The build also creates `assets\icon.ico` if it is missing.

## Source layout

| File | Contents |
|---|---|
| `src/Trainer.cs` | Game memory access, features, main window |
| `src/Setup.cs` | Installer and uninstaller |
| `src/Ui.cs` | Theme and controls shared by both programs |
| `src/Localization.cs` | English, Russian, Ukrainian and Chinese strings, saved settings |
| `build.ps1` | Build script |

## Diagnostics

The trainer accepts a few switches for testing against a running game. Each writes a log to the given file:

- `--selftest <log>`: turns every feature on and off and checks that the game memory changed and was restored
- `--abandon <log>`: turns everything on and exits without cleaning up, like a crash
- `--recover <log>`: attaches to a game left in that state and restores it
- `--watch <log> <seconds>`: logs whether the game is found

For single-player use only.
