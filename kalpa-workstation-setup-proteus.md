# openSUSE Kalpa Workstation Setup Guide

**Hostname:** `proteus`

A workstation setup guide for [openSUSE Kalpa](https://kalpadesktop.org/) with KDE Plasma, installed via the Agama installer.

Kalpa is fundamentally different from Tumbleweed and traditional distros. The root filesystem is **read-only** and managed through atomic btrfs snapshots. Software is installed primarily via **Flatpak**, and any host OS-level package changes go through `transactional-update`, which builds a new snapshot that takes effect after a reboot. **[linuxbrew](https://docs.brew.sh/Homebrew-on-Linux)** will be added to install command line (CLI) software not in the default repository.

---

## 1. Backup Current Workstation

> **Note on USB mount paths:** the path your USB drive lands on depends on the running distro:
> - openSUSE / Fedora / other systemd-mount distros: `/run/media/morgan/TRANSFERS/`
> - Debian / Ubuntu and derivatives: `/media/morgan/TRANSFERS/`

### 1a. Backup SSH Keys

```bash
cp -r ~/.ssh /run/media/morgan/TRANSFERS/ssh-backup
```

### 1b. Backup SOPS Age Key

```bash
cp -r ~/.config/sops /run/media/morgan/TRANSFERS/sops-backup
```

### 1c. Note Current Browser Bookmarks / Extensions

Chrome syncs via your Google account, so just confirm sync is enabled before reinstalling.

---

## 2. Install openSUSE Kalpa (via Agama)

### 2a. Create Bootable USB

Download the **Kalpa** ISO:

- [Kalpa Offline installer](https://download.opensuse.org/repositories/devel:/microos:/kalpa:/images/images/iso/Kalpa_Desktop.x86_64-Offline.iso)
- [Kalpa Online installer](https://download.opensuse.org/repositories/devel:/microos:/kalpa:/images/images/iso/Kalpa_Desktop.x86_64-Online.iso)

Write it to a USB drive.

### 2b. Installation Options

At the system role screen, choose **openSUSE Kalpa (Plasma)** — it may be labelled Alpha.

- **Localization:** set timezone, keyboard, language as appropriate
- **Network:** connect to WiFi or Ethernet
- **Storage:** Kalpa uses the full disk by default with encrypted Btrfs; there is no partition choice during install. zram swap is enabled by default.
- **Authentication:** create user `morgan` with a strong password; enable **Use this password for system administrator**
- **System:** hostname `proteus` — set as **static**

Click **Install**.

---

## 3. Understanding Kalpa Software Sources

Kalpa has a strict, intentional philosophy about software installation.

### Flatpak (primary — use this for everything possible)

Flathub is enabled by default in Discover. For almost all desktop apps, Flatpak is the right answer. It keeps the base system clean, doesn't interfere with transactional updates, and survives rollbacks.

```bash
# Flatpak is pre-installed; verify Flathub is enabled
flatpak remotes
# Should show: flathub
```

### transactional-update (host OS only)

For packages that need to live on the host OS — drivers, kernel modules, system daemons, compilers, and core tools — use `transactional-update`. It creates a new btrfs snapshot and the changes take effect after a reboot.

```bash
sudo transactional-update pkg install <package>   # install
sudo transactional-update pkg remove <package>    # remove
sudo transactional-update dup                     # full system upgrade
sudo reboot                                       # required to activate changes
```

> **Important:** Multiple `transactional-update pkg install` calls before rebooting overwrite each other (each builds from the currently-active snapshot). Batch all your installs into a single call.

### Linuxbrew (CLI tools not in the default repository)

Some CLI tools simply aren't packaged for openSUSE, don't fit the Flatpak sandbox model (e.g. language servers meant to be found on `$PATH` by an editor), and aren't worth burning a host OS snapshot on. Linuxbrew fills that gap: it installs into `/home/linuxbrew/.linuxbrew`, entirely outside the read-only root filesystem, so it never touches a transactional-update snapshot and can be updated independently on its own schedule. In this guide it's used specifically to install the language servers Kate needs for Markdown/YAML/Bash editing support (see Section 10).

### Distrobox (for anything else)

Kalpa ships Distrobox pre-installed. It lets you run a full mutable Linux environment (Tumbleweed, Fedora, Ubuntu, etc.) inside a container where you can install anything with `zypper` or `apt`. GUI apps can be exported to the host launcher. This is the recommended way to run tools that don't fit Flatpak or the host OS.

---

## 4. First Boot — Update the System

System updates on Kalpa happen automatically via `transactional-update.timer` (runs daily). To trigger one manually:

```bash
sudo transactional-update
sudo reboot
```

---

## 5. Install Google Chrome, OnlyOffice and Nextcloud (Flatpak)

```bash
flatpak uninstall org.mozilla.firefox
flatpak install flathub com.google.Chrome org.onlyoffice.desktopeditors com.nextcloud.desktopclient.nextcloud
flatpak uninstall --unused
```

- Sign in to Chrome to sync bookmarks and extensions.
- Pin Google Chrome to the taskbar.
- Launch Nextcloud from the application menu, enter server URL and credentials, set sync folder to `~/Nextcloud`.

---

## 6. KDE Plasma Tweaks

### 6a. Set Desktop Theme to Breeze Dark

**System Settings → Quick Settings → Theme → Breeze Dark → Apply.**

or

```bash
lookandfeeltool -a org.kde.breezedark.desktop
```

### 6b. Move Window Buttons to the Right

**System Settings → Global Theme → Window Decorations → Configure Title bar buttons**

The commands below put just the Menu button (`M`) on the left, and Minimize (`I`), Maximize (`A`), and Close (`X`) grouped on the right — dragging the same buttons into that arrangement in the GUI panel achieves the same result.

or

```bash
kwriteconfig6 --file kwinrc --group "org.kde.kdecoration2" --key "ButtonsOnLeft" "M"
kwriteconfig6 --file kwinrc --group "org.kde.kdecoration2" --key "ButtonsOnRight" "IAX"
kwriteconfig6 --file kwinrc --group "org.kde.kdecoration2" --key "ButtonsOnTopLeft" --delete
kwriteconfig6 --file kwinrc --group "org.kde.kdecoration2" --key "ButtonsOnTopRight" --delete
qdbus6 org.kde.KWin /KWin reconfigure
```

### 6c. Set Digital Clock to Use Regional Defaults

1. Right-click on the clock applet
2. Select **Configure Digital Clock**
3. Change **Time Display** to **Uses Regional Defaults**

or

```bash
qdbus6 org.kde.plasmashell /PlasmaShell evaluateScript '
  panels().concat(desktops()).forEach(function(container) {
    container.widgets("org.kde.plasma.digitalclock").forEach(function(w) {
      w.currentConfigGroup = ["Appearance"];
      w.writeConfig("use24hFormat", 1);
      w.reloadConfig();
    });
  });
'
```

### 6d. Enable Num Lock at Login

**System Settings → Input Devices → Keyboard → Hardware tab** → set **NumLock on Plasma Startup** to **Turn On**.

or

```bash
kwriteconfig6 --file kcminputrc --group "Keyboard" --key "NumLock" 0
```

---

## 7. Install Host OS Packages

The packages below cover four things: core CLI tools (git, age, sops, nano, micro), spell-check dictionaries and Markdown preview support for Kate, a few fonts missing from the base install, and everything needed to run local VMs (libvirt/QEMU/virt-manager).

```bash
sudo transactional-update pkg install \
  git-core \
  age \
  sops \
  nano \
  micro-editor \
  myspell-en_CA \
  myspell-en_US \
  myspell-en_GB \
  markdownpart \
  fetchmsttfonts \
  google-caladea-fonts \
  google-noto-serif-sc-fonts \
  virt-manager \
  libvirt \
  libvirt-daemon-qemu \
  qemu-x86 \
  qemu-tools \
  virt-install \
  bridge-utils \
  dnsmasq \
  swtpm \
  virt-viewer
```

Add your user to the `libvirt` group so you can create and manage VMs without needing `sudo` every time:

```bash
sudo usermod -aG libvirt $USER
```

Reboot so the new snapshot (and group membership) takes effect:

```bash
sudo systemctl reboot
```

---

## 8. Restore Keys and Configure Shell Environment

### 8a. Restore SSH Keys

```bash
cp -r /run/media/morgan/TRANSFERS/ssh-backup ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub 2>/dev/null
chmod 644 ~/.ssh/known_hosts 2>/dev/null
chmod 644 ~/.ssh/config 2>/dev/null
```

### 8b. Restore SOPS Age Key

```bash
mkdir -p ~/.config/sops
cp -r /run/media/morgan/TRANSFERS/sops-backup/* ~/.config/sops/
chmod 600 ~/.config/sops/age/keys.txt
```

### 8c. Configure Git

```bash
git config --global user.name "Morgan Smith"
git config --global user.email "13087392+Smithoo4@users.noreply.github.com"
git config --global init.defaultBranch main
```

### 8d. Configure Nano

```bash
cat > ~/.nanorc <<'EOF'
# Include all bundled syntax highlighting
include "/usr/share/nano/*.nanorc"

# Quality-of-life settings
set linenumbers
set mouse
set autoindent
set smarthome
set softwrap
set tabsize 4
set tabstospaces
set constantshow
set historylog
set positionlog

# Show matching bracket
set matchbrackets "(<[{)>]}"
EOF
```

### 8e. Configure Bashrc

```bash
cat >> ~/.bashrc <<'EOF'

# Default editor
export EDITOR=nano
export VISUAL=nano

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
EOF

source ~/.bashrc
```

---

## 9. Install Citrix Workspace App

Citrix Workspace App isn't packaged for openSUSE, so it has to be downloaded and installed manually: no login is required, just go to the [Citrix Workspace App for Linux download page](https://www.citrix.com/downloads/workspace-app/linux/), download the RPM, save it to `~/Downloads/` as `ICAClient-suse-gcc-8-<version>-0.x86_64.rpm`, then install it via `transactional-update` and reboot to activate the snapshot.

```bash
sudo transactional-update -n pkg install ~/Downloads/ICAClient-suse-gcc-8-*.rpm
```

```bash
sudo systemctl reboot
```

---

## 10. Install Linuxbrew

### 10a. Install/Setup Linuxbrew

Installs Homebrew and wires it into Konsole shells only, so it doesn't pollute non-interactive/system scripts.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
echo '' >> ~/.bashrc
echo '# Homebrew: load shellenv only in Konsole sessions' >> ~/.bashrc
echo 'if [ $(basename $(printf "%s" "$(ps -p $(ps -p $$ -o ppid=) -o cmd=)" | cut --delimiter " " --fields 1)) = konsole ] ; then '$'\n''eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"'$'\n''fi'$'\n' >> ~/.bashrc
```

### 10b. Setup Auto Updates

Sets up a weekly systemd user timer that updates, upgrades, and cleans up Homebrew automatically.

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/brew-upgrade.service <<'EOF'
[Unit]
Description=Update Homebrew and upgrade packages
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/home/linuxbrew/.linuxbrew/bin/brew update
ExecStart=/home/linuxbrew/.linuxbrew/bin/brew upgrade
ExecStart=/home/linuxbrew/.linuxbrew/bin/brew cleanup
EOF

cat > ~/.config/systemd/user/brew-upgrade.timer <<'EOF'
[Unit]
Description=Weekly Homebrew upgrade

[Timer]
OnCalendar=weekly
Persistent=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now brew-upgrade.timer
```

### 10c. Install Software

Installs the language servers Kate will use in Section 11.

```bash
brew install yaml-language-server bash-language-server vscode-langservers-extracted marksman
```

---

## 11. Configure Kate

**To be done later.**
