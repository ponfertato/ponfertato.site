---
title: "System Configuration: Nix & Dotfiles"
description: "Declarative configuration via NixOS Flakes and traditional dotfiles. Modularity, reproducibility, cross-platform support."
tags:
  ["nix", "nixos", "dotfiles", "home-manager", "reproducibility", "automation"]
---

Two configuration management approaches used in parallel:

| Method          | Tool                        | Scope                        | Advantages                                     |
| --------------- | --------------------------- | ---------------------------- | ---------------------------------------------- |
| **Declarative** | NixOS Flakes + Home Manager | System, packages, services   | Reproducibility, atomic rollbacks, testing     |
| **Traditional** | chezmoi / git + symlinks    | User configs, cross-platform | Flexibility, non-Nix support, minimal overhead |

Both methods are versioned in Git, deployed via `pull + apply`, and support multi-host configurations.

---

## 🧊 Nix: Declarative Configuration

### `nixcfg` Architecture

```
nixcfg/
├── flake.nix              # Entry point: system definitions, inputs
├── profiles/
│   ├── base/              # Base settings: timezone, locale, networking
│   ├── common/            # Shared across hosts: users, security, nix settings
│   ├── desktop/           # Graphical environment: Wayland, portals, fonts
│   ├── laptop/            # Laptop-specific: bluetooth, fingerprint, power
│   ├── mobile/            # Nix-on-Droid: Android packages
│   └── workstation/       # Server settings: Wake-on-LAN, sysctl
├── hosts/
│   ├── potatoWork/        # Host config: hostname, kernel params, microcode
│   ├── potatoLaptop/
│   ├── potatoPhone/       # Nix-on-Droid host
│   └── potatoTablet/
├── modules/
│   ├── docker.nix         # Module: enable Docker, daemon settings
│   ├── flatpak.nix        # Flatpak: repos, packages
│   ├── ollama.nix         # Local LLMs: service, models, API
│   ├── waydroid.nix       # Android containers in Linux
│   └── home-manager/      # User environment
│       ├── common.nix     # Bash, Git, Firefox, aliases
│       ├── packages.nix   # User packages
│       ├── potatoWork.nix # Host-specific packages
│       └── potatoLaptop.nix
└── scripts/               # Utilities: update, build, rollback
```

### Core Principles

1. **Flakes-first approach**:

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

2. **Modularity via `imports`**:

```nix
# profiles/base/default.nix
{ config, pkgs, lib, ... }: {
  time.timeZone = "Europe/Moscow";
  i18n.defaultLocale = "ru_RU.UTF-8";
  networking.networkmanager.enable = true;
  security.polkit.enable = true;
}
```

3. **Host-specific settings**:

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

4. **Home Manager for user environment**:

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

### Maintenance & Updates

```bash
# Full system update (recommended)
nix flake update && sudo nixos-rebuild switch --flake .#$(hostname) --impure

# Rollback to previous configuration
sudo nixos-rebuild switch --rollback

# Build without activation (test)
sudo nixos-rebuild build --flake .#$(hostname) --impure

# Validate configuration
nix flake check

# Clean old generations
nix-collect-garbage -d
```

**Convenience aliases** (`modules/home-manager/common.nix`):

```nix
home.shellAliases = {
  nix-apply = "sudo nixos-rebuild switch --flake .#$(hostname) --impure";
  nix-apply-user = "home-manager switch --flake .#$(hostname)";
  nix-build = "sudo nixos-rebuild build --flake .#$(hostname) --impure";
  nix-roll = "sudo nixos-rebuild switch --rollback";
  nix-update = "nix flake update";
};
```

### Mobile Support (Nix-on-Droid)

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

**Build for Android**:

```bash
nix-on-droid switch --flake ~/.config/nixpkgs#potatoPhone
```

---

## 📁 Dotfiles: Traditional Approach

### `dotfiles` Architecture

```
dotfiles/
├── install                 # Installation script: symlink, bootstrap
├── shell/
│   ├── bash/
│   │   ├── bashrc          # Base settings: PATH, PS1, aliases
│   │   └── functions/      # Custom functions
│   ├── zsh/
│   │   ├── zshrc           # Oh-my-zsh config, plugins
│   │   ├── aliases.zsh     # Aliases
│   │   └── functions.zsh   # Functions
│   └── powershell/
│       ├── settings.json   # Terminal settings
│       ├── profile.ps1     # PowerShell profile
│       └── modules/        # Custom modules
├── git/
│   └── gitconfig           # Global Git config
├── ssh/
│   └── config              # SSH hosts, keys, ForwardAgent
├── nvim/                   # Neovim config (or external repo link)
├── wsl/
│   └── .wslconfig          # WSL settings: memory, kernel
├── browser/
│   └── firefox/user.js     # Firefox about:config settings
└── wakatime/
    └── .wakatime.cfg       # Development time tracking
```

