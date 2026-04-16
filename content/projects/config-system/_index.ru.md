---
title: "Управление конфигурацией: Nix и dotfiles"
description: "Декларативная конфигурация через NixOS Flakes и традиционные dotfiles. Модульность, воспроизводимость, кроссплатформенность."
tags:
  ["nix", "nixos", "dotfiles", "home-manager", "reproducibility", "automation"]
---

Два подхода к управлению конфигурацией, используемые параллельно:

| Метод             | Инструмент                  | Область применения                       | Преимущества                                           |
| ----------------- | --------------------------- | ---------------------------------------- | ------------------------------------------------------ |
| **Декларативный** | NixOS Flakes + Home Manager | Система, пакеты, сервисы                 | Воспроизводимость, атомарные откаты, тестирование      |
| **Традиционный**  | chezmoi / git + symlinks    | Пользовательские конфиги, кроссплатформа | Гибкость, поддержка не-Nix систем, минимальный оверхед |

Оба метода версионируются в Git, деплоятся через `pull + apply`, поддерживают мультихост-конфигурации.

---

## 🧊 Nix: Декларативная конфигурация

### Архитектура `nixcfg`

```
nixcfg/
├── flake.nix              # Точка входа: определения систем, входные данные
├── profiles/
│   ├── base/              # Базовые настройки: timezone, locale, networking
│   ├── common/            # Общие для всех хостов: users, security, nix settings
│   ├── desktop/           # Графическое окружение: Wayland, portals, fonts
│   ├── laptop/            # Специфика ноутбуков: блютуз, fingerprint, power
│   ├── mobile/            # Nix-on-Droid: пакеты для Android
│   └── workstation/       # Серверные настройки: Wake-on-LAN, sysctl
├── hosts/
│   ├── potatoWork/        # Конфигурация хоста: hostname, kernel params, микрокод
│   ├── potatoLaptop/
│   ├── potatoPhone/       # Nix-on-Droid хост
│   └── potatoTablet/
├── modules/
│   ├── docker.nix         # Модуль: включение Docker, настройки daemon
│   ├── flatpak.nix        # Flatpak: репозитории, пакеты
│   ├── ollama.nix         # Локальные LLM: сервис, модели, API
│   ├── waydroid.nix       # Android-контейнеры в Linux
│   └── home-manager/      # Пользовательская среда
│       ├── common.nix     # Bash, Git, Firefox, алиасы
│       ├── packages.nix   # Пользовательские пакеты
│       ├── potatoWork.nix # Хост-специфичные пакеты
│       └── potatoLaptop.nix
└── scripts/               # Утилиты: обновление, сборка, откат
```

### Ключевые принципы

1. **Flakes-первый подход**:

```nix
# flake.nix
outputs = { self, nixpkgs, home-manager, ... }@inputs: {
  nixosConfigurations.potatoWork = nixpkgs.lib.nixosSystem {
    system = "x86_64-linux";
    modules = [
      ./profiles/base
      ./profiles/common
      ./profiles/desktop
      ./hosts/potatoWork/default.nix
      ./modules/docker.nix
      home-manager.nixosModules.home-manager
    ];
  };
};
```

2. **Модульность через `imports`**:

```nix
# profiles/base/default.nix
{ config, pkgs, lib, ... }: {
  time.timeZone = "Europe/Moscow";
  i18n.defaultLocale = "ru_RU.UTF-8";
  networking.networkmanager.enable = true;
  security.polkit.enable = true;
}
```

3. **Хост-специфичные настройки**:

```nix
# hosts/potatoWork/default.nix
{ config, pkgs, ... }: {
  networking.hostName = "potatoWork";
  boot.kernelParams = [
    "i915.enable_fbc=1"
    "i915.enable_guc=2"
    "intel_pstate=active"
  ];
  hardware.cpu.intel.updateMicrocode = true;
}
```

4. **Home Manager для пользовательской среды**:

```nix
# modules/home-manager/common.nix
{ config, pkgs, ... }: {
  programs.git = {
    enable = true;
    settings.user = {
      name = "ponfertato";
      email = "ponfertato@ya.ru";
    };
    ignores = [ "*~" ".DS_Store" ".idea/" ];
  };
  programs.firefox = {
    enable = true;
    profiles.default = {
      isDefault = true;
      search.default = "ddg";
    };
  };
}
```

