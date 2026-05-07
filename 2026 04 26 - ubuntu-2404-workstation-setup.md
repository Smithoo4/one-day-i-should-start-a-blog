# Ubuntu 24.04 LTS Workstation Setup Guide

> **Hostname:** `<hostname>` — come up with a meaningful name for each new machine before starting setup.

---

## 1. Backup Current Workstation

### 1a. Backup Virtual Machines (virt-manager)

VM disk images are large, so back them up to Nextcloud or an external drive.

```bash
# List all VMs
virsh list --all

# Find the disk image paths
virsh dumpxml <vm-name> | grep "source file"

# Export the VM XML definition
virsh dumpxml <vm-name> > ~/vm-<vm-name>.xml

# Copy disk images (typically in /var/lib/libvirt/images/)
sudo cp /var/lib/libvirt/images/<vm-disk>.qcow2 ~/

# Move both the XML and qcow2 to Nextcloud or external storage
cp ~/vm-<vm-name>.xml /media/morgan/TRANSFERS/vm-backup/
cp ~/<vm-disk>.qcow2 /media/morgan/TRANSFERS/vm-backup/
```

> **Note:** If the qcow2 file is too large for Nextcloud sync, use an external USB drive instead.

### 1b. Backup SSH Keys

```bash
cp -r ~/.ssh /media/morgan/TRANSFERS/ssh-backup
```

### 1c. Backup SOPS Age Key

```bash
cp -r ~/.config/sops /media/morgan/TRANSFERS/sops-backup
```

### 1d. Backup Zed Configuration

```bash
cp -r ~/.config/zed /media/morgan/TRANSFERS/zed-backup
```

The key files inside are:
- `settings.json` — your editor settings
- `keymap.json` — any custom keybindings

> Safely eject the USB drive. You will need it after the fresh install.

---

## 2. Install Ubuntu 24.04 LTS (Minimal)

### 2a. Create Bootable USB

