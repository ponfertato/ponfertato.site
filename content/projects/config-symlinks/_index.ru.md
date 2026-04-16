---
title: "Версионирование конфигураций: Git + символьные ссылки"
description: "Управление конфигами приложений через Git-репозитории и symlinks. PowerShell, BetterDiscord, Minecraft - один подход."
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

Конфигурации приложений разбросаны по системе:

- `~/.config/`, `AppData\Roaming\`, `Documents\`, `~/.minecraft/`
- При переустановке ОС или смене устройства - всё теряется
- Ручное копирование конфигов - медленно, ошибкоопасно, не версионируется

## Решение

Хранить конфигурации в Git-репозитории и подключать их к целевым путям через **символьные ссылки**.

```
~/.config/nixcfg/ (Git-репо)
    ↓ (symlink)
~/.config/nix/ (реальное расположение)

~/Git/PowerShell-Stuff/ (Git-репо)
    ↓ (mklink /D)
C:\Users\<user>\Documents\PowerShell\ (реальное расположение)
```

**Преимущества**:

- ✅ Версионирование: `git log`, `git diff`, откаты
- ✅ Бэкап: один `git push` - все конфиги в облаке
- ✅ Синхронизация: `git clone + symlinks` - новая система готова за минуты
- ✅ Тестирование: ветки для экспериментов (`feature/new-prompt`)

---

## Принцип работы

1. **Клонировать репозиторий** в удобное место (например, `~/Git/`)
2. **Удалить оригинальную папку** с конфигами (предварительно сделав бэкап)
3. **Создать символьную ссылку** из целевого пути в репозиторий
4. **Приложение работает как обычно**, но конфиги теперь в Git

---

## Пример 1: PowerShell-Stuff

**Репозиторий**: [ponfertato/PowerShell-Stuff](https://github.com/ponfertato/PowerShell-Stuff)

### Структура

```
PowerShell-Stuff/
├── Modules/          # Пользовательские модули
├── Microsoft.PowerShell_profile.ps1  # Профиль оболочки
└── powershell.config.json  # Настройки движка
```

### Установка (Windows)

```powershell
# 1. Клонировать репо
git clone https://github.com/ponfertato/PowerShell-Stuff.git ~/Git/PowerShell-Stuff

# 2. Удалить оригинальные конфиги (предварительно сделать бэкап)
Remove-Item "$env:USERPROFILE\Documents\PowerShell\Modules" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\Documents\PowerShell\powershell.config.json" -Force -ErrorAction SilentlyContinue

# 3. Создать символьные ссылки
# Для директорий: mklink /D <target> <source>
# Для файлов: mklink <target> <source>
mklink /D "$env:USERPROFILE\Documents\PowerShell\Modules" "$env:USERPROFILE\Git\PowerShell-Stuff\Modules"
mklink "$env:USERPROFILE\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "$env:USERPROFILE\Git\PowerShell-Stuff\Microsoft.PowerShell_profile.ps1"
mklink "$env:USERPROFILE\Documents\PowerShell\powershell.config.json" "$env:USERPROFILE\Git\PowerShell-Stuff\powershell.config.json"

# 4. Установить Oh My Posh (опционально)
winget install JanDeDobbeleer.OhMyPosh -s winget
```

### Результат

- Профиль загружается при старте PowerShell
- Модули доступны через `Import-Module`
- Все изменения версионируются: `git add . && git commit -m "Update prompt"`

---

## Пример 2: BetterDiscord-Stuff

**Репозиторий**: [ponfertato/BetterDiscord-Stuff](https://github.com/ponfertato/BetterDiscord-Stuff)

### Структура

```
BetterDiscord-Stuff/
├── data/             # Общие данные
├── plugins/          # Плагины: ActivityIcons, DoNotTrack, ShowAllActivities
└── themes/           # Темы: ClearVision, Fluent Discord, Material Discord
```

### Установка (Windows)

```powershell
# 1. Клонировать репо
git clone https://github.com/ponfertato/BetterDiscord-Stuff.git ~/Git/BetterDiscord-Stuff

