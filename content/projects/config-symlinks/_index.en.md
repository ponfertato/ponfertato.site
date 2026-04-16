---
title: "Config Versioning: Git + Symbolic Links"
description: "Managing app configs via Git repositories and symlinks. PowerShell, BetterDiscord, Minecraft - one approach."
tags:
  [
    "git",
    "symlinks",
    "dotfiles",
    "powershell",
    "discord",
    "minecraft",
    "versioning",
  ]
---

App configurations are scattered across the system:

- `~/.config/`, `AppData\Roaming\`, `Documents\`, `~/.minecraft/`
- Lost on OS reinstall or device change
- Manual copying is slow, error-prone, unversioned

## The Solution

Store configurations in a Git repository and connect them to target paths via **symbolic links**.

```
~/.config/nixcfg/ (Git repo)
    ↓ (symlink)
~/.config/nix/ (actual location)

~/Git/PowerShell-Stuff/ (Git repo)
    ↓ (mklink /D)
C:\Users\<user>\Documents\PowerShell\ (actual location)
```

**Benefits**:

- ✅ Versioning: `git log`, `git diff`, rollbacks
- ✅ Backup: one `git push` - all configs in the cloud
- ✅ Sync: `git clone + symlinks` - new system ready in minutes
- ✅ Testing: branches for experiments (`feature/new-prompt`)

---

## How It Works

1. **Clone the repository** to a convenient location (e.g., `~/Git/`)
2. **Remove the original config folder** (after backing up)
3. **Create a symbolic link** from the target path to the repo
4. **App works as usual**, but configs are now in Git

---

## Example 1: PowerShell-Stuff

**Repository**: [ponfertato/PowerShell-Stuff](https://github.com/ponfertato/PowerShell-Stuff)

### Structure

```
PowerShell-Stuff/
├── Modules/          # Custom modules
├── Microsoft.PowerShell_profile.ps1  # Shell profile
└── powershell.config.json  # Engine settings
```

### Installation (Windows)

```powershell
# 1. Clone repo
git clone https://github.com/ponfertato/PowerShell-Stuff.git ~/Git/PowerShell-Stuff

# 2. Remove original configs (backup first)
Remove-Item "$env:USERPROFILE\Documents\PowerShell\Modules" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\Documents\PowerShell\powershell.config.json" -Force -ErrorAction SilentlyContinue

# 3. Create symbolic links
# For directories: mklink /D <target> <source>
# For files: mklink <target> <source>
mklink /D "$env:USERPROFILE\Documents\PowerShell\Modules" "$env:USERPROFILE\Git\PowerShell-Stuff\Modules"
mklink "$env:USERPROFILE\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "$env:USERPROFILE\Git\PowerShell-Stuff\Microsoft.PowerShell_profile.ps1"
mklink "$env:USERPROFILE\Documents\PowerShell\powershell.config.json" "$env:USERPROFILE\Git\PowerShell-Stuff\powershell.config.json"

# 4. Install Oh My Posh (optional)
winget install JanDeDobbeleer.OhMyPosh -s winget
```

### Result

- Profile loads on PowerShell start
- Modules available via `Import-Module`
- All changes versioned: `git add . && git commit -m "Update prompt"`

---

## Example 2: BetterDiscord-Stuff

**Repository**: [ponfertato/BetterDiscord-Stuff](https://github.com/ponfertato/BetterDiscord-Stuff)

### Structure

```
BetterDiscord-Stuff/
├── data/             # Shared data
├── plugins/          # Plugins: ActivityIcons, DoNotTrack, ShowAllActivities
└── themes/           # Themes: ClearVision, Fluent Discord, Material Discord
```

### Installation (Windows)

```powershell
# 1. Clone repo
git clone https://github.com/ponfertato/BetterDiscord-Stuff.git ~/Git/BetterDiscord-Stuff

# 2. Remove original folders
Remove-Item "$env:APPDATA\BetterDiscord\plugins" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:APPDATA\BetterDiscord\themes" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:APPDATA\BetterDiscord\data" -Recurse -Force -ErrorAction SilentlyContinue

# 3. Create symbolic links
mklink /D "$env:APPDATA\BetterDiscord\plugins" "$env:USERPROFILE\Git\BetterDiscord-Stuff\plugins"
mklink /D "$env:APPDATA\BetterDiscord\themes" "$env:USERPROFILE\Git\BetterDiscord-Stuff\themes"
mklink /D "$env:APPDATA\BetterDiscord\data" "$env:USERPROFILE\Git\BetterDiscord-Stuff\data"
```

### Installation (Linux - Native)

```bash
# 1. Clone repo
git clone https://github.com/ponfertato/BetterDiscord-Stuff.git ~/Git/BetterDiscord-Stuff

# 2. Remove original folders
rm -rf ~/.config/BetterDiscord/plugins
rm -rf ~/.config/BetterDiscord/themes
rm -rf ~/.config/BetterDiscord/data

# 3. Create symbolic links
ln -s ~/Git/BetterDiscord-Stuff/plugins ~/.config/BetterDiscord/plugins
ln -s ~/Git/BetterDiscord-Stuff/themes ~/.config/BetterDiscord/themes
ln -s ~/Git/BetterDiscord-Stuff/data ~/.config/BetterDiscord/data
```

### Installation (Linux - Flatpak)

```bash
# Path for Flatpak Discord
BD_PATH="$HOME/.var/app/com.discordapp.Discord/config/BetterDiscord"

# Remove originals
rm -rf "$BD_PATH/plugins" "$BD_PATH/themes" "$BD_PATH/data"