Download the Ubuntu 24.04 LTS ISO from [ubuntu.com](https://ubuntu.com/download/desktop) and write it to a USB drive using [Balena Etcher](https://etcher.balena.io/) or `dd`.

### 2b. Installation Options

- Select **Minimal Installation** when prompted.
- **Hostname:** Enter your chosen hostname
- **User:** Create user `morgan` with a strong password.
- Ubuntu does not create a root account by default — `morgan` will have `sudo` access automatically.

### 2c. First Boot — Update the System

Before doing anything else, update all packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

---

## 3. Install Google Chrome and Remove Firefox/Thunderbird

### 3a. Install Google Chrome

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt install -f ./google-chrome-stable_current_amd64.deb
rm google-chrome-stable_current_amd64.deb
```

The `.deb` package automatically adds the Google apt repository for future updates.

Open Chrome and sign in to sync bookmarks, extensions, and settings.

### 3b. Remove Firefox and Thunderbird

```bash
# Remove Firefox (snap version on Ubuntu)
sudo snap remove firefox

# Remove Thunderbird if installed
sudo snap remove thunderbird
```

> **Optional:** If you want to remove `snapd` entirely:
> ```bash
> sudo apt remove --purge snapd
> sudo apt-mark hold snapd
> ```
> **Note:** Removing snapd will prevent any snap-based installs later in this guide. Skip this if you intend to use snaps.

---

## 4. Install Multimedia Codecs

```bash
sudo add-apt-repository multiverse
sudo apt update
sudo apt install ubuntu-restricted-extras
```

Accept the Microsoft fonts EULA when prompted (Tab to OK, Enter to confirm).

For DVD playback (optional):

```bash
sudo apt install libdvd-pkg
sudo dpkg-reconfigure libdvd-pkg
```

---

## 5. Install Nextcloud Desktop Client

The snap version of the Nextcloud desktop client did not work reliably on Ubuntu 24.04. Use the official Nextcloud PPA instead:

```bash
sudo add-apt-repository ppa:nextcloud-devs/client
sudo apt update
sudo apt install nextcloud-desktop
```

Launch Nextcloud from the application menu:

1. Enter your Nextcloud server URL
2. Log in with your credentials
3. Set the local sync folder to `~/Nextcloud`
4. Start the sync

> Wait for the sync to complete before restoring VMs from Nextcloud (Step 10).

---

## 6. Restore SSH Keys and SOPS Age Key

### 6a. SSH Keys

```bash
# Insert the USB drive
cp -r /media/morgan/TRANSFERS/ssh-backup ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub 2>/dev/null
chmod 644 ~/.ssh/known_hosts 2>/dev/null
chmod 644 ~/.ssh/config 2>/dev/null
```

### 6b. SOPS Age Key

```bash
mkdir -p ~/.config/sops
cp -r /media/morgan/TRANSFERS/sops-backup/* ~/.config/sops/
chmod 600 ~/.config/sops/age/keys.txt
```

### 6c. Test SSH Connections

```bash
# Test connection to a server
ssh user@your-server-hostname

# Test GitHub SSH
ssh -T git@github.com
```

Expected GitHub output: `Hi Smithoo4! You've successfully authenticated...`

---

## 7. Install Git and SOPS

### 7a. Git

```bash
sudo apt install git
```

Configure Git:

```bash
git config --global user.name "Morgan Smith"
git config --global user.email "13087392+Smithoo4@users.noreply.github.com"
git config --global init.defaultBranch main
```

### 7b. SOPS (via WakeMeOps)

WakeMeOps is a community-maintained third-party apt repository that packages many DevOps tools including SOPS. It is not an official Canonical or SOPS project repo, but is widely used and well regarded.

```bash
curl -sSL https://raw.githubusercontent.com/upciti/wakemeops/main/assets/install_repository | sudo bash
sudo apt update
sudo apt install sops
```

Verify:

```bash
sops --version
```

---

## 8. Install Nix Package Manager

Nix is installed using the Determinate Systems installer, which enables flakes by default and integrates cleanly with Ubuntu's systemd.

> **Note:** The Determinate Systems installer is a well-regarded third-party script, similar in nature to WakeMeOps used for SOPS. It is not the official Nix installer but is widely recommended for non-NixOS systems.

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
```

Follow the prompts and accept the defaults. Once complete, open a new terminal session and verify the install:

```bash
nix run 'nixpkgs#hello'
# Expected output: Hello, world!
```

### 8a. Enable Auto Store Optimisation

Determinate Nix uses `/etc/nix/nix.custom.conf` for user modifications (do not edit `nix.conf` directly — it warns you it will be replaced). Enable automatic store deduplication:

```bash
echo "auto-optimise-store = true" | sudo tee -a /etc/nix/nix.custom.conf
sudo systemctl restart nix-daemon
```

### 8b. Set Up Periodic Garbage Collection

Nix accumulates old store paths over time. Set up a monthly cron job to clean them up:

```bash
crontab -e
```

Add the following line:

```cron
0 3 1 * * /nix/var/nix/profiles/default/bin/nix store gc
```

This runs `nix store gc` at 3am on the first of every month. You can also run it manually at any time:

```bash
nix store gc
```

To check how much space the Nix store is using:

```bash
du -sh /nix
```

---

## 9. Set Up Podman (Rootless) for User morgan

```bash
sudo apt install podman
```

Verify rootless operation:

```bash
podman run --rm docker.io/library/hello-world
```

Enable Podman socket for user:

```bash
systemctl --user enable --now podman.socket
```

> Podman on Ubuntu 24.04 ships version 4.x+ which supports Quadlet systemd units natively.

---

## 10. Set Up virt-manager and Restore VM

### 10a. Install virt-manager

```bash
sudo apt install virt-manager qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
```

Add your user to the libvirt group:

```bash
sudo usermod -aG libvirt morgan
sudo usermod -aG kvm morgan
```

Log out and back in for group changes to take effect.

Verify:

```bash
virsh list --all
```

### 10b. Restore the VM

Copy the disk image and XML definition from Nextcloud (or external storage):

```bash
sudo cp ~/Nextcloud/vm-backup/<vm-disk>.qcow2 /var/lib/libvirt/images/
sudo chown libvirt-qemu:kvm /var/lib/libvirt/images/<vm-disk>.qcow2
```

Edit the XML if the disk path has changed:

```bash
nano ~/Nextcloud/vm-backup/vm-<vm-name>.xml
# Verify the <source file="..."/> path matches /var/lib/libvirt/images/<vm-disk>.qcow2
```

Import the VM:

```bash
virsh define ~/Nextcloud/vm-backup/vm-<vm-name>.xml
virsh start <vm-name>
```

### 10c. Check VM IP and Update DNS

```bash
virsh domifaddr <vm-name>
```

If the IP has changed, update your DNS records accordingly.

---

## 11. Set Up Zed Editor

### 11a. Check Vulkan Support

Zed requires a GPU with Vulkan support. Verify before installing:

```bash
sudo apt install vulkan-tools
vkcube
```

If a spinning cube appears, you are good to go. For AMD/Intel GPUs, install Mesa Vulkan drivers if needed:

```bash
sudo apt install mesa-vulkan-drivers
```

### 11b. Install Zed

```bash
curl -f https://zed.dev/install.sh | sh
```

Zed installs to `~/.local/bin/zed` and updates itself automatically. Launch it:

```bash
zed
```

### 11c. Install nil (Nix Language Server)

The Zed Nix extension requires `nil` to be installed manually — it is not in Ubuntu's apt repositories. Install it via Nix:

```bash
nix profile install nixpkgs#nil
```

Verify it is in your PATH:

```bash
which nil
```

Optionally install `nixfmt` for Nix file formatting:

```bash
nix profile install nixpkgs#nixfmt-rfc-style
```

### 11d. Install Extensions

Open the Command Palette with `Ctrl+Shift+P` and type `zed: extensions`, then search for and install each of:

- **Harper** — grammar and spell checking (downloads `harper-ls` automatically, no separate install needed)
- **Marksman** — Markdown LSP for link validation and heading navigation
- **Nix** — syntax highlighting and LSP support for `.nix` files
- **HTML** — language support for HTML files
- **Git Firefly** — enhanced Git blame and history viewer inline in the editor

### 11e. Configure Settings

Open settings with `Ctrl+,` and replace the contents with the following:

```json
// Zed settings
//
// For information on how to configure Zed, see the Zed
// documentation: https://zed.dev/docs/configuring-zed
//
// To see all of Zed's default settings without changing your
// custom settings, run `zed: open default settings` from the
// command palette (ctrl-shift-p)
{
  "cli_default_open_behavior": "existing_window",
  "ui_font_size": 16,
  "buffer_font_size": 15,
  "theme": {
    "mode": "system",
    "light": "Gruvbox Light",
    "dark": "Gruvbox Dark",
  },
  "lsp": {
    "harper-ls": {
      "settings": {
        "harper-ls": {
          "dialect": "Canadian",
        },
      },
    },
  },
  "languages": {
    "Markdown": {
      "soft_wrap": "editor_width",
      "preferred_line_length": 80,
      "language_servers": ["harper-ls"],
    },
    "Nix": {
      "tab_size": 2,
      "language_servers": ["nil", "!nixd"],
      "formatter": {
        "external": {
          "command": "nixfmt",
          "arguments": ["--quiet", "--"],
        },
      },
    },
  },
}
```

> **Note:** Zed uses JSONC (JSON with Comments), so trailing commas are fine and comments are supported.

> **Note on `"!nixd"`:** The Nix extension tries to use both `nil` and `nixd` by default. The `!nixd` disables nixd since only `nil` is installed.

### 11f. Restore Zed Configuration from Backup

If restoring from a previous machine, you can copy your backed-up config directly:

```bash
cp -r /media/morgan/TRANSFERS/zed-backup/* ~/.config/zed/
```

> **Note:** This has not been fully tested across machines. Extensions are not stored in `~/.config/zed` and will still need to be reinstalled manually via the Extensions panel (Step 11d). Only `settings.json` and `keymap.json` are restored this way.

### 11g. Key Shortcuts to Know

| Action | Shortcut |
|---|---|
| Command Palette | `Ctrl+Shift+P` |
| Open file | `Ctrl+P` |
| Settings | `Ctrl+,` |
| Find in file | `Ctrl+F` |
| Find in project | `Ctrl+Shift+F` |
| Toggle sidebar | `Ctrl+B` |
| Open terminal | `Ctrl+`` ` |
| Split pane | `Ctrl+\` |
| Code actions (Harper fixes) | `Ctrl+.` |

---

## 12. Install Citrix Workspace App

### 12a. Download

Go to [Citrix Downloads](https://www.citrix.com/downloads/workspace-app/linux/workspace-app-for-linux-latest.html) and download the **Debian Full Package (.deb)** for x86_64.

### 12b. Install

```bash
sudo apt install -f ./icaclient_*_amd64.deb
```

### 12c. Fix webkit2gtk Dependency (Ubuntu 24.04) — Last Resort Only

> **Note:** Citrix Workspace app 2601 and later bundle the webkit2gtk-4.0 library directly, so this workaround should no longer be needed. Try installing and launching Citrix first. Only follow these steps if you see webkit2gtk-related errors with an older package.

Ubuntu 24.04 ships `webkit2gtk-4.1` but older Citrix packages expect `webkit2gtk-4.0`:

```bash
# Add jammy repos temporarily
sudo apt-add-repository "deb http://us.archive.ubuntu.com/ubuntu jammy main"
sudo apt-add-repository "deb http://us.archive.ubuntu.com/ubuntu jammy-updates main"
sudo apt-add-repository "deb http://us.archive.ubuntu.com/ubuntu jammy-security main"

sudo apt install libwebkit2gtk-4.0-dev

# Remove the jammy repos after install
sudo apt-add-repository -r "deb http://us.archive.ubuntu.com/ubuntu jammy main"
sudo apt-add-repository -r "deb http://us.archive.ubuntu.com/ubuntu jammy-updates main"
sudo apt-add-repository -r "deb http://us.archive.ubuntu.com/ubuntu jammy-security main"
sudo apt update
```

### 12d. SSL Certificates

Link system certificates so Citrix can find them:

```bash
sudo ln -sf /etc/ssl/certs/* /opt/Citrix/ICAClient/keystore/cacerts/
```

### 12e. Configure and Test

1. Launch **Citrix Workspace** from the application menu or run:
   ```bash
   /opt/Citrix/ICAClient/selfservice
   ```
2. Enter your Citrix server URL
3. Log in with your credentials

---

## Post-Install Checklist

- [ ] WiFi connected and working
- [ ] Google Chrome signed in and syncing
- [ ] Firefox and Thunderbird removed
- [ ] Media codecs installed (test an MP3/MP4 file)
- [ ] Nextcloud PPA installed and syncing to ~/Nextcloud
- [ ] SSH keys working (`ssh -T git@github.com`)
- [ ] SOPS age key in place
- [ ] Git configured (`git config --global --list`)
- [ ] SOPS installed and working (`sops --version`)
- [ ] Nix installed and working (`nix run 'nixpkgs#hello'`)
- [ ] Nix auto-optimise enabled (`cat /etc/nix/nix.custom.conf`)
- [ ] Nix GC cron job set up (`crontab -l`)
- [ ] Podman running rootless containers
- [ ] VM restored and accessible in virt-manager
- [ ] VM IP/DNS records verified
- [ ] Zed installed and launching
- [ ] Vulkan working (`vkcube`)
- [ ] nil installed (`which nil`)
- [ ] Zed extensions installed (Harper, Marksman, Nix)
- [ ] Zed settings.json configured
- [ ] Citrix Workspace connecting successfully
- [ ] USB backup drive wiped (`sudo shred -vuz -n 3 /dev/sdX` — replace `sdX` with your USB device)