### Обслуживание и обновления

```bash
# Полное обновление системы (рекомендуется)
nix flake update && sudo nixos-rebuild switch --flake .#$(hostname) --impure

# Откат к предыдущей конфигурации
sudo nixos-rebuild switch --rollback

# Сборка без активации (тест)
sudo nixos-rebuild build --flake .#$(hostname) --impure

# Проверка конфигурации
nix flake check

# Очистка старых поколений
nix-collect-garbage -d
```

**Алиасы для удобства** (`modules/home-manager/common.nix`):

```nix
home.shellAliases = {
  nix-apply = "sudo nixos-rebuild switch --flake .#$(hostname) --impure";
  nix-apply-user = "home-manager switch --flake .#$(hostname)";
  nix-build = "sudo nixos-rebuild build --flake .#$(hostname) --impure";
  nix-roll = "sudo nixos-rebuild switch --rollback";
  nix-update = "nix flake update";
};
```

### Поддержка мобильных устройств (Nix-on-Droid)

```nix
# flake.nix
nixOnDroidConfigurations = {
  potatoPhone = mkNixOnDroidConfig "potatoPhone" [];
  potatoTablet = mkNixOnDroidConfig "potatoTablet" [];
};

# hosts/potatoPhone/default.nix
{ config, pkgs, ... }: {
  environment.packages = with pkgs; [ termux-api curl git lazygit nano openssh wget ];
  time.timeZone = "Europe/Moscow";
  i18n.defaultLocale = "ru_RU.UTF-8";
}
```

**Сборка для Android**:

```bash
nix-on-droid switch --flake ~/.config/nixpkgs#potatoPhone
```

---

## 📁 Dotfiles: Традиционный подход

### Архитектура `dotfiles`

```
dotfiles/
├── install                 # Скрипт установки: symlink, bootstrap
├── shell/
│   ├── bash/
│   │   ├── bashrc          # Базовые настройки: PATH, PS1, aliases
│   │   └── functions/      # Пользовательские функции
│   ├── zsh/
│   │   ├── zshrc           # Oh-my-zsh конфиг, плагины
│   │   ├── aliases.zsh     # Алиасы
│   │   └── functions.zsh   # Функции
│   └── powershell/
│       ├── settings.json   # Terminal settings
│       ├── profile.ps1     # PowerShell профиль
│       └── modules/        # Кастомные модули
├── git/
│   └── gitconfig           # Глобальный Git конфиг
├── ssh/
│   └── config              # SSH хосты, ключи, ForwardAgent
├── nvim/                   # Neovim конфиг (или ссылка на внешний репо)
├── wsl/
│   └── .wslconfig          # Настройки WSL: память, ядро
├── browser/
│   └── firefox/user.js     # Firefox about:config настройки
└── wakatime/
    └── .wakatime.cfg       # Трекинг времени разработки
```

### Ключевые особенности

1. **Кроссплатформенность**:

```bash
# install - кроссплатформенный скрипт
#!/usr/bin/env bash
OS=$(uname -s)
case "$OS" in
  Linux)
    ln -sf "$PWD/shell/bash/bashrc" ~/.bashrc
    ln -sf "$PWD/git/gitconfig" ~/.gitconfig
    ;;
  Darwin)
    # macOS специфика
    ;;
  MINGW*|CYGWIN*)
    # Windows (Git Bash / WSL)
    ;;
esac
```

2. **Условная загрузка конфигов**:

```bash
# ~/.bashrc
if [ -f ~/.bash_aliases ]; then
  source ~/.bash_aliases
fi
if [ -d ~/.bash_functions ]; then
  for f in ~/.bash_functions/*.sh; do source "$f"; done
fi
```

3. **SSH конфигурация с шаблонами**:

```ssh-config
# ~/.ssh/config
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  AddKeysToAgent yes
  ForwardAgent no

Host *.potatoenergy.ru
  User ponfertato
  IdentityFile ~/.ssh/id_ed25519_potato
  ForwardAgent yes
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```

4. **Firefox пользовательские настройки**:

```javascript
// user.js
user_pref("privacy.resistFingerprinting", true);
user_pref("privacy.tracking_protection.enabled", true);
user_pref("network.dns.disablePrefetch", true);
user_pref("browser.startup.page", 3); // restore session
```