# Create links
ln -s ~/Git/BetterDiscord-Stuff/plugins "$BD_PATH/plugins"
ln -s ~/Git/BetterDiscord-Stuff/themes "$BD_PATH/themes"
ln -s ~/Git/BetterDiscord-Stuff/data "$BD_PATH/data"
```

> ⚠️ **Note**: if the repo is outside Flatpak sandbox, grant filesystem access:
>
> ```bash
> flatpak override --user --filesystem=~/Git com.discordapp.Discord
> ```

### Key Plugins

| Plugin                     | Purpose                                          |
| -------------------------- | ------------------------------------------------ |
| `ActivityIcons`            | Custom status icons (Spotify, games)             |
| `DoNotTrack`               | Block Discord telemetry                          |
| `enhancecodeblocks`        | Syntax highlighting, line numbers in code blocks |
| `ShowAllActivities`        | Display all user activities                      |
| `SplitLargeMessages`       | Auto-split long messages                         |
| `PluginRepo` / `ThemeRepo` | Install plugins/themes directly from Discord     |

---

## Example 3: Minecraft-Stuff

**Repository**: [ponfertato/Minecraft-Stuff](https://github.com/ponfertato/Minecraft-Stuff)

### Structure

```
Minecraft-Stuff/
├── config/           # Mod configs (CraftPresence, Iris, Sodium)
├── mods/             # Quilt mods for 1.20.1
├── resourcepacks/    # Resource packs
├── shaderpacks/      # Shaders (Iris-compatible)
└── texturepacks/     # Texture packs
```

### Installation (Windows)

```powershell
# 1. Clone repo
git clone https://github.com/ponfertato/Minecraft-Stuff.git ~/Git/Minecraft-Stuff

# 2. Remove original folders (.minecraft)
$MC_PATH = "$env:APPDATA\.minecraft"
Remove-Item "$MC_PATH\config" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$MC_PATH\mods" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$MC_PATH\resourcepacks" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$MC_PATH\shaderpacks" -Recurse -Force -ErrorAction SilentlyContinue

# 3. Create symbolic links
mklink /D "$MC_PATH\config" "$env:USERPROFILE\Git\Minecraft-Stuff\config"
mklink /D "$MC_PATH\mods" "$env:USERPROFILE\Git\Minecraft-Stuff\mods"
mklink /D "$MC_PATH\resourcepacks" "$env:USERPROFILE\Git\Minecraft-Stuff\resourcepacks"
mklink /D "$MC_PATH\shaderpacks" "$env:USERPROFILE\Git\Minecraft-Stuff\shaderpacks"
```

### Installation (Linux - Flatpak PrismLauncher)

```bash
# Instance path
MC_PATH="$HOME/.var/app/org.prismlauncher.PrismLauncher/data/PrismLauncher/instances/1.20.X/.minecraft"

# Clone repo
git clone https://github.com/ponfertato/Minecraft-Stuff.git ~/Git/Minecraft-Stuff

# Remove originals
rm -rf "$MC_PATH/config" "$MC_PATH/mods" "$MC_PATH/resourcepacks" "$MC_PATH/shaderpacks"

# Create links (one-liner)
ln -s ~/Git/Minecraft-Stuff/{config,mods,resourcepacks,shaderpacks} "$MC_PATH/"
```

### Key Mods

| Mod                          | Purpose                           |
| ---------------------------- | --------------------------------- |
| `CraftPresence`              | Discord Rich Presence integration |
| `CustomPlayerModels`         | Custom 3D player models           |
| `Iris` + `Sodium`            | Shaders + rendering optimization  |
| `Quilted Fabric API` + `QSL` | Base for Quilt mods               |

---

## 🔄 Updating Configurations

```bash
# 1. Navigate to repository
cd ~/Git/PowerShell-Stuff  # or BetterDiscord-Stuff / Minecraft-Stuff

# 2. Pull changes
git pull

# 3. (Optional) Commit your changes
git add .
git commit -m "Update prompt colors"
git push
```

**For Discord/Minecraft**: changes apply immediately or after app restart.

---

## ⚠️ Nuances and Solutions

| Issue                                       | Solution                                                                       |
| ------------------------------------------- | ------------------------------------------------------------------------------ |
| **Link creation fails (Windows)**           | Run terminal as Administrator or enable "Developer Mode"                       |
| **Flatpak can't see files outside sandbox** | Grant permission: `flatpak override --user --filesystem=~/Git <app-id>`        |
| **Conflict when updating mod/plugin**       | Use `git stash` before `pull`, then `git stash pop`                            |
| **Backup before removing originals**        | `mv ~/.config/app ~/.config/app.bak` or `git add . && git commit`              |
| **Verify link works**                       | `ls -l ~/.config/app` (Linux) or `dir /AL "%APPDATA%\BetterDiscord"` (Windows) |

---

## Links

- 🐙 [PowerShell-Stuff](https://github.com/ponfertato/PowerShell-Stuff)
- 🎨 [BetterDiscord-Stuff](https://github.com/ponfertato/BetterDiscord-Stuff)
- ⛏️ [Minecraft-Stuff](https://github.com/ponfertato/Minecraft-Stuff)
- 📘 [mklink (Windows)](https://learn.microsoft.com/windows-server/administration/windows-commands/mklink)
- 🔗 [ln (Linux)](https://man7.org/linux/man-pages/man1/ln.1.html)
- 🧰 [Oh My Posh](https://ohmyposh.dev)
