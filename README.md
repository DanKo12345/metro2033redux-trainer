# Metro 2033 Redux Trainer and Metro Last Light Redux Trainer

Two trainers for the Steam versions of Metro 2033 Redux and Metro Last Light Redux, each with a per-user installer. They are built from the same source: since the November 2026 update both games run the same `metro.exe`, so the game code, the signatures and the features are the same. Each build only attaches to its own game. Both share one design and differ only in colour: amber on a green tunnel light for 2033, the orange-red of the Last Light menu on a rusty glow for Last Light.

## How it works

The trainer finds `metro.exe`, checks from the folder it runs in that it is the right game (the Steam app manifest, or a file only Last Light ships), and locates the game code it needs by byte signatures, so it never relies on fixed addresses. It changes the game only in memory, never on disk, and closing the trainer undoes every change. It does not connect to the internet.

| Kind of change | Used for | Where in the source |
|---|---|---|
| Engine debug flags | Infinite ammo, invisibility | `Catalog`, `Game.Sync` in `src/Trainer.cs` |
| Engine setting | Fast movement (`g_player_speed_scale`) | `Game.Sync` |
| Small code patches, restored on exit | Medkits, filters, gas mask, reload, recoil, air pressure, Volt Driver, freeze, money | `Catalog.Build` (signature, offset, bytes) |
| Code cave next to the game module | Noclip, position save and teleport; god mode, no reload, no spread and rapid fire for the player only | `Game.BuildMoveCave`, `Game.BuildGodCave`, `Game.BuildHookCave` |

Military-grade ammo is also the game's money. The engine's ammo flag on its own would make the shop free too, so the ammo feature also masks the flag out of the two money functions, and money is a separate feature that patches those same functions.

Every firearm fires through one shared shot routine. No reload sends the step that takes rounds out of the magazine to a cave that skips it when the weapon belongs to the player. No recoil or spread does the same with the weapon's spread, so the player's bullets go where the crosshair points, and also starts the camera kick of each shot from zero. Enemies' weapons are left as they are.

Weapons fire while their cooldown is not above zero, and every shot adds the weapon's shot interval to it. Rapid fire halves that interval for the player's weapon, so automatic weapons fire twice as fast and the others are ready for the next shot sooner.

To keep the game from running half-written code, the process is paused for a few microseconds while code is patched (`NtSuspendProcess`).

The Windows APIs it uses on the game process are `OpenProcess`, `ReadProcessMemory`, `WriteProcessMemory`, `VirtualProtectEx`, `VirtualAllocEx` and `FlushInstructionCache`. Hotkeys are read with `GetAsyncKeyState` and only count while the game or the trainer window is in front.

The installer (`src/Setup.cs`) embeds the trainer, copies it to a folder the user picks (by default `%LOCALAPPDATA%\Programs\Metro 2033 Redux Trainer` or `...\Metro Last Light Redux Trainer`), creates shortcuts and registers an uninstall entry under `HKCU`. No administrator rights are needed. To uninstall, `Uninstall.exe` copies itself to `%TEMP%` so it can remove the install folder, then deletes that copy.

Settings (language, sound, always on top, hotkeys) are stored in `HKCU\Software\Metro2033ReduxTrainer` or `HKCU\Software\MetroLastLightReduxTrainer`.

## Building

Requirements: Windows 10 or 11 with .NET Framework 4.x, which is part of Windows. No other tools are needed: the build uses the C# compiler that ships with .NET Framework (`csc.exe`).

```
powershell -ExecutionPolicy Bypass -File build.ps1
```

Output:

- `dist\Metro2033ReduxTrainer.exe` and `dist\MetroLastLightReduxTrainer.exe`: the trainers on their own
- `dist\Metro2033ReduxTrainerSetup.exe` and `dist\MetroLastLightReduxTrainerSetup.exe`: the installers, with the trainer embedded as a resource

The Last Light programs are compiled with `/define:LASTLIGHT`. The build also creates `assets\icon.ico` and `assets\icon_lastlight.ico` if they are missing.

## Source layout

| File | Contents |
|---|---|
| `src/Product.cs` | Which game a build is for: names, Steam app ids, telling the two games apart |
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