### Key Features

1. **Cross-platform support**:

```bash
# install - cross-platform script
#!/usr/bin/env bash
OS=$(uname -s)
case "$OS" in
  Linux)
    ln -sf "$PWD/shell/bash/bashrc" ~/.bashrc
    ln -sf "$PWD/git/gitconfig" ~/.gitconfig
    ;;
  Darwin)
    # macOS specific
    ;;
  MINGW*|CYGWIN*)
    # Windows (Git Bash / WSL)
    ;;
esac
```

2. **Conditional config loading**:

```bash
# ~/.bashrc
if [ -f ~/.bash_aliases ]; then
  source ~/.bash_aliases
fi
if [ -d ~/.bash_functions ]; then
  for f in ~/.bash_functions/*.sh; do source "$f"; done
fi
```

3. **SSH configuration with templates**:

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

4. **Firefox user preferences**:

```javascript
// user.js
user_pref("privacy.resistFingerprinting", true);
user_pref("privacy.tracking_protection.enabled", true);
user_pref("network.dns.disablePrefetch", true);
user_pref("browser.startup.page", 3); // restore session
```

### Integration with Nix

Dotfiles can be used **inside** NixOS configuration:

```nix
# modules/home-manager/common.nix
home.file.".bashrc".source = ../../../dotfiles/shell/bash/bashrc;
home.file.".gitconfig".source = ../../../dotfiles/git/gitconfig;
home.file.".ssh/config".source = ../../../dotfiles/ssh/config;

# Or via generation:
programs.bash.bashrcExtra = builtins.readFile ../../../dotfiles/shell/bash/bashrc;
```

**Benefits**:

- ✅ Single source of truth for configs
- ✅ Versioning and review via Git
- ✅ Atomic application: `home-manager switch` or `nixos-rebuild`

---

## 🔀 When to Use What

| Task                                         | Recommended Approach      | Why                                 |
| -------------------------------------------- | ------------------------- | ----------------------------------- |
| OS, services, packages config                | NixOS Flakes              | Reproducibility, testing, rollbacks |
| User aliases, themes, plugins                | Home Manager (in Nix)     | System integration, versioning      |
| Configs for non-Nix systems (Windows, macOS) | Dotfiles + chezmoi        | Cross-platform, minimal deps        |
| Quick config prototyping                     | Direct edit + Git         | Speed, flexibility                  |
| Team config standardization                  | Dotfiles repo + CI checks | Unified style, auto-validation      |

### Hybrid Approach (Recommended)

1. **System layer** (NixOS):
   - Packages, services, users, network, security
   - Managed via `flake.nix` + `nixos-rebuild`

2. **User layer** (Home Manager):
   - Shell, editor, browser, keys
   - Managed via `home-manager switch`

3. **Cross-platform layer** (Dotfiles):
   - SSH, Git, shared aliases
   - Synced via Git, installed via `install` script

---

## ⚙️ Automation & Testing

### Pre-apply validation

```bash
# Nix: syntax and dependency check
nix flake check

# Home Manager: dry-run
home-manager switch --flake .#$(hostname) --dry-run

# Dotfiles: config validation
bash -n ~/.bashrc          # Bash syntax
zsh -n ~/.zshrc           # Zsh syntax
git config --global --get user.name  # Check loading
```

### Testing in isolated environment

```bash
# Nix: build in chroot (if enabled)
sudo nixos-rebuild build --flake .#test-host --impure --option build-users-group ""

# Dotfiles: test install in temp dir
export HOME=$(mktemp -d)
./install --dry-run
```

### CI/CD for configs (optional)

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

## ⚠️ Common Issues

```bash
# Nix: "flake input 'nixpkgs' has changed"
→ Update lock: `nix flake lock --update-input nixpkgs`

# Home Manager: file conflict
→ Remove existing file or use `home.file.<name>.force = true`

# Dotfiles: symlink already exists
→ Use `install --force` or backup first

# Cross-platform aliases not working
→ Verify file is loaded: `echo $BASH_VERSION`, `echo $ZSH_VERSION`

# SSH keys not picked up
→ Check permissions: `chmod 600 ~/.ssh/id_*`, `chmod 644 ~/.ssh/config`
```

---

## Links

- ❄️ [NixOS Manual](https://nixos.org/manual/nixos/stable/)
- 🏠 [Home Manager Docs](https://nix-community.github.io/home-manager/)
- 🧩 [Flakes Reference](https://nixos.wiki/wiki/Flakes)
- 📦 [Chezmoi Documentation](https://www.chezmoi.io/)
- 🐙 [Nix-on-Droid](https://github.com/nix-community/nix-on-droid)
