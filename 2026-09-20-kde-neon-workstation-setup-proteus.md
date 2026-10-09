# KDE neon User Edition Workstation Setup Guide

**Hostname:** `proteus`  
**User:** `morgan`

A workstation setup guide for **KDE neon User Edition**, based on Ubuntu 24.04 LTS with the latest stable KDE Plasma software. KDE neon is rolling for KDE software while retaining an Ubuntu LTS base. This guide favours native DEB packages, uses Flatpak where appropriate, and uses Nix with Home Manager for declarative CLI and development tooling.

> **Scope note:** Sections marked **TBD** require validation on a fresh KDE neon installation before this guide is considered final.

## Contents

1. [Back Up Current Workstation](#1-back-up-current-workstation)
2. [Install KDE neon User Edition](#2-install-kde-neon-user-edition)
3. [First Boot and System Update](#3-first-boot-and-system-update)
4. [Enable Flathub in Discover](#4-enable-flathub-in-discover)
5. [Install Applications](#5-install-applications-native)
6. [Install Citrix Workspace App](#6-install-citrix-workspace-app)
7. [Desktop Configuration](#7-desktop-configuration-tbd)
8. [Install Virtualization Packages](#8-install-virtualization-packages)
9. [Restore Keys](#9-restore-keys)
10. [Install and Configure Nix with Home Manager](#10-install-and-configure-nix-with-home-manager)
11. [Configure Kate](#11-configure-kate)

---

## 1. Back Up Current Workstation

The USB drive path depends on the current distribution:

- openSUSE, Fedora, and similar distributions: `/run/media/morgan/TRANSFERS/`
- Debian, Ubuntu, KDE neon, and derivatives: `/media/morgan/TRANSFERS/`

The commands below assume the current workstation uses the Debian/Ubuntu path. Adjust it if required.

### 1a. Back Up SSH Keys

```bash
cp -r ~/.ssh /media/morgan/TRANSFERS/ssh-backup
```

### 1b. Back Up SOPS Age Key

```bash
cp -r ~/.config/sops /media/morgan/TRANSFERS/sops-backup
```

### 1c. Confirm Chrome Synchronization

Chrome synchronizes bookmarks and extensions through the Google account. Confirm synchronization is enabled and current before reinstalling.

### 1d. Confirm Nextcloud Synchronization

Confirm the Nextcloud desktop client has completed synchronizing all required files before reinstalling.

---

## 2. Install KDE neon User Edition

### 2a. Create the Bootable USB

1. Download **KDE neon User Edition 64-bit** from the official download page:
   - <https://neon.kde.org/download>
2. Download the associated PGP signature.
3. Verify the ISO using the KDE neon ISO signing key identified on the download page.
4. Write the verified ISO to a USB drive using **KDE ISO Image Writer**, **Fedora Media Writer**, **Rufus**, or another trusted imaging tool.
5. Boot the workstation from the USB in **UEFI mode**.

KDE recommends KDE ISO Image Writer and publishes PGP signatures alongside its live images.

### 2b. Start the Installer

After the live desktop loads:

1. Optionally test networking, display output, sound, keyboard, and mouse from the live session.
2. Launch **Install System** from the desktop.
3. Work through the Calamares installer using the settings below.

Calamares normally presents the following stages: Welcome, Location, Keyboard, Partitions, Users, Summary, Install, and Finish. Exact wording can vary slightly with the image.

### 2c. Installer Settings

#### Welcome

- **Language:** American English

#### Location

- **Region:** America
- **Zone:** Edmonton
- Confirm the displayed time corresponds to **Mountain Time**.

#### Keyboard

- **Keyboard model:** Generic 104-key PC, if prompted
- **Layout:** English (US)
- **Variant:** Default
- Test letters, numbers, punctuation, and the numeric keypad.

#### Partitions

Select the intended system drive carefully, then choose:

- **Erase disk**
- **Do not enable encryption**
- Use the default filesystem and partition layout offered by KDE neon
- Confirm the correct bootloader target if the installer presents one

The erase-disk option removes all existing partitions on the selected drive. On UEFI systems, Calamares normally creates an EFI System Partition and the required Linux partitions automatically.

> **Warning:** Verify the selected drive by model and capacity before continuing. Disconnect unrelated external drives if practical.

#### Users

Enter:

- **Your name:** Morgan Smith
- **Login name:** `morgan`
- **Computer name / hostname:** `proteus`
- **Password:** set a strong password
- **Log in automatically:** disabled
- **Use the same password for the administrator account:** enabled, if offered

The Calamares user page normally sets the username and hostname during installation. For that reason, a separate first-boot hostname command should not be required.

#### Summary and Installation

1. Review the selected drive, partition changes, username, hostname, locale, and keyboard layout.
2. Select **Install**.
3. Confirm the destructive disk operation when prompted.
4. When installation finishes, select **Restart now**.
5. Remove the USB drive when prompted.

---

## 3. First Boot and System Update

### 3a. Remove Firefox and Snap

Remove Firefox, Snap, and the Discover Snap backend **before** the first update and reboot:

```bash
sudo apt remove --purge firefox snapd plasma-discover-backend-snap
sudo apt autoremove --purge
```

Before accepting the removal transaction, confirm that it does **not** remove `plasma-discover` or other required KDE neon packages.

### 3b. Install All Updates

KDE neon recommends Plasma Discover or `pkcon` because updates may need to install new packages. Some simpler APT upgrade operations may not do so.

```bash
sudo pkcon refresh
sudo pkcon update
```

Alternatively, install all updates through **Discover → Updates**.

### 3c. Reboot

```bash
sudo reboot
```

After rebooting, open **System Settings → About this System** and confirm the system reports KDE neon User Edition and the expected Plasma version.

---

## 4. Enable Flathub in Discover

Flatpak is already installed on KDE neon, but no remote is configured. Add Flathub through Discover:

1. Open **Discover**.
2. Go to **Settings**.
3. Select **Add Flathub**.

---

## 5. Install Applications (Native)

KDE neon is based on Ubuntu 24.04 LTS, so native Debian packages and Ubuntu-compatible repositories are used.

### 5a. Google Chrome

Download and install the Chrome package, then remove the downloaded file:

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt install ./google-chrome-stable_current_amd64.deb
rm google-chrome-stable_current_amd64.deb
```

Launch Chrome, sign in, and confirm bookmarks and extensions synchronize.

### 5b. ONLYOFFICE Desktop Editors

Add the official ONLYOFFICE repository and install the native package:

```bash
curl -fsSL https://download.onlyoffice.com/GPG-KEY-ONLYOFFICE \
  | sudo gpg --dearmor -o /usr/share/keyrings/onlyoffice.gpg

echo "deb [signed-by=/usr/share/keyrings/onlyoffice.gpg] https://download.onlyoffice.com/repo/debian squeeze main" \
  | sudo tee /etc/apt/sources.list.d/onlyoffice.list

sudo apt update
sudo apt install onlyoffice-desktopeditors fonts-croscore fonts-crosextra-caladea
```

ONLYOFFICE supports installation from its Debian repository on Ubuntu derivatives.

### 5c. Nextcloud Desktop Client

Install the native Nextcloud client and Dolphin integration from the standard repositories:

```bash
sudo apt update
sudo apt install nextcloud-desktop dolphin-nextcloud
```

Launch Nextcloud from the application menu and configure:

- **Server URL:** enter the existing Nextcloud server address
- **Authentication:** sign in using the normal account
- **Local folder:** `~/Nextcloud`
- Confirm the initial synchronization completes successfully.

### 5d. KDE ISO Image Writer

```bash
sudo apt install isoimagewriter
```

### 5e. Ubuntu Restricted Extras

```bash
sudo apt install ubuntu-restricted-extras
```

Accept the Microsoft core fonts EULA when prompted.

---

## 6. Install Citrix Workspace App

Citrix provides a native Debian package and supports Ubuntu 24.04 x86-64. Citrix recommends using the distribution package format because dependencies can be resolved by the package manager.

### 6a. Download the Package

1. Open the [Citrix Workspace app for Linux download page](https://www.citrix.com/downloads/workspace-app/linux/).
2. Download the **Debian Full Package (Self-Service Support), x86_64**.
3. Save it in `~/Downloads/`.

### 6b. Install Citrix Workspace

```bash
sudo dpkg -i /home/morgan/Downloads/icaclient*_amd64.deb
sudo apt install -f
```

> **Note:** The installer asks several questions. Answer them as follows:
>
> - **App protection component:** select **No**. This component can cause Citrix Workspace to crash.
> - **deviceTRUST component:** select **No**, unless corporate IT specifically requires it. Skipping it keeps the installation clean.
> - **EPA component:** select **No**. Endpoint Analysis is only needed if the corporate login page runs a security scan before allowing a connection.

### 6c. Launch and Verify

```bash
/opt/Citrix/ICAClient/selfservice
```

Accept the EULA when prompted, add the required store or account, and test launching a published desktop or application. Citrix documents the same executable for starting the self-service interface.

### 6d. Clean Up

```bash
rm /home/morgan/Downloads/icaclient*_amd64.deb
```

If display scaling or multi-monitor behaviour is poor under Wayland, test a Plasma X11 session if it is available, or adjust scaling inside the remote Windows session.

---

## 7. Desktop Configuration (TBD)

> **TBD: Finalize after testing the default KDE neon Plasma configuration.**

### 7a. Appearance

Open **System Settings → Quick Settings** and set **Theme** to **Breeze Dark**.

### 7b. Removable Storage and USB Automount

Check **System Settings → Disks & Cameras → Device Auto-Mount** and determine the preferred behaviour for:

- Automatically mounting removable media at login
- Automatically mounting media when attached
- Opening removable drives in Dolphin

### 7c. Num Lock

Check **System Settings → Keyboard → NumLock on Plasma Startup** and set it to **Turn On**.

### 7d. Pin Google Chrome to the Dock

1. Open the **Application Launcher** and search for **Google Chrome**.
2. Right-click **Google Chrome** and select **Pin to Task Manager**.

---

## 8. Install Virtualization Packages

Install KVM/QEMU, libvirt, Virtual Machine Manager, UEFI firmware, and TPM emulation:

```bash
sudo apt update
sudo apt install \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  virt-manager \
  bridge-utils \
  ovmf \
  swtpm
```

Enable libvirt and add the user to the required groups:

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm "$USER"
```

Reboot or log out and back in so the new group memberships take effect:

```bash
sudo reboot
```

Verify access:

```bash
virsh list --all
```

---

## 9. Restore Keys

The commands below assume the backup USB is mounted at `/media/morgan/TRANSFERS/`.

### 9a. Restore SSH Keys

```bash
cp -r /media/morgan/TRANSFERS/ssh-backup ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub 2>/dev/null
chmod 644 ~/.ssh/known_hosts 2>/dev/null
chmod 644 ~/.ssh/config 2>/dev/null
```

Review the directory afterward to ensure private keys are not world-readable:

```bash
ls -la ~/.ssh
```

### 9b. Restore SOPS Age Key

```bash
mkdir -p ~/.config/sops
cp -r /media/morgan/TRANSFERS/sops-backup/* ~/.config/sops/
chmod 600 ~/.config/sops/age/keys.txt
```

Verify the key exists:

```bash
ls -l ~/.config/sops/age/keys.txt
```

---

## 10. Install and Configure Nix with Home Manager

This section follows the declarative Nix and Home Manager design from the supplied setup guide: Determinate Nix, root-level maintenance timers, a Home Manager flake, weekly Home Manager updates, user garbage collection, and Plasma access to Nix-installed tools.

### 10a. Install Nix

Install Nix using the Determinate Systems installer with automatic store optimization and sandboxing disabled for compatibility with restricted unprivileged user namespaces:

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix \
  | sh -s -- install \
      --extra-conf "auto-optimise-store = true" \
      --extra-conf "sandbox = false"
```

Load the Nix environment and verify the installation:

```bash
. /nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh
nix --version
```

### 10b. Configure Determinate Nix Auto-Upgrade

Create the service:

```bash
sudo tee /etc/systemd/system/determinate-nix-upgrade.service > /dev/null <<'EOF'
[Unit]
Description=Upgrade Determinate Nix Daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/determinate-nixd upgrade
EOF
```

Create the timer:

```bash
sudo tee /etc/systemd/system/determinate-nix-upgrade.timer > /dev/null <<'EOF'
[Unit]
Description=Weekly upgrade of Determinate Nix Daemon

[Timer]
OnCalendar=Sun *-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

Enable and test it:

```bash
sudo systemctl daemon-reload
sudo systemctl start determinate-nix-upgrade.service
sudo systemctl status determinate-nix-upgrade.service
sudo systemctl enable --now determinate-nix-upgrade.timer
sudo systemctl status determinate-nix-upgrade.timer
```

If the service cannot find `determinate-nixd`, locate it and update `ExecStart`:

```bash
command -v determinate-nixd
```

### 10c. Configure Root-Level Nix Garbage Collection

Create the service:

```bash
sudo tee /etc/systemd/system/nix-gc.service > /dev/null <<'EOF'
[Unit]
Description=Nix Root Garbage Collection

[Service]
Type=oneshot
ExecStart=/nix/var/nix/profiles/default/bin/nix-collect-garbage -d
EOF
```

Create the timer:

```bash
sudo tee /etc/systemd/system/nix-gc.timer > /dev/null <<'EOF'
[Unit]
Description=Run Nix Root Garbage Collection Weekly

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

Enable and test it:

```bash
sudo systemctl daemon-reload
sudo systemctl start nix-gc.service
sudo systemctl status nix-gc.service
sudo systemctl enable --now nix-gc.timer
sudo systemctl status nix-gc.timer
```

### 10d. Create the Home Manager Flake

```bash
mkdir -p /home/morgan/.config/home-manager
```

Create `/home/morgan/.config/home-manager/flake.nix`:

```bash
cat > /home/morgan/.config/home-manager/flake.nix <<'EOF'
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
EOF
```

### 10e. Create the Home Manager Configuration

Create `/home/morgan/.config/home-manager/home.nix`:

```bash
cat > /home/morgan/.config/home-manager/home.nix <<'EOF'
{ config, pkgs, ... }:
{
  home.username = "morgan";
  home.homeDirectory = "/home/morgan";
  home.stateVersion = "26.05";

  programs.home-manager.enable = true;

  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 30d";
  };

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

  home.packages = with pkgs; [
    sops
    age
    go
    yaml-language-server
    bash-language-server
    vscode-langservers-extracted
    marksman
    nil
  ];

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

  programs.bash = {
    enable = true;
    initExtra = ''
      # Full repo dump. Respects .gitignore and excludes LICENSE and repo_dump.txt.
      repodump() {
          git ls-files \
              | grep -vE '^(LICENSE|repo_dump\\.txt)$' \
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
EOF
```

> **Validation note:** Confirm that `home.stateVersion = "26.05"` is accepted by the Home Manager release resolved when this guide is used. If it is rejected, use the current valid Home Manager state version shown in its release notes. Do not change it casually after the first successful activation.

### 10f. Activate Home Manager

Bootstrap Home Manager and back up conflicting unmanaged files:

```bash
nix run home-manager/master -- switch \
  --flake /home/morgan/.config/home-manager#morgan \
  -b backup
```

Future manual rebuilds use:

```bash
home-manager switch --flake /home/morgan/.config/home-manager#morgan
```

### 10g. Make Nix Tools Available to Plasma Applications

GUI applications launched from Plasma do not read `.bashrc`. Add the Nix profile to the systemd user environment:

```bash
mkdir -p /home/morgan/.config/environment.d
cat > /home/morgan/.config/environment.d/10-nix.conf <<'EOF'
PATH=$HOME/.nix-profile/bin:$PATH
EOF
```

Log out and back in so Plasma applications such as Kate can find Nix-installed language servers.

### 10h. Verify Home Manager Automation

```bash
systemctl --user start home-manager-auto-update.service
systemctl --user status home-manager-auto-update.service
journalctl --user -u hm-auto-update-runner.service -n 30 --no-pager
systemctl --user list-timers --all | grep home-manager
```

---

## 11. Configure Kate

Kate is included with KDE neon. Install `markdownpart` so Kate's Document Preview plugin can render Markdown:

```bash
sudo apt update
sudo apt install markdownpart
```

### 11a. Enable Useful Plugins

Open **Kate → Settings → Configure Kate → Plugins** and enable:

- **Project Plugin**: project tree and quick file access
- **LSP Client**: language-server support
- **Git Blame**: inline Git history information
- **Document Preview**: rendered Markdown preview using `markdownpart`

### 11b. Configure Spell Checking

Open **Settings → Configure Kate → Editing → Spellcheck** and set:

- **Default language:** English (Canada), if available; otherwise American English
- **Preferred languages:** English (Canada) or American English
- **Enable autodetection of language:** disabled
- **Automatic spell checking enabled by default:** enabled
- **Skip all uppercase words:** enabled
- **Skip run-together words:** enabled

### 11c. Verify Language Servers

After logging out and back in, verify that Kate can locate the Home Manager packages:

```bash
command -v yaml-language-server
command -v bash-language-server
command -v vscode-html-language-server
command -v marksman
command -v nil
```

Open representative `.yaml`, `.sh`, `.html`, `.json`, and `.md` files and confirm diagnostics, completion, and Markdown preview work.

---

## References

- KDE neon download and image information: <https://neon.kde.org/download>
- KDE neon FAQ: <https://neon.kde.org/faq>
- KDE neon update guidance: <https://blog.neon.kde.org/2017/06/16/how-do-i-to-update-to-the-latest-software/>
- Calamares user guide: <https://calamares.io/docs/users-guide/>
- Calamares partition options: <https://calamares.io/docs/partitions/>
- Citrix system requirements: <https://docs.citrix.com/en-us/citrix-workspace-app-for-linux/system-requirements.html>
- Citrix installation instructions: <https://docs.citrix.com/en-us/citrix-workspace-app-for-linux/installation.html>
- ONLYOFFICE Debian/Ubuntu installation: <https://helpcenter.onlyoffice.com/desktop/installation/desktop-install-ubuntu.aspx>
- Supplied Pop!_OS workstation guide.
- Supplied Ultramarine workstation guide.
- Supplied Declarative Nix and Home Manager Setup Guide.

**Version:** 0.2 Draft  
**Last updated:** September 30, 2026