# 2. Удалить оригинальные папки
Remove-Item "$env:APPDATA\BetterDiscord\plugins" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:APPDATA\BetterDiscord\themes" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:APPDATA\BetterDiscord\data" -Recurse -Force -ErrorAction SilentlyContinue

# 3. Создать символьные ссылки
mklink /D "$env:APPDATA\BetterDiscord\plugins" "$env:USERPROFILE\Git\BetterDiscord-Stuff\plugins"
mklink /D "$env:APPDATA\BetterDiscord\themes" "$env:USERPROFILE\Git\BetterDiscord-Stuff\themes"
mklink /D "$env:APPDATA\BetterDiscord\data" "$env:USERPROFILE\Git\BetterDiscord-Stuff\data"
```

### Установка (Linux - Native)

```bash
# 1. Клонировать репо
git clone https://github.com/ponfertato/BetterDiscord-Stuff.git ~/Git/BetterDiscord-Stuff

# 2. Удалить оригинальные папки
rm -rf ~/.config/BetterDiscord/plugins
rm -rf ~/.config/BetterDiscord/themes
rm -rf ~/.config/BetterDiscord/data

# 3. Создать символьные ссылки
ln -s ~/Git/BetterDiscord-Stuff/plugins ~/.config/BetterDiscord/plugins
ln -s ~/Git/BetterDiscord-Stuff/themes ~/.config/BetterDiscord/themes
ln -s ~/Git/BetterDiscord-Stuff/data ~/.config/BetterDiscord/data
```

### Установка (Linux - Flatpak)

```bash
# Путь для Flatpak-версии Discord
BD_PATH="$HOME/.var/app/com.discordapp.Discord/config/BetterDiscord"

# Удалить оригиналы
rm -rf "$BD_PATH/plugins" "$BD_PATH/themes" "$BD_PATH/data"

# Создать ссылки
ln -s ~/Git/BetterDiscord-Stuff/plugins "$BD_PATH/plugins"
ln -s ~/Git/BetterDiscord-Stuff/themes "$BD_PATH/themes"
ln -s ~/Git/BetterDiscord-Stuff/data "$BD_PATH/data"
```

> ⚠️ **Важно**: если репозиторий вне sandbox Flatpak, выдать разрешение на доступ к папке:
>
> ```bash
> flatpak override --user --filesystem=~/Git com.discordapp.Discord
> ```

### Ключевые плагины

| Плагин                     | Назначение                                   |
| -------------------------- | -------------------------------------------- |
| `ActivityIcons`            | Кастомные иконки статусов (Spotify, игры)    |
| `DoNotTrack`               | Блокировка телеметрии Discord                |
| `enhancecodeblocks`        | Подсветка синтаксиса, нумерация строк в коде |
| `ShowAllActivities`        | Отображение всех активностей пользователя    |
| `SplitLargeMessages`       | Авто-разбивка длинных сообщений              |
| `PluginRepo` / `ThemeRepo` | Установка плагинов/тем прямо из Discord      |

---

## Пример 3: Minecraft-Stuff

**Репозиторий**: [ponfertato/Minecraft-Stuff](https://github.com/ponfertato/Minecraft-Stuff)

### Структура

```
Minecraft-Stuff/
├── config/           # Конфиги модов (CraftPresence, Iris, Sodium)
├── mods/             # Quilt-моды для 1.20.1
├── resourcepacks/    # Ресурспаки
├── shaderpacks/      # Шейдеры (Iris-compatible)
└── texturepacks/     # Текстуры
```

### Установка (Windows)

```powershell
# 1. Клонировать репо
git clone https://github.com/ponfertato/Minecraft-Stuff.git ~/Git/Minecraft-Stuff

