# Declarative Nix and Home Manager Setup Guide

This guide details the step-by-step process for installing Nix via the Determinate Systems installer, configuring root-level systemd timers for Nix upgrades and garbage collection, and setting up an automated, declarative Home Manager environment on Linux distributions like Ubuntu.

## 1. Nix Installation and Initial Setup

Install Nix using the Determinate Systems installer. This command enables automatic store optimization and disables sandboxing (required for Ubuntu due to restricted unprivileged user namespaces by default).

```
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install --extra-conf "auto-optimise-store = true" --extra-conf "sandbox = false"

```

After installation completes, source the Nix environment and verify the installation:

```
. /nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh
nix --version

```

## 2. Root-Level System Maintenance (Systemd Services)

Set up systemd services and timers to automate daemon upgrades and root-level garbage collection.

### 2.1 Determinate Nix Daemon Auto-Upgrade Service

1. Create `/etc/systemd/system/determinate-nix-upgrade.service`:

```
[Unit]
Description=Upgrade Determinate Nix Daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/determinate-nixd upgrade

```

2. Create `/etc/systemd/system/determinate-nix-upgrade.timer`:

```
[Unit]
Description=Weekly upgrade of Determinate Nix Daemon

[Timer]
OnCalendar=Sun *-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target

```

3. Reload systemd, test, and enable the timer:

```
sudo systemctl daemon-reload
sudo systemctl start determinate-nix-upgrade.service
sudo systemctl status determinate-nix-upgrade.service
sudo systemctl enable --now determinate-nix-upgrade.timer
sudo systemctl status determinate-nix-upgrade.timer

```

### 2.2 Root-Level Garbage Collection

1. Create `/etc/systemd/system/nix-gc.service`:

```
[Unit]
Description=Nix Root Garbage Collection

[Service]
Type=oneshot
ExecStart=/nix/var/nix/profiles/default/bin/nix-collect-garbage -d

```

2. Create `/etc/systemd/system/nix-gc.timer`:

```
[Unit]
Description=Run Nix Root Garbage Collection Weekly

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target

```

3. Reload systemd, test, and enable the timer:

```
sudo systemctl daemon-reload
sudo systemctl start nix-gc.service
sudo systemctl status nix-gc.service
sudo systemctl enable --now nix-gc.timer
sudo systemctl status nix-gc.timer

```

## 3. Declarative Home Manager Setup

Create the directory for your Home Manager configuration:

```
mkdir -p /home/morgan/.config/home-manager

```

### 3.1 Flake Configuration (`/home/morgan/.config/home-manager/flake.nix`)

Create `/home/morgan/.config/home-manager/flake.nix` with the following content:

```
{
  description = "Home Manager configuration for morgan";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { nixpkgs, home-manager, ... }:
    let
      system = "x86_64-linux";
      pkgs = nixpkgs.legacyPackages.${system};
    in {
      homeConfigurations."morgan" = home-manager.lib.homeManagerConfiguration {
        inherit pkgs;
        modules = [ ./home.nix ];
      };
    };
}

```

### 3.2 Home Configuration (`/home/morgan/.config/home-manager/home.nix`)

Create `/home/morgan/.config/home-manager/home.nix`:

```
{ config, pkgs, ... }:

{
  # ---------------------------------------------------------------------------
  # Basic User and Environment Configuration
  # ---------------------------------------------------------------------------
  home.username = "morgan";
  home.homeDirectory = "/home/morgan";
  home.stateVersion = "26.05"; 
  programs.home-manager.enable = true;

  # ---------------------------------------------------------------------------
  # Automatic Garbage Collection of User Nix Profiles
  # ---------------------------------------------------------------------------
  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 30d";
  };

  # ---------------------------------------------------------------------------
  # Automatic Update of Flake Inputs and Installed Packages
  # ---------------------------------------------------------------------------
  systemd.user.services.home-manager-auto-update = {
    Unit.Description = "Auto-update Home Manager flake inputs and apply changes";
    Service = {
      Type = "oneshot";
      ExecStart = toString (pkgs.writeShellScript "hm-auto-update-launcher" ''
        set -euo pipefail
        ${pkgs.systemd}/bin/systemd-run --user --unit=hm-auto-update-runner ${pkgs.writeShellScript "hm-auto-update-exec" ''
          set -euo pipefail
          export PATH="/nix/var/nix/profiles/default/bin:$PATH"
          ${pkgs.nix}/bin/nix flake update --flake /home/morgan/.config/home-manager
          ${pkgs.home-manager}/bin/home-manager switch --flake /home/morgan/.config/home-manager#morgan
        ''}
      '');
    };
  };

  systemd.user.timers.home-manager-auto-update = {
    Unit.Description = "Weekly timer for Home Manager updates";
    Timer = {
      OnCalendar = "weekly";
      Persistent = true;
    };
    Install.WantedBy = [ "timers.target" ];
  };

  # ---------------------------------------------------------------------------
  # Software Packages to Install
  # ---------------------------------------------------------------------------
  home.packages = with pkgs; [
    sops
    age
    go
    yaml-language-server
    bash-language-server
    vscode-langservers-extracted
    marksman
  ];

  # ---------------------------------------------------------------------------
  # Git Configuration
  # ---------------------------------------------------------------------------
  programs.git = {
    enable = true;
    settings = {
      user = {
        name = "smithoo4";
        email = "13087392+Smithoo4@users.noreply.github.com";
      };
      init.defaultBranch = "main";
    };
  };

  # ---------------------------------------------------------------------------
  # Bash Configuration
  # ---------------------------------------------------------------------------
  programs.bash = {
    enable = true;
    initExtra = ''
      # Full repo dump (respects .gitignore, excludes LICENSE and repo_dump.txt)
      repodump() {
          git ls-files \
              | grep -vE '^(LICENSE|repo_dump\.txt)$' \
              | sort \
              | while read -r f; do
                  echo "===== $f ====="
                  cat "$f"
                  echo ""
              done > repo_dump.txt \
              && echo "[OK] repo_dump.txt created"
      }
    '';
  };
}

```

## 4. Activation and Switch Workflow

To bootstrap Home Manager for the first time without pre-installing the standalone executable, run `nix run` with the `-b backup` flag to safely move any existing unmanaged files (like `.bashrc` or `.profile`):

```
nix run home-manager/master -- switch --flake /home/morgan/.config/home-manager#morgan -b backup

```

Once Home Manager has been activated, future manual updates or configuration rebuilds can be run directly via the managed `home-manager` command:

```
home-manager switch --flake /home/morgan/.config/home-manager#morgan

```

## 5. Verification and Troubleshooting

### 5.1 Test Auto-Update Service

Manually trigger the Home Manager auto-update service to ensure it executes without errors:

```
systemctl --user start home-manager-auto-update.service
systemctl --user status home-manager-auto-update.service

```

### 5.2 Check Update Execution Logs

Inspect the execution logs created by the decoupled runner process:

```
journalctl --user -u hm-auto-update-runner.service -n 30 --no-pager

```

## 6. Desktop Environment Integration (GUI Applications & LSPs)

Desktop applications (such as Kate or VS Code launched from application menus) do not source shell files like `.bashrc` directly. To expose binaries installed via Nix (such as language servers) to GUI programs:

```
mkdir -p /home/morgan/.config/environment.d
echo 'PATH=$HOME/.nix-profile/bin:$PATH' > /home/morgan/.config/environment.d/10-nix.conf

```

> **Note:** Log out and log back into your desktop session for the systemd user environment generator to pick up the new configuration.
