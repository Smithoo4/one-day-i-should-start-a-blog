**Debian 13 GNOME Workstation Setup Guide**  
**Hostname:** `nereus`  
   
 **User:** `morgan`  
   
 **Architecture:** `amd64` (Intel/AMD 64-bit)  
   
 **Last updated:** October 31, 2026  
A setup guide for a workstation running **Debian 13 (Trixie)** and **GNOME**. Native DEB packages are preferred for desktop applications, Flatpak is enabled for additional applications, and Nix with Home Manager manages CLI and development tools. Zed manages editor language support through its built-in support and extensions; Home Manager supplies the Nix formatter.  
**Contents**  
1. [Back Up Current Workstation](#anchor-1 "#anchor-1")  
2. [Install Debian 13 and Configure GNOME at First Login](#anchor-2 "#anchor-2")  
3. [Remove Unwanted Applications and Update Debian](#anchor-3 "#anchor-3")  
4. [Enable Flatpak and Flathub](#anchor-4 "#anchor-4")  
5. [Install Applications](#anchor-5 "#anchor-5")  
6. [Install Citrix Workspace](#anchor-6 "#anchor-6")  
7. [Restore Keys](#anchor-7 "#anchor-7")  
8. [Install Virtualization Packages for Debian 13](#anchor-8 "#anchor-8")  
9. [Set Up the Incus Remote CLI](#anchor-9 "#anchor-9")  
10. [Install and Configure Nix with Home Manager](#anchor-10 "#anchor-10")  
11. [Install and Configure Zed](#anchor-11 "#anchor-11")  
  
---  
**1. Back Up Current Workstation**  
The USB drive path depends on the current distribution:  
- openSUSE, Fedora, and similar distributions: `/run/media/morgan/TRANSFERS/`  
- Debian, Ubuntu, KDE neon, and derivatives: `/media/morgan/TRANSFERS/`  
The commands below assume the current workstation uses the Debian/Ubuntu path. Adjust it if required.  
**1a. Back Up SSH Keys**  
```  
mkdir -p /media/morgan/TRANSFERS/ssh-backup  
 cp -a ~/.ssh/. /media/morgan/TRANSFERS/ssh-backup/  
   
```  
**1b. Back Up SOPS Age Key**  
```  
mkdir -p /media/morgan/TRANSFERS/sops-backup  
 cp -a ~/.config/sops/. /media/morgan/TRANSFERS/sops-backup/  
   
```  
**Security:** These backups contain private keys. Use a secure backup drive and verify the files were copied before erasing the workstation.  
**1c. Confirm Chrome Synchronization**  
Chrome synchronizes bookmarks and extensions through the Google account. Confirm synchronization is enabled and current before reinstalling.  
**1d. Confirm Nextcloud Synchronization**  
Confirm the Nextcloud desktop client has completed synchronizing all required files before reinstalling.  
  
---  
**2. Install Debian 13 and Configure GNOME at First Login**  
**2a. Create the Bootable USB**  
1. Download the **Debian 13 amd64 netinst ISO** from the [Debian Trixie installer page. Choose the current Debian 13 point release, not testing or another major version.](https://www.debian.org/releases/trixie/debian-installer/index "https://www.debian.org/releases/trixie/debian-installer/index")  
2. Download its checksum file and signature from the same image directory. Follow [Debian's image verification instructions before writing the image.](https://www.debian.org/CD/verify "https://www.debian.org/CD/verify")  
3. Write the ISO using the current workstation's ISO writer. On a GNOME workstation, **Disks → select the USB drive → drive menu → Restore Disk Image** can be used. This erases the USB drive.  
4. Boot the target workstation from that USB in **UEFI mode**.  
5. Select **Graphical install**.  
**2b. Language, Network, and User Settings**  
Use these settings as the installer presents them:  
- **Language:** English.  
- **Location:** Canada.  
- **Keyboard:** American English / English (US).  
- **Primary network:** wired Ethernet when available; allow automatic DHCP configuration.  
- **Hostname:** `nereus`.  
- **Domain name:** leave blank unless your existing network requires one.  
- **Root password:** leave both root-password fields blank. Debian then disables direct root login and grants the first user `sudo` access.  
- **Full name:** Morgan Smith.  
- **Username:** `morgan`.  
- **User password:** set a strong password.  
- **Time zone:** Mountain / Edmonton, corresponding to `America/Edmonton`.  
**2c. Optional: Connect to Wi-Fi**  
**During installation, if Ethernet is unavailable:**  
1. Select the wireless adapter as the primary network interface.  
2. Enter or select the wireless network's SSID when prompted.  
3. Select the appropriate security method and enter the WPA/WPA2 passphrase.  
4. Continue after networking succeeds. If the installer reports missing firmware, follow its firmware-loading prompt rather than guessing a driver package. \[^1\]  
**2d. Partitioning and Software Selection**  
**Warning:** The following layout erases the selected system disk. Verify its model and capacity. Disconnect unrelated drives, including the backup USB, before partitioning.  
1. Choose **Guided - use entire disk**.  
2. Select the intended system drive.  
3. Choose **All files in one partition (recommended for new users)**.  
4. Keep the default filesystem and UEFI partition layout. This workstation layout does **not** use encrypted LVM. Be aware that the system disk, including restored private keys, will not be protected by full-disk encryption.  
5. Review the partition summary, select **Finish partitioning and write changes to disk**, and confirm.  
6. Configure a Debian network mirror, using `deb.debian.org` when offered. Leave the HTTP proxy blank unless needed.  
7. At **Software selection**, select:  
  - **Debian desktop environment**.  
  - **GNOME**.  
  - **Standard system utilities**.  
8. Deselect other desktop environments.  
9. Finish installation, allow GRUB installation to the intended system disk when prompted, restart, and remove the installer USB.  
10. Log in as `morgan`. \[^1\]  
**2e. Desktop Configuration**  
***Dark Mode***  
Open **Settings → Appearance** and select **Dark**.  
For GTK 3 applications, also review the per-user GTK settings file. Create its directory, then open the file:  
```  
mkdir -p ~/.config/gtk-3.0  
 nano ~/.config/gtk-3.0/settings.ini  
   
```  
Add the following section if it does not exist. If it already exists, update the preference within that section without duplicating it or deleting other settings:  
```  
[Settings]  
 gtk-application-prefer-dark-theme=true  
   
```  
Save the file and restart the affected applications. This requests a dark variant where the GTK 3 theme supports one; it does not force every application or toolkit into dark mode. Keep GNOME's Appearance setting enabled as well. \[^26\]  
***Minimize and Maximize Buttons***  
Open **Tweaks** from the application grid. In **Tweaks → Windows / Window Titlebars**, enable both **Minimize** and **Maximize** under **Titlebar Buttons**. The page label varies with the Tweaks version. \[^2\]  
***Files: Tree-Style List View***  
1. Open **Files**.  
2. Select **List View** using the toolbar view control.  
3. Open **Files → menu → Preferences**.  
4. Enable **Expandable Folders in List View** (also described as expanding folders).  
5. Expand a folder using the arrow beside its name.  
This provides a tree-style hierarchy within the file list, rather than a separate Dolphin-style folder-tree sidebar. \[^3\]  
***Num Lock***  
Press the physical **Num Lock** key to enable it. Check whether the session's Num Lock state is retained after the reboot in Section 3; this does not set a firmware or login-screen Num Lock policy.  
***Remove Unwanted Dock Favorites***  
Press **Super** to open Activities. Right-click each of these pinned icons and select **Remove from Favorites** / **Unpin**:  
- Firefox / Firefox ESR.  
- Evolution.  
- Help.  
- Text Editor.  
- Calculator.  
In stock GNOME the dock-like favorites bar is the **dash**, visible in Activities. Unpinning an application does not uninstall it. Help can remain installed. \[^4\]  
  
---  
**3. Remove Unwanted Applications and Update Debian**  
**3a. Remove Firefox, Evolution, LibreOffice, and GNOME Tour**  
Debian's Firefox package is named `firefox-esr`:  
```  
sudo apt remove --purge firefox-esr evolution 'libreoffice*' gnome-tour  
 sudo apt autoremove --purge  
   
```  
Review the proposed removals before confirming, especially desktop metapackages and any applications you want to keep. Leave user-profile directories in place unless you deliberately want to delete their settings and local data.  
**3b. Enable the Debian Contrib Component**  
```  
sudo nano /etc/apt/sources.list  
   
```  
Add `contrib` to each active Debian repository line. Preserve the existing mirror, options, suite, and other components. For a typical configuration that already includes `main non-free-firmware`, the result is:  
```  
deb https://deb.debian.org/debian/ trixie main contrib non-free-firmware  
 deb https://deb.debian.org/debian/ trixie-updates main contrib non-free-firmware  
 deb https://security.debian.org/debian-security/ trixie-security main contrib non-free-firmware  
   
```  
These are examples for comparison, not additional entries to append. If `deb-src` entries are enabled, update their components consistently. Comment out an installer `deb cdrom:` entry if you no longer want APT to request the installation media.  
Refresh the package index and resolve any errors before continuing:  
```  
sudo apt update  
   
```  
**3c. Install Updates and Basic Tools**  
```  
sudo apt update  
 sudo apt full-upgrade  
 sudo apt install curl ffmpeg rsync  
   
```  
Review the full-upgrade transaction before confirming, especially any proposed removals.  
**3d. Reboot**  
```  
sudo reboot  
   
```  
After logging back in, open **Settings → System → About** and confirm Debian/GNOME. In Terminal:  
```  
hostnamectl  
 cat /etc/os-release  
 dpkg --print-architecture  
   
```  
Confirm hostname `nereus`, codename `trixie`, and architecture `amd64`. Check that Num Lock is still enabled.  
  
---  
**4. Enable Flatpak and Flathub**  
Install Flatpak and the GNOME Software integration:  
```  
sudo apt install flatpak gnome-software-plugin-flatpak  
 sudo flatpak remote-add --system --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo  
 flatpak --system remotes --columns=name  
   
```  
Confirm `flathub` is listed. Log out and back in so the desktop picks up Flatpak application paths and the Software plugin. Open **Software** to browse applications; check the source when an application is available as both a Debian package and a Flatpak. \[^5\]  
To update installed system-wide Flatpaks later:  
```  
sudo flatpak update --system  
   
```  
  
---  
**5. Install Applications**  
Use Debian packages or repositories intended for Debian.  
**5a. Google Chrome**  
Download and install the Chrome package, then remove the downloaded file:  
```  
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb  
 sudo apt install ./google-chrome-stable_current_amd64.deb  
 rm google-chrome-stable_current_amd64.deb  
   
```  
Launch Chrome, sign in, and confirm bookmarks and extensions synchronize. Set it as the default browser if desired.  
**Pin it:** press **Super**, find **Google Chrome**, right-click it, and select **Add to Favorites** / **Pin to Dash**. \[^4\]  
**5b. ONLYOFFICE Desktop Editors**  
Add the official ONLYOFFICE repository and install the native package:  
***Install the Signing Key***  
```  
sudo install -d -m 0755 /etc/apt/keyrings  
 curl -fsSL https://download.onlyoffice.com/GPG-KEY-ONLYOFFICE -o /tmp/onlyoffice-key.asc  
 gpg --show-keys --fingerprint /tmp/onlyoffice-key.asc  
   
```  
Before importing the key, compare its identity with the publisher's installation documentation. Stop if the download or key inspection fails. The publisher currently identifies the key as `CB2DE8E5`. \[^6\]  
```  
sudo gpg --dearmor --yes -o /etc/apt/keyrings/onlyoffice.gpg /tmp/onlyoffice-key.asc  
 sudo chmod 0644 /etc/apt/keyrings/onlyoffice.gpg  
 rm /tmp/onlyoffice-key.asc  
   
```  
***Add the Repository***  
```  
sudo tee /etc/apt/sources.list.d/onlyoffice.sources > /dev/null <<'SOURCES'  
 Types: deb  
 URIs: https://download.onlyoffice.com/repo/debian  
 Suites: squeeze  
 Components: main  
 Architectures: amd64  
 Signed-By: /etc/apt/keyrings/onlyoffice.gpg  
 SOURCES  
   
```  
```  
sudo apt update  
 sudo apt install onlyoffice-desktopeditors fonts-croscore fonts-crosextra-caladea  
 fc-cache -f  
   
```  
**`squeeze` is intentional.** ONLYOFFICE documents this repository suite for Debian and its derivatives. Do not replace it with `trixie`. The key location and Deb822 format modernize the APT configuration without changing the publisher's repository layout. \[^6\]  
Launch **ONLYOFFICE Desktop Editors** and pin it to the dash using **Add to Favorites** / **Pin to Dash**. \[^4\]  
**5c. Nextcloud Desktop Client and GNOME Files Integration**  
Install the native Nextcloud client and GNOME Files integration from the standard repositories:  
```  
sudo apt install nextcloud-desktop nautilus-nextcloud  
   
```  
Launch Nextcloud from the application menu and configure:  
- **Server URL:** enter the existing Nextcloud server address  
- **Authentication:** sign in using the normal account  
- **Local folder:** `~/Nextcloud`  
- Confirm the initial synchronization completes successfully.  
  
---  
**6. Install Citrix Workspace**  
**Compatibility note:** the Citrix system-requirements page lists Debian 11/12, not Debian 13. Treat Debian 13 as a tested-on-your-workstation configuration rather than claiming vendor support. \[^9\]  
**6a. Download the Package**  
1. Open the [Citrix Workspace app for Linux download page.](https://www.citrix.com/downloads/workspace-app/linux/ "https://www.citrix.com/downloads/workspace-app/linux/")  
2. Download **Debian Full Package (Self-Service Support), x86_64**.  
3. Save it in `/home/morgan/Downloads/` and verify the published checksum for that exact download.  
4. Keep only the intended installer matching the wildcard below in Downloads. \[^9\]  
**6b. Install Citrix Workspace**  
```  
sudo apt install /home/morgan/Downloads/icaclient*_amd64.deb  
   
```  
Review APT's proposed installation and dependency changes. If it proposes removing Citrix because dependencies cannot be satisfied, stop and investigate rather than adding Ubuntu repositories or forcing installation.  
***Note:*** *The installer asks several questions. Answer them as follows:*  
- ***App Protection component:*** *select* ***No*** *for this workstation unless IT requires it.*  
- ***deviceTRUST component:*** *select* ***No***, *unless corporate IT specifically requires it. Install it if required by the access policy.*  
- ***EPA component:*** *select* ***No***. *Endpoint Analysis is only needed if the corporate login page runs a security scan before allowing a connection.*  
Do not disable a required corporate security component to bypass an access check. If a component is required, obtain the supported installation settings from IT.  
**6c. Verify and Clean Up**  
Launch a work session and check sign-in, keyboard input, clipboard transfer in both directions, and any required audio or device access before removing the installer. The published requirements also list X11/X.Org and state that Wayland is not supported; a working GNOME session on Debian 13 should not be presented as a vendor-supported configuration. \[^9\]  
After successful installation:  
```  
rm /home/morgan/Downloads/icaclient*_amd64.deb  
   
```  
  
---  
**7. Restore Keys**  
The commands assume it is mounted at `/media/morgan/TRANSFERS/`.  
**7a. Restore SSH Keys**  
The following copies the backup contents into `.ssh`, avoiding a nested `ssh-backup` directory if `.ssh` already exists. Review existing files first; copying can overwrite matching names.  
```  
mkdir -p ~/.ssh  
 cp -a /media/morgan/TRANSFERS/ssh-backup/. ~/.ssh/  
 chmod 700 ~/.ssh  
 find ~/.ssh -type d -exec chmod 700 {} +  
 find ~/.ssh -type f -exec chmod 600 {} +  
 find ~/.ssh -type f -name '*.pub' -exec chmod 644 {} +  
 ls -la ~/.ssh  
   
```  
Keep private keys private. Do not print their contents into logs or the guide.  
**7b. Restore the SOPS Age Key**  
```  
mkdir -p ~/.config/sops  
 cp -a /media/morgan/TRANSFERS/sops-backup/. ~/.config/sops/  
 chmod 700 ~/.config/sops ~/.config/sops/age  
 chmod 600 ~/.config/sops/age/keys.txt  
 ls -l ~/.config/sops/age/keys.txt  
   
```  
The `sops` and `age` commands are installed through Home Manager in Section 10. Safely eject the backup USB when finished. Keep it secure because it contains private keys.  
  
---  
**8. Install Virtualization Packages for Debian 13**  
This section enables **local KVM/QEMU virtual machines**. Section 9 separately installs the client for the existing remote IncusOS server; local KVM is not required just to manage that server.  
**8a. Check Hardware Virtualization**  
Enable Intel VT-x / AMD-V (SVM) in the workstation firmware if needed. Check CPU flags:  
```  
grep -cE '(vmx|svm)' /proc/cpuinfo  
   
```  
A nonzero result indicates exposed virtualization extensions. If the result is zero, check firmware settings before continuing. \[^10\]  
**8b. Install the Packages**  
Use Debian's explicit x86 QEMU package:  
```  
sudo apt update  
 sudo apt install \  
   qemu-system-x86 \  
   qemu-utils \  
   libvirt-daemon-system \  
   libvirt-clients \  
   virt-manager \  
   virtinst \  
   ovmf \  
   swtpm  
 sudo systemctl enable --now libvirtd  
 sudo usermod -aG libvirt,kvm "$USER"  
 sudo reboot  
   
```  
OVMF supplies UEFI firmware for guests; `swtpm` supplies software TPM support. No manual bridge is required for the normal libvirt NAT workflow. \[^10\]  
**8c. Verify Local Access**  
After logging back in:  
```  
id  
 ls -l /dev/kvm  
 virsh -c qemu:///system list --all  
 virsh -c qemu:///system net-list --all  
   
```  
Open **Virtual Machine Manager** and use the local **QEMU/KVM system** connection. If the default NAT network is inactive, allow Virtual Machine Manager to start it when prompted. An empty VM list is normal on a new workstation. \[^10\]  
  
---  
**9. Set Up the Incus Remote CLI**  
**IncusOS server:** `192.168.0.50:8443`  
   
 **Daily-use remote:** `incusos` (TLS)  
   
 **Bootstrap remote:** `incusos-sso` (OIDC/SSO)  
   
 **New workstation certificate label:** `nereus-cli`  
Install the **client tools**, not a local Incus server. Do not run `incus admin init`.  
**9a. Confirm Browser SSO Access**  
1. Open the [IncusOS web interface in Chrome.](https://192.168.0.50:8443 "https://192.168.0.50:8443")  
2. If the certificate is untrusted, verify the server identity through an already trusted device or the server console before accepting a site-specific exception.  
3. Sign in using SSO and confirm you can view the server's instances.  
No `client.pfx` import into Chrome is needed for this SSO workflow.  
**9b. Add the Zabbly Stable Repository**  
```  
sudo install -d -m 0755 /etc/apt/keyrings  
 curl -fsSL https://pkgs.zabbly.com/key.asc -o /tmp/zabbly-key.asc  
 gpg --show-keys --fingerprint /tmp/zabbly-key.asc  
   
```  
Confirm that the fingerprint matches Zabbly's published value before installing the key: \[^11\]  
```  
4EFC 5906 96CB 15B8 7C73 A3AD 82CC 8797 C838 DCFD  
   
```  
```  
sudo install -m 0644 /tmp/zabbly-key.asc /etc/apt/keyrings/zabbly.asc  
 rm /tmp/zabbly-key.asc  
   
```  
```  
sudo tee /etc/apt/sources.list.d/zabbly-incus-stable.sources > /dev/null <<'SOURCES'  
 Enabled: yes  
 Types: deb  
 URIs: https://pkgs.zabbly.com/incus/stable  
 Suites: trixie  
 Components: main  
 Architectures: amd64  
 Signed-By: /etc/apt/keyrings/zabbly.asc  
 SOURCES  
   
```  
```  
sudo apt update  
 sudo apt install --no-install-recommends incus-client incus-extra  
   
```  
Use only one Zabbly Incus release channel. If APT proposes `incus` or `incus-base`, stop and inspect the dependency transaction instead of accepting a local server installation. \[^11\]  
```  
command -v incus  
 command -v incus-migrate  
 incus --version  
 incus-migrate --help  
   
```  
This installs and verifies `incus-migrate`; it does not perform a migration.  
**9c. Bootstrap SSO and Register a New TLS Certificate**  
Run these commands as `morgan`, **without sudo**:  
```  
incus remote add incusos-sso https://192.168.0.50:8443 --auth-type=oidc  
 incus list incusos-sso:  
   
```  
Verify the server fingerprint before accepting it. Follow the browser sign-in prompt and confirm the device code if shown.  
Create a trust token labelled for this workstation:  
```  
incus config trust add incusos-sso:nereus-cli  
   
```  
Treat the printed token as a credential. Use it promptly and do not paste it into messages. Then add the daily-use remote:  
```  
incus remote add incusos https://192.168.0.50:8443 --auth-type=tls  
   
```  
Verify the fingerprint, then paste the trust token when prompted. Incus creates its own client keypair under `~/.config/incus/`; these credentials are separate from the restored SSH and SOPS keys. \[^12\]  
**9d. Verify Access and Select the Default Remote**  
```  
incus remote list  
 incus info incusos:  
 incus list incusos:  
 incus storage list incusos:  
 incus network list incusos:  
 incus config trust list incusos:  
 incus remote switch incusos  
 incus remote get-default  
 incus list  
   
```  
Confirm `incusos` uses TLS and the server trust list contains `nereus-cli`. An empty instance list is valid if the selected project has no instances. The alias is a local shortcut, not a server rename.  
After TLS access works:  
```  
incus remote remove incusos-sso  
   
```  
Removing the temporary remote does not remove the trusted TLS certificate or browser SSO access. Before future create/delete/configuration operations, confirm the intended remote; use an explicit `incusos:` prefix to avoid ambiguity.  
**9e. Review and Revoke Credentials When Needed**  
Keep a working second access method, normally browser SSO, before revoking any credential.  
```  
incus config trust list incusos:  
 incus config trust list-tokens incusos:  
   
```  
To inspect a certificate, replace `FINGERPRINT` with the actual value:  
```  
incus config trust show incusos:FINGERPRINT  
   
```  
Only if deliberately revoking an unused certificate:  
```  
incus config trust remove incusos:FINGERPRINT  
   
```  
To revoke an unused token, replace `TOKEN_NAME` with its actual name:  
```  
incus config trust revoke-token incusos:TOKEN_NAME  
   
```  
Do not revoke `nereus-cli`. The trust list does not establish last-use dates. SSO access is managed separately through the identity provider. \[^12\]  
  
---  
**10. Install and Configure Nix with Home Manager**  
This section configures Determinate Nix, root-level maintenance timers, a Home Manager flake, weekly Home Manager updates, user garbage collection, and GNOME access to Nix-installed tools.  
**Update policy:** The flake tracks `nixos-unstable` and Home Manager `master`. Weekly updates apply changes automatically and may introduce regressions. Keep the configuration and `flake.lock` backed up. User timers normally run while the user manager is active; `Persistent=true` allows missed runs to be triggered when it becomes active again.  
**10a. Install Nix**  
Install Determinate Nix with automatic store optimization. \[^13\]  
```  
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix \  
   | sh -s -- install --determinate \  
       --extra-conf "auto-optimise-store = true"  
   
```  
Load the Nix environment and verify the installation:  
```  
. /nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh  
 nix --version  
   
```  
**10b. Configure Determinate Nix Auto-Upgrade**  
First verify the daemon path and command:  
```  
command -v determinate-nixd  
 sudo /usr/local/bin/determinate-nixd upgrade --help  
   
```  
If the binary is elsewhere, use its actual absolute path in `ExecStart`. If it is absent, stop and confirm that Determinate Nix, rather than upstream Nix, was installed.  
Create the service:  
```  
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
```  
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
```  
sudo systemctl daemon-reload  
 sudo systemctl start determinate-nix-upgrade.service  
 sudo systemctl status determinate-nix-upgrade.service  
 sudo systemctl enable --now determinate-nix-upgrade.timer  
 sudo systemctl status determinate-nix-upgrade.timer  
   
```  
If the service cannot find `determinate-nixd`, locate it and update `ExecStart`:  
```  
command -v determinate-nixd  
   
```  
**10c. Configure Root-Level Nix Garbage Collection**  
Create the service:  
```  
sudo tee /etc/systemd/system/nix-gc.service > /dev/null <<'EOF'  
 [Unit]  
 Description=Nix Root Garbage Collection  
   
 [Service]  
 Type=oneshot  
 ExecStart=/nix/var/nix/profiles/default/bin/nix-collect-garbage --delete-older-than 30d  
 EOF  
   
```  
Create the timer:  
```  
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
```  
sudo systemctl daemon-reload  
 sudo systemctl start nix-gc.service  
 sudo systemctl status nix-gc.service  
 sudo systemctl enable --now nix-gc.timer  
 sudo systemctl status nix-gc.timer  
   
```  
**10d. Create the Home Manager Flake**  
```  
mkdir -p /home/morgan/.config/home-manager  
   
```  
Create `/home/morgan/.config/home-manager/flake.nix`:  
```  
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
**10e. Create the Home Manager Configuration**  
Create `/home/morgan/.config/home-manager/home.nix`:  
```  
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
         ${pkgs.systemd}/bin/systemd-run --user --collect --wait --unit=hm-auto-update-runner ${pkgs.writeShellScript "hm-auto-update-exec" ''  
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
     nil  
     nixfmt  
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
       # Dump Git-tracked files only; exclude LICENSE and repo_dump.txt.  
       repodump() {  
         git ls-files -z \  
           | sort -z \  
           | while IFS= read -r -d $'\0' f; do  
               [[ "$f" == "repo_dump.txt" || "$f" == "LICENSE" ]] && continue  
               [[ -f "$f" ]] || continue  
               printf '===== %s =====\n' "$f"  
               cat -- "$f"  
               printf '\n'  
             done > repo_dump.txt \  
           && echo "[OK] repo_dump.txt created"  
       }  
     '';  
   };  
 }  
 EOF  
   
```  
***State version:*** *`26.05` is retained from the supplied configuration. Keep it fixed after the first successful activation; it is not the version of the installed packages. \[^14\]*  
**10f. Activate Home Manager**  
Bootstrap Home Manager and back up conflicting unmanaged files:  
```  
nix run home-manager/master -- switch \  
   --flake /home/morgan/.config/home-manager#morgan \  
   -b backup  
   
```  
Future manual rebuilds use:  
```  
home-manager switch --flake /home/morgan/.config/home-manager#morgan  
   
```  
**10g. Make Nix Tools Available to GNOME Applications**  
GUI applications launched from GNOME do not read `.bashrc`. Add the Nix profile to the systemd user environment:  
```  
mkdir -p /home/morgan/.config/environment.d  
 cat > /home/morgan/.config/environment.d/10-nix.conf <<'EOF'  
 PATH=$HOME/.nix-profile/bin:$PATH  
 EOF  
   
```  
Log out and back in so GNOME applications such as Zed can find Nix-installed language servers.  
**10h. Verify Home Manager Automation**  
```  
systemctl --user start home-manager-auto-update.service  
 systemctl --user status home-manager-auto-update.service  
 journalctl --user -u hm-auto-update-runner.service -n 30 --no-pager  
 systemctl --user list-timers --all | grep home-manager  
   
```  
  
---  
**11. Install and Configure Zed**  
Use Zed's built-in language support and extension gallery instead of installing a parallel collection of language servers through Home Manager. Retain Nix itself from Section 10, `nil` for the Nix language server, and `nixfmt` for formatting Nix files. These Nix-specific tools remain supplied by Home Manager; Zed manages the other editor language servers. \[^17\] \[^20\]  
**11a. Install Zed**  
Use Zed's official stable installer as `morgan`, not root. Download it first so it can be reviewed:  
```  
curl -fL https://zed.dev/install.sh -o /tmp/zed-install.sh  
 less /tmp/zed-install.sh  
 sh /tmp/zed-install.sh  
   
```  
Launch **Zed** and pin it to the dash using **Add to Favorites** / **Pin to Dash**. \[^4\]  
Zed requires working Vulkan graphics support. If it cannot open a window, follow [Zed's Linux graphics troubleshooting before changing graphics drivers or desktop-session settings. \[^15\]](#anchor-12 "#anchor-12")  
**11b. Install Extensions and Configure Language Support**  
Open the Command Palette with `Ctrl+Shift+P` and type `zed: extensions`, then search for and install each of:  
- **Harper** ; grammar and spell checking (downloads `harper-ls` automatically, no separate install needed)  
- **Marksman:** Markdown language-server support for link validation and heading navigation. \[^29\]  
- **Nix** ; syntax highlighting and LSP support for `.nix` files  
- **HTML:** language support for HTML files. \[^23\]  
- **Git Firefly:** syntax highlighting for Git configuration, ignore, attributes, and rebase files. Git blame and history features belong to Zed's built-in Git integration, not this extension. \[^16\] \[^21\]  
**11c. Configure Settings**  
Open the Command Palette with **Ctrl+Shift+P** and run **zed: open settings file** to edit `~/.config/zed/settings.json`. On a fresh installation, use the configuration below. If settings already exist, merge these entries instead of replacing the entire file. \[^18\]  
```  
{  
   "cli_default_open_behavior": "existing_window",  
   "ui_font_size": 16,  
   "buffer_font_size": 15,  
   "theme": {  
     "mode": "system",  
     "light": "One Light",  
     "dark": "One Dark",  
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
       "language_servers": ["marksman", "harper-ls"],  
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
***Note:*** *Zed uses JSONC (JSON with Comments), so trailing commas are fine and comments are supported.*  
***Language servers:*** *Markdown explicitly enables both Marksman and Harper. A list containing only Harper would exclude Marksman. For Nix, `"!nixd"` disables `nixd` while `nil` remains enabled. \[^17\] \[^20\]*  
**11d. Verify the Editor Setup**  
1. Open a Markdown file and confirm both link/navigation support and Harper diagnostics are available.  
2. Use **Ctrl+.** on a Harper diagnostic to review suggested corrections or add a word to the dictionary. \[^19\]  
3. Open a `.nix` file and run **editor: format** from the Command Palette. Confirm `nil` and `nixfmt` are found after the logout/login in Section 10g.  
4. If a tool cannot be found, check `command -v nil` and `command -v nixfmt` in a new terminal, then inspect the editor's language-server logs.  
The Harper setting retains Canadian English. Zed settings use JSONC, so comments and trailing commas are supported.  
  
---  
**External References**  
\[^1\]: [Debian 13 installation guide and](https://www.debian.org/releases/trixie/amd64/ "https://www.debian.org/releases/trixie/amd64/") [installer component details.  
   
 \[^2\]:](https://www.debian.org/releases/trixie/amd64/ch06s03.en.html "https://www.debian.org/releases/trixie/amd64/ch06s03.en.html") [GNOME minimize and maximize buttons using Tweaks.  
   
 \[^3\]:](https://itsfoss.com/gnome-minimize-button/ "https://itsfoss.com/gnome-minimize-button/") [GNOME Files display preferences and expandable folders.  
   
 \[^4\]:](https://teams.pages.gitlab.gnome.org/Websites/help.gnome.org/gnome-help/nautilus-display.html "https://teams.pages.gitlab.gnome.org/Websites/help.gnome.org/gnome-help/nautilus-display.html") [GNOME dash favorites.  
   
 \[^5\]:](https://help.gnome.org/gnome-help/shell-apps-favorites.html "https://help.gnome.org/gnome-help/shell-apps-favorites.html") [Flatpak and Flathub setup.  
   
 \[^6\]:](https://flatpak.org/setup/Debian "https://flatpak.org/setup/Debian") [ONLYOFFICE Debian installation and repository suite and](https://helpcenter.onlyoffice.com/desktop/installation/desktop-install-ubuntu.aspx "https://helpcenter.onlyoffice.com/desktop/installation/desktop-install-ubuntu.aspx") [publisher signing key.  
   
 \[^7\]:](https://download.onlyoffice.com/GPG-KEY-ONLYOFFICE "https://download.onlyoffice.com/GPG-KEY-ONLYOFFICE") [GNOME Disks.  
   
 \[^8\]:](https://apps.gnome.org/DiskUtility/ "https://apps.gnome.org/DiskUtility/") [Debian GStreamer libav package,](https://packages.debian.org/trixie/amd64/gstreamer1.0-libav "https://packages.debian.org/trixie/amd64/gstreamer1.0-libav") [Debian Microsoft font installer, and](https://tracker.debian.org/pkg/ttf-mscorefonts-installer "https://tracker.debian.org/pkg/ttf-mscorefonts-installer") [Debian fonts and contrib configuration.  
   
 \[^9\]:](https://linuxcapable.com/how-to-install-microsoft-fonts-on-debian-linux/ "https://linuxcapable.com/how-to-install-microsoft-fonts-on-debian-linux/") [Citrix Linux downloads and](https://www.citrix.com/downloads/workspace-app/linux/workspace-app-for-linux-latest.html "https://www.citrix.com/downloads/workspace-app/linux/workspace-app-for-linux-latest.html") [system requirements.  
   
 \[^10\]:](https://docs.citrix.com/us/en/citrix-workspace-app-for-linux/system-requirements.html "https://docs.citrix.com/us/en/citrix-workspace-app-for-linux/system-requirements.html") [Debian 13 KVM installation and](https://computingforgeeks.com/how-to-install-kvm-virtualization-on-debian/ "https://computingforgeeks.com/how-to-install-kvm-virtualization-on-debian/") [libvirt QEMU system/session connections.  
   
 \[^11\]:](https://libvirt.org/drvqemu.html "https://libvirt.org/drvqemu.html") [Zabbly Incus repository.  
   
 \[^12\]:](https://github.com/zabbly/incus "https://github.com/zabbly/incus") [Incus remote API authentication.  
   
 \[^13\]:](https://linuxcontainers.org/incus/docs/main/authentication/ "https://linuxcontainers.org/incus/docs/main/authentication/") [Determinate Linux installation and](https://github.com/DeterminateSystems/determinate "https://github.com/DeterminateSystems/determinate") [Determinate Nixd.  
   
 \[^14\]:](https://docs.determinate.systems/determinate-nix/determinate-nixd/ "https://docs.determinate.systems/determinate-nix/determinate-nixd/") [Home Manager manual and release notes.  
   
 \[^15\]:](https://nix-community.github.io/home-manager/ "https://nix-community.github.io/home-manager/") [Zed Linux installation.  
   
 \[^16\]:](https://zed.dev/docs/linux "https://zed.dev/docs/linux") [Zed Git integration and inline blame.  
   
 \[^17\]:](https://zed.dev/docs/git "https://zed.dev/docs/git") [Zed language-server configuration.  
   
 \[^18\]:](https://zed.dev/docs/configuring-languages "https://zed.dev/docs/configuring-languages") [Zed action reference.  
   
 \[^19\]:](https://zed.dev/docs/all-actions "https://zed.dev/docs/all-actions") [Harper Zed extension installation, settings, and dictionaries and](https://github.com/zed-extensions/harper "https://github.com/zed-extensions/harper") [extension listing.  
   
 \[^20\]:](https://zed.dev/extensions/harper "https://zed.dev/extensions/harper") [Nix Zed extension configuration and formatting and](https://github.com/zed-extensions/nix "https://github.com/zed-extensions/nix") [extension listing.  
   
 \[^21\]:](https://zed.dev/extensions/nix "https://zed.dev/extensions/nix") [Git Firefly extension and](https://zed.dev/extensions/git-firefly "https://zed.dev/extensions/git-firefly") [supported file types.  
   
 \[^22\]:](https://github.com/zed-extensions/git_firefly "https://github.com/zed-extensions/git_firefly") [Nixpkgs formatter alias and](https://github.com/NixOS/nixpkgs/blob/master/pkgs/top-level/aliases.nix "https://github.com/NixOS/nixpkgs/blob/master/pkgs/top-level/aliases.nix") [official Nixfmt documentation.  
   
 \[^23\]:](https://github.com/NixOS/nixfmt "https://github.com/NixOS/nixfmt") [Zed extension gallery.  
   
 \[^24\]:](https://zed.dev/extensions "https://zed.dev/extensions") [Harper privacy and supported English dialects.  
   
 \[^25\]:](https://writewithharper.com/ "https://writewithharper.com/") [Debian GNOME Tweaks package.  
   
 \[^26\]:](https://packages.debian.org/trixie/gnome-tweaks "https://packages.debian.org/trixie/gnome-tweaks") [GTK 3 settings files and](https://docs.gtk.org/gtk3/class.Settings.html "https://docs.gtk.org/gtk3/class.Settings.html") [dark-theme preference.  
   
 \[^27\]:](https://docs.gtk.org/gtk3/property.Settings.gtk-application-prefer-dark-theme.html "https://docs.gtk.org/gtk3/property.Settings.gtk-application-prefer-dark-theme.html") [Debian Reference: APT source formats and archive components.  
   
 \[^28\]:](https://www.debian.org/doc/manuals/debian-reference/ch02.en.html "https://www.debian.org/doc/manuals/debian-reference/ch02.en.html") [Nix garbage collection and profile search locations.  
   
 \[^29\]:](https://nix.dev/manual/nix/latest/command-ref/nix-collect-garbage.html "https://nix.dev/manual/nix/latest/command-ref/nix-collect-garbage.html") [Marksman Zed extension.](https://zed.dev/extensions/marksman "https://zed.dev/extensions/marksman")  