# 2. Удалить оригинальные папки (.minecraft)
$MC_PATH = "$env:APPDATA\.minecraft"
Remove-Item "$MC_PATH\config" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$MC_PATH\mods" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$MC_PATH\resourcepacks" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$MC_PATH\shaderpacks" -Recurse -Force -ErrorAction SilentlyContinue

# 3. Создать символьные ссылки
mklink /D "$MC_PATH\config" "$env:USERPROFILE\Git\Minecraft-Stuff\config"
mklink /D "$MC_PATH\mods" "$env:USERPROFILE\Git\Minecraft-Stuff\mods"
mklink /D "$MC_PATH\resourcepacks" "$env:USERPROFILE\Git\Minecraft-Stuff\resourcepacks"
mklink /D "$MC_PATH\shaderpacks" "$env:USERPROFILE\Git\Minecraft-Stuff\shaderpacks"
```

### Установка (Linux - Flatpak PrismLauncher)

```bash
# Путь к инстанции
MC_PATH="$HOME/.var/app/org.prismlauncher.PrismLauncher/data/PrismLauncher/instances/1.20.X/.minecraft"

# Клонировать репо
git clone https://github.com/ponfertato/Minecraft-Stuff.git ~/Git/Minecraft-Stuff

# Удалить оригиналы
rm -rf "$MC_PATH/config" "$MC_PATH/mods" "$MC_PATH/resourcepacks" "$MC_PATH/shaderpacks"

# Создать ссылки (одной командой)
ln -s ~/Git/Minecraft-Stuff/{config,mods,resourcepacks,shaderpacks} "$MC_PATH/"
```

### Ключевые моды

| Мод                          | Назначение                         |
| ---------------------------- | ---------------------------------- |
| `CraftPresence`              | Интеграция с Discord Rich Presence |
| `CustomPlayerModels`         | Кастомные 3D-модели игроков        |
| `Iris` + `Sodium`            | Шейдеры + оптимизация рендеринга   |
| `Quilted Fabric API` + `QSL` | База для модов на Quilt            |

---

## 🔄 Обновление конфигураций

```bash
# 1. Перейти в репозиторий
cd ~/Git/PowerShell-Stuff  # или BetterDiscord-Stuff / Minecraft-Stuff

# 2. Подтянуть изменения
git pull

# 3. (Опционально) Закоммитить свои правки
git add .
git commit -m "Update prompt colors"
git push
```

**Для Discord/Minecraft**: изменения применяются сразу или после перезапуска приложения.

---

## ⚠️ Нюансы и решения

| Проблема                                 | Решение                                                                         |
| ---------------------------------------- | ------------------------------------------------------------------------------- |
| **Ссылка не создаётся (Windows)**        | Запустить терминал от имени администратора или включить "Режим разработчика"    |
| **Flatpak не видит файлы вне sandbox**   | Выдать разрешение: `flatpak override --user --filesystem=~/Git <app-id>`        |
| **Конфликт при обновлении мода/плагина** | Использовать `git stash` перед `pull`, затем `git stash pop`                    |
| **Бэкап перед удалением оригиналов**     | `mv ~/.config/app ~/.config/app.bak` или `git add . && git commit`              |
| **Проверка, что ссылка работает**        | `ls -l ~/.config/app` (Linux) или `dir /AL "%APPDATA%\BetterDiscord"` (Windows) |

---

## Ссылки

- 🐙 [PowerShell-Stuff](https://github.com/ponfertato/PowerShell-Stuff)
- 🎨 [BetterDiscord-Stuff](https://github.com/ponfertato/BetterDiscord-Stuff)
- ⛏️ [Minecraft-Stuff](https://github.com/ponfertato/Minecraft-Stuff)
- 📘 [mklink (Windows)](https://learn.microsoft.com/windows-server/administration/windows-commands/mklink)
- 🔗 [ln (Linux)](https://man7.org/linux/man-pages/man1/ln.1.html)
- 🧰 [Oh My Posh](https://ohmyposh.dev)