### Интеграция с Nix

Dotfiles могут использоваться **внутри** NixOS-конфигурации:

```nix
# modules/home-manager/common.nix
home.file.".bashrc".source = ../../../dotfiles/shell/bash/bashrc;
home.file.".gitconfig".source = ../../../dotfiles/git/gitconfig;
home.file.".ssh/config".source = ../../../dotfiles/ssh/config;

# Или через генерацию:
programs.bash.bashrcExtra = builtins.readFile ../../../dotfiles/shell/bash/bashrc;
```

**Преимущества**:

- ✅ Единый источник истины для конфигов
- ✅ Версионирование и ревью изменений через Git
- ✅ Атомарное применение: `home-manager switch` или `nixos-rebuild`

---

## 🔀 Когда что использовать

| Задача                                     | Рекомендуемый подход        | Почему                                        |
| ------------------------------------------ | --------------------------- | --------------------------------------------- |
| Конфигурация ОС, сервисов, пакетов         | NixOS Flakes                | Воспроизводимость, тестирование, откаты       |
| Пользовательские алиасы, темы, плагины     | Home Manager (в Nix)        | Интеграция с системой, версионирование        |
| Конфиги для не-Nix систем (Windows, macOS) | Dotfiles + chezmoi          | Кроссплатформенность, минимальные зависимости |
| Быстрое прототипирование конфигов          | Прямое редактирование + Git | Скорость, гибкость                            |
| Командная стандартизация конфигов          | Dotfiles-репо + CI-проверки | Единый стиль, автоматическая валидация        |

### Гибридный подход (рекомендуется)

1. **Системный слой** (NixOS):
   - Пакеты, сервисы, пользователи, сеть, безопасность
   - Управление через `flake.nix` + `nixos-rebuild`

2. **Пользовательский слой** (Home Manager):
   - Оболочка, редактор, браузер, ключи
   - Управление через `home-manager switch`

3. **Кроссплатформенный слой** (Dotfiles):
   - SSH, Git, общие алиасы
   - Синхронизация через Git, установка через `install`-скрипт

---

## ⚙️ Автоматизация и тестирование

### Проверка конфигурации перед применением

```bash
# Nix: проверка синтаксиса и зависимостей
nix flake check

# Home Manager: dry-run
home-manager switch --flake .#$(hostname) --dry-run

# Dotfiles: валидация конфигов
bash -n ~/.bashrc          # Синтаксис bash
zsh -n ~/.zshrc           # Синтаксис zsh
git config --global --get user.name  # Проверка загрузки
```

### Тестирование в изолированной среде

```bash
# Nix: сборка в chroot (если включено)
sudo nixos-rebuild build --flake .#test-host --impure --option build-users-group ""

# Dotfiles: тестовая установка в temp-директорию
export HOME=$(mktemp -d)
./install --dry-run
```

### CI/CD для конфигов (опционально)

```yaml
# .github/workflows/check.yml
name: Validate Configs
on: [push, pull_request]
jobs:
  nix:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cachix/install-nix-action@v25
      - run: nix flake check
  dotfiles:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: bash -n shell/bash/bashrc
      - run: zsh -n shell/zsh/zshrc
```

---

## ⚠️ Частые проблемы

```bash
# Nix: "flake input 'nixpkgs' has changed"
→ Обновить лок: `nix flake lock --update-input nixpkgs`

# Home Manager: конфликт файлов
→ Удалить существующий файл или использовать `home.file.<name>.force = true`

# Dotfiles: symlink уже существует
→ Использовать `install --force` или предварительно сделать бэкап

# Кроссплатформенные алиасы не работают
→ Проверить, что файл загружается: `echo $BASH_VERSION`, `echo $ZSH_VERSION`

# SSH ключи не подхватываются
→ Проверить права: `chmod 600 ~/.ssh/id_*`, `chmod 644 ~/.ssh/config`
```

---

## Ссылки

- ❄️ [NixOS Manual](https://nixos.org/manual/nixos/stable/)
- 🏠 [Home Manager Docs](https://nix-community.github.io/home-manager/)
- 🧩 [Flakes Reference](https://nixos.wiki/wiki/Flakes)
- 📦 [Chezmoi Documentation](https://www.chezmoi.io/)
- 🐙 [Nix-on-Droid](https://github.com/nix-community/nix-on-droid)
