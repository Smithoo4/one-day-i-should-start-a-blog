# NixOS Home Server Installation with Flakes, Disko, sops-nix, and Home Manager

A practical guide to installing a minimal, headless NixOS home server using
[Flakes](https://nixos-and-flakes.thiscute.world) for reproducible configuration,
[Disko](https://github.com/nix-community/disko) for declarative disk partitioning,
[sops-nix](https://github.com/Mic92/sops-nix) for encrypted secret management,
and [Home Manager](https://github.com/nix-community/home-manager) for declarative
user environment configuration.

By the end of this guide you will have a fully declarative NixOS server that
you can rebuild, version-control, and redeploy from a single `flake.nix`, with
secrets — such as hashed user passwords and SSH private keys — encrypted at rest
in your repository and decrypted at boot time using [age](https://github.com/FiloSottile/age)
keys, user packages and dotfiles managed declaratively alongside the system
configuration, and the configuration itself pushed to a private remote Git
repository over a declaratively managed SSH key.

---

## Assumptions

- The target machine has already been booted into the **NixOS minimal ISO**.
- **UEFI** firmware (no legacy BIOS support needed).
- Single disk, **ext4** root, **zram** for compressed memory swap (no disk swap partition).
- Network address assigned via **DHCP**.
- You have a second machine (your workstation) to SSH in from.
- Architecture: `x86_64-linux`.
- The target machine has at least **8 GB of RAM**. Builds will likely fail with less.
- The following values are used throughout this guide and should be substituted
  with your own where appropriate:

  | Item | Value used in this guide |
  |---|---|
  | Hostname | `oneohm` |
  | Admin user | `smithoo4` |
  | Timezone | `America/Edmonton` (Calgary, Alberta, Canada) |
  | Locale | `en_CA.UTF-8` |
  | SSH public key | `ssh-ed25519 AAAA...` (replace with your own) |
  | NixOS channel | `nixos-25.11` |

---

## Table of Contents

1. [Prepare the Live Environment](#1-prepare-the-live-environment)
2. [SSH In From Your Workstation](#2-ssh-in-from-your-workstation)
3. [Identify Your Disk](#3-identify-your-disk)
4. [Create the Configuration Directory](#4-create-the-configuration-directory)
5. [Generate Hardware Configuration](#5-generate-hardware-configuration)
6. [Write the Configuration Files](#6-write-the-configuration-files)
7. [Generate Age Keys and Encrypt Secrets](#7-generate-age-keys-and-encrypt-secrets)
8. [Run the Installation](#8-run-the-installation)
9. [Copy the Configuration to the Installed System](#9-copy-the-configuration-to-the-installed-system)
10. [First Boot](#10-first-boot)
11. [Ongoing Maintenance](#11-ongoing-maintenance)

---

## 1. Prepare the Live Environment

You are at the console of the booted NixOS installer. Everything in this
section is typed directly at that console. After Step 2, all remaining work
happens over SSH from your workstation.

### 1.1 Verify network connectivity

```bash
ping -c 3 bing.com
```

### 1.2 Set a temporary password for the installer session

The live environment logs in as `nixos` with no password. SSH requires one,
so set a temporary password now:

```bash
passwd nixos
```

Enter and confirm any password. This is only used during installation.

### 1.3 Confirm SSH is running

`sshd` is enabled by default in the NixOS live environment. Confirm it is
running before attempting to connect:

```bash
sudo systemctl status sshd
```

### 1.4 Note your IP address

```bash
ip addr
```

Note the `inet` address on your network interface (e.g. `ens3`, `eth0`,
`enp1s0`) — you will need it to SSH in from your workstation.

---

## 2. SSH In From Your Workstation

From your **workstation**, connect to the installer:

```bash
ssh -o StrictHostKeyChecking=accept-new nixos@<installer-ip>
```

Enter the password you set above.

> [!NOTE]
> All remaining commands are run from this SSH session as the `nixos` user,
> using `sudo` where root privileges are needed.

---

## 3. Identify Your Disk

List block devices to find your target disk:

```bash
lsblk -o NAME,SIZE,TYPE,MODEL
```

Example output on a VM (KVM/QEMU with a virtio disk):

```
NAME   SIZE TYPE MODEL
vda     40G disk
sr0    1.1G rom  QEMU DVD-ROM
```

Example output on bare metal (SATA):

```
NAME    SIZE TYPE MODEL
sda   476.9G disk Samsung SSD 870
sr0     1.1G rom
```

Example output on bare metal (NVMe):

```
NAME        SIZE TYPE MODEL
nvme0n1   476.9G disk Samsung SSD 980 PRO
sr0         1.1G rom
```

> [!WARNING]
> Confirm you are targeting the correct disk. The next steps will **erase all
> data on it**.

### Set the DISK variable

**On bare metal**, use the disk's stable persistent identifier from
`/dev/disk/by-id/`. These identifiers are derived from the disk's hardware
serial number and remain consistent across reboots, unlike kernel device names
such as `/dev/sda` which can change depending on the order hardware is
detected.

List the available identifiers:

```bash
ls -la /dev/disk/by-id/
```

Example output (SATA):

```
lrwxrwxrwx ... ata-Samsung_SSD_870_QVO_1TB_S5VVNX0R123456 -> ../../sda
lrwxrwxrwx ... ata-Samsung_SSD_870_QVO_1TB_S5VVNX0R123456-part1 -> ../../sda1
```

Example output (NVMe):

```
lrwxrwxrwx ... nvme-Samsung_SSD_980_PRO_1TB_S5GXNX0R123456 -> ../../nvme0n1
lrwxrwxrwx ... nvme-Samsung_SSD_980_PRO_1TB_S5GXNX0R123456-part1 -> ../../nvme0n1p1
```

Select the entry that points to the whole disk (no `-part` suffix) and set it
as the `DISK` variable:

```bash
DISK=/dev/disk/by-id/ata-Samsung_SSD_870_QVO_1TB_S5VVNX0R123456
echo $DISK
```

> [!NOTE]
> **VMs** with virtio disks typically do not have `/dev/disk/by-id/` entries.
> On a VM, set `DISK` to the kernel device name directly instead:
> ```bash
> DISK=/dev/vda
> echo $DISK
> ```

---

## 4. Create the Configuration Directory

Create the directory structure that will hold your NixOS configuration and
secrets:

```bash
mkdir -p ~/nixos-config/hosts/oneohm
mkdir -p ~/nixos-config/secrets
cd ~/nixos-config
```

All configuration files are written here. The final layout will be:

```
~/nixos-config/
├── .sops.yaml
├── flake.nix
├── flake.lock
├── secrets/
│   ├── secrets.yaml
│   └── smithoo4_github_ed25519.pub
└── hosts/
    └── oneohm/
        ├── default.nix
        ├── configuration.nix
        ├── disko.nix
        ├── home.nix
        └── hardware-configuration.nix
```

---

## 5. Generate Hardware Configuration

Generate the hardware-specific configuration. The `--no-filesystems` flag tells
the generator to skip `fileSystems` entries — Disko supplies those instead
through its NixOS module.

```bash
sudo nixos-generate-config --no-filesystems --root /mnt
```

> [!NOTE]
> `/mnt` is empty at this point — the disk has not been formatted yet. That is
> intentional. `nixos-generate-config` detects hardware by reading the live
> system, not by inspecting `/mnt`.

Copy the generated file into the config directory:

```bash
cp /mnt/etc/nixos/hardware-configuration.nix hosts/oneohm/hardware-configuration.nix
```

---

## 6. Write the Configuration Files

All five files below are written from inside `~/nixos-config`.

### disko.nix

`hosts/oneohm/disko.nix`
```nix
{ ... }:
{
  disko.devices = {
    disk = {
      main = {
        device = "/dev/disk/by-id/REPLACE-ME";
        type = "disk";
        content = {
          type = "gpt";
          partitions = {
            ESP = {
              type = "EF00";
              size = "512M";
              content = {
                type = "filesystem";
                format = "vfat";
                mountpoint = "/boot";
                mountOptions = [ "umask=0077" ];
              };
            };
            root = {
              size = "100%";
              content = {
                type = "filesystem";
                format = "ext4";
                mountpoint = "/";
                mountOptions = [ "defaults" "noatime" ];
              };
            };
          };
        };
      };
    };
  };
}
```

After creating the file, replace the `REPLACE-ME` placeholder with the actual
value of `$DISK`:

```bash
sed -i "s|/dev/disk/by-id/REPLACE-ME|$DISK|g" hosts/oneohm/disko.nix
```

Verify the replacement was applied correctly:

```bash
grep "device" hosts/oneohm/disko.nix
```

Expected output (bare metal example):

```
        device = "/dev/disk/by-id/ata-Samsung_SSD_870_QVO_1TB_S5VVNX0R123456";
```

Expected output (VM example):

```
        device = "/dev/vda";
```

> [!WARNING]
> Do not proceed if the output still shows `REPLACE-ME`. Re-run the `sed`
> command and check that `$DISK` is set correctly with `echo $DISK`.

### configuration.nix

`hosts/oneohm/configuration.nix`
```nix
{ pkgs, config, ... }:
{
  # Bootloader
  boot.loader.systemd-boot.enable = true;
  boot.loader.systemd-boot.configurationLimit = 10;
  boot.loader.efi.canTouchEfiVariables = true;

  # zram
  zramSwap.enable = true;

  # Hostname
  networking.hostName = "oneohm";

  # Locale & time
  time.timeZone = "America/Edmonton";
  i18n.defaultLocale = "en_CA.UTF-8";

  # Enable the Flakes feature and the accompanying new nix command-line tool
  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  # sops-nix
  sops.defaultSopsFile = ../../secrets/secrets.yaml;
  sops.defaultSopsFormat = "yaml";
  sops.age.keyFile = "/var/lib/sops-nix/key.txt";

  # Secrets
  sops.secrets.smithoo4-password = {
    neededForUsers = true;
  };

  sops.secrets.smithoo4-ssh-key = {
    owner = "smithoo4";
    mode = "0600";
  };

  # Users
  users.mutableUsers = false;
  users.users.smithoo4 = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    hashedPasswordFile = config.sops.secrets.smithoo4-password.path;
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMpbgg2fUmjFNcQ/ByytJKnYZYpl8kOcocbK8vAl/8Yq smithoo4@fedora"
    ];
  };

  # Secure OpenSSH (key-only, no root login)
  services.openssh = {
    enable = true;
    openFirewall = true;
    settings = {
      PasswordAuthentication = false;
      KbdInteractiveAuthentication = false;
      PermitRootLogin = "no";
    };
  };

  # Packages
  environment.systemPackages = with pkgs; [
    age
    git
    mkpasswd
    sops
  ];

  # Perform garbage collection weekly
  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 1w";
  };

  # Optimize storage
  nix.settings.auto-optimise-store = true;

  # Set once at install time. Do NOT change it after the first boot.
  system.stateVersion = "25.11";
}
```

A few things to note in this configuration compared to a basic install:

- `{ pkgs, config, ... }:` — `config` is added to the function arguments so
  that `config.sops.secrets.smithoo4-password.path` can be referenced inside the
  same file.
- `sops.defaultSopsFile` points to `secrets/secrets.yaml` at the root of your
  configuration directory.
- `sops.age.keyFile` tells sops-nix where to find the server's age private key
  at runtime. This key is placed on the installed system before the first boot
  in [Step 8](#8-run-the-installation).
- `users.mutableUsers = false` makes the user table fully declarative. Passwords
  cannot be changed with `passwd` — they are always sourced from the decrypted
  secret. Remove this if you prefer to allow out-of-band password changes.

### default.nix

`hosts/oneohm/default.nix`
```nix
{ ... }:
{
  imports = [
    ./configuration.nix
    ./hardware-configuration.nix
    ./disko.nix
    ./home.nix
  ];
}
```

### home.nix

`hosts/oneohm/home.nix`
```nix
{ pkgs, ... }:
{
  home-manager = {

    useGlobalPkgs = true;
    useUserPackages = true;

    users.smithoo4 = {

      # Packages installed to the user profile.
      home.packages = with pkgs; [
        htop
        tree
        wget
      ];

      # Git
      programs.git = {
        enable = true;
        settings = {
          user = {
            name = "smithoo4";
            email = "13087392+Smithoo4@users.noreply.github.com";
          };
          init = {
            defaultBranch = "main";
          };
        };
      };

      # SSH client — GitHub identity
      programs.ssh = {
        enable = true;
        matchBlocks = {
          "github.com" = {
            hostname = "github.com";
            user = "git";
            identityFile = "/run/secrets/smithoo4-ssh-key";
          };
        };
      };

      # GitHub SSH public key — placed at ~/.ssh/ for easy reference
      home.file.".ssh/smithoo4_github_ed25519.pub" = {
        source = ../../secrets/smithoo4_github_ed25519.pub;
      };

      # Required
      home.stateVersion = "25.11";
    };
  };
}
```

A few things to note about this file:

- `useGlobalPkgs = true` — Home Manager uses the same `pkgs` instance as
  NixOS rather than its own, keeping the package set consistent and avoiding
  duplicate downloads.
- `useUserPackages = true` — packages declared in `home.packages` are
  installed into the user's profile rather than into a separate Home Manager
  profile, making them behave like any other user package.
- `programs.git` configures git declaratively. With `useUserPackages = true`
  Home Manager will install git into smithoo4's profile; the `git` entry in
  `configuration.nix` systemPackages remains for system-level operations such
  as `nixos-rebuild` and flake fetching, which run as root and do not have
  access to the user profile.
- `programs.ssh` — Home Manager writes `~/.ssh/config` declaratively. The
  `github.com` match block tells the SSH client to always use
  `/run/secrets/smithoo4-ssh-key` as the identity file when connecting to
  GitHub. That path is where sops-nix decrypts and places the private key at
  boot, owned by `smithoo4` with mode `0600`.
- `home.file.".ssh/smithoo4_github_ed25519.pub"` — places the public key at
  `~/.ssh/smithoo4_github_ed25519.pub`. The `source` path
  `../../secrets/smithoo4_github_ed25519.pub` is resolved relative to
  `hosts/oneohm/home.nix`, which puts it at `secrets/smithoo4_github_ed25519.pub`
  in the repository root. The file is copied into the Nix store at evaluation
  time. The public key is not a secret and safe to store as a plain file.
- `home.stateVersion` is independent of `system.stateVersion`. Set it once
  at install time and do not change it.

### flake.nix

`flake.nix`
```nix
{
  description = "oneohm — NixOS home server";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.11";

    disko = {
      url = "github:nix-community/disko/latest";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    sops-nix = {
      url = "github:Mic92/sops-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    home-manager = {
      url = "github:nix-community/home-manager/release-25.11";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, ... }@inputs: {
    nixosConfigurations.oneohm = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";

      specialArgs = { inherit inputs self; };

      modules = [
        inputs.disko.nixosModules.disko
        inputs.sops-nix.nixosModules.sops
        inputs.home-manager.nixosModules.home-manager
        ./hosts/oneohm
      ];
    };
  };
}
```

A few things to note about this flake structure:

- `{ self, nixpkgs, ... }@inputs` — the `@inputs` pattern binds the entire
  attribute set to the name `inputs`, making all flake inputs accessible under
  a single variable. Individual inputs such as `disko`, `sops-nix`, and
  `home-manager` no longer need to be destructured in the function signature;
  they are accessed as `inputs.disko`, `inputs.sops-nix`, and
  `inputs.home-manager` instead. As the number of inputs grows this keeps the
  outputs signature clean.
- `specialArgs = { inherit inputs self; }` — passes `inputs` and `self` into
  the NixOS module system as extra arguments. Any module that declares `inputs`
  or `self` in its function arguments will receive them automatically. Neither
  is used in the host modules written in this guide, but having them available
  means you never need to revisit `flake.nix` just to thread a new input
  through to a module that needs it.
- `inputs.home-manager.nixosModules.home-manager` — wires Home Manager into
  the NixOS module system as a NixOS module. This is what makes the
  `home-manager.*` options available in `home.nix`.
- `home-manager.url = "github:nix-community/home-manager/release-25.11"` —
  the `release-25.11` branch is pinned to match the NixOS channel. Always
  keep the Home Manager release in sync with `nixpkgs` to avoid incompatibilities.

With all four files in place, lock the flake inputs to fetch exact revisions
for a reproducible build:

```bash
nix --extra-experimental-features "nix-command flakes" flake update
```

This creates `flake.lock`, which pins the exact versions of `nixpkgs`, `disko`,
`sops-nix`, and `home-manager` that will be used for the installation.

---

## 7. Generate Keys and Encrypt Secrets

This step generates all keying material needed for the installation: the GitHub
SSH keypair for smithoo4, the server age key used by sops-nix to decrypt secrets
at boot, and the admin age key used to encrypt and edit secrets from outside the
server. All secrets are then collected into a single encrypted file.

The GitHub SSH private key will be encrypted into `secrets/secrets.yaml` and the
public key committed to the repository, so neither needs separate safekeeping.
The only item that must be backed up independently is the admin age private key
— it is the only key that can re-encrypt or edit secrets from outside the server,
and `/tmp` is a tmpfs that is lost when the installer shuts down.

### 7.1 Enter a temporary shell with the required tools

The NixOS live environment does not include `age`, `sops`, `mkpasswd`, or a
standalone `ssh-keygen` build. Start a temporary shell that provides everything
without modifying the installer:

```bash
nix --extra-experimental-features "nix-command flakes" \
  shell nixpkgs#age nixpkgs#sops nixpkgs#mkpasswd nixpkgs#openssh
```

All commands in this section are run inside this shell. Exit it with `exit`
when the section is complete.

### 7.2 Generate the GitHub SSH keypair

Generate an ed25519 keypair that smithoo4 will use to authenticate to GitHub.
The empty passphrase (`-N ""`) is intentional — the private key will be
protected by sops encryption rather than a passphrase, and sops-nix will
decrypt it at boot without any interactive prompt.

```bash
ssh-keygen -t ed25519 -C "13087392+Smithoo4@users.noreply.github.com" \
  -f /tmp/smithoo4_github_ed25519 -N ""
```

Verify both files were created:

```bash
ls -la /tmp/smithoo4_github_ed25519 /tmp/smithoo4_github_ed25519.pub
```

Display the public key and save it somewhere handy — you will need to paste
it into GitHub later:

```bash
cat /tmp/smithoo4_github_ed25519.pub
```

### 7.3 Generate the server age key

```bash
mkdir -p /tmp/sops-nix
age-keygen -o /tmp/sops-nix/key.txt
chmod 600 /tmp/sops-nix/key.txt
```

Extract the public key (the *recipient*) for use in `.sops.yaml`:

```bash
SERVER_RECIPIENT=$(age-keygen -y /tmp/sops-nix/key.txt)
echo $SERVER_RECIPIENT
```

`age-keygen -y` prints the public key that corresponds to the private key file.

### 7.4 Generate the admin age key

```bash
mkdir -p /tmp/sops-admin
age-keygen -o /tmp/sops-admin/key.txt
chmod 600 /tmp/sops-admin/key.txt
```

Extract the admin public key:

```bash
ADMIN_RECIPIENT=$(age-keygen -y /tmp/sops-admin/key.txt)
echo $ADMIN_RECIPIENT
```

> [!IMPORTANT]
> The admin private key at `/tmp/sops-admin/key.txt` is the only way to
> re-encrypt or edit secrets from outside the server. Back it up to a secure
> location (a password manager, encrypted USB drive, etc.) before you reboot.
> `/tmp` is a tmpfs and will be lost when the installer shuts down.

### 7.5 Write .sops.yaml

`.sops.yaml` tells sops which age keys can encrypt and decrypt each secrets
file. Create it at the root of your configuration directory:

`.sops.yaml`
```yaml
keys:
  - &admin ADMIN_RECIPIENT_PLACEHOLDER
  - &oneohm ONEOHM_RECIPIENT_PLACEHOLDER

creation_rules:
  - path_regex: secrets/.*\.yaml$
    key_groups:
      - age:
          - *admin
          - *oneohm
```

Substitute the placeholders with the actual public keys:

```bash
sed -i "s|ADMIN_RECIPIENT_PLACEHOLDER|${ADMIN_RECIPIENT}|g" .sops.yaml
sed -i "s|ONEOHM_RECIPIENT_PLACEHOLDER|${SERVER_RECIPIENT}|g" .sops.yaml
```

Verify the result:

```bash
cat .sops.yaml
```

Both anchors should show full `age1...` public key strings, not placeholders.

### 7.6 Generate a hashed password

Generate a SHA-512 hashed password for smithoo4. You will be prompted to type
the password:

```bash
HASH=$(mkpasswd -m sha-512)
echo $HASH
```

### 7.7 Create and encrypt the secrets file

Build the plaintext secrets file containing both the hashed password and the
GitHub SSH private key. `printf` is used to safely insert the hash (which
contains `$` characters that would be misinterpreted in a heredoc), and `sed`
indents the private key with two spaces to satisfy the YAML block scalar format:

```bash
printf 'smithoo4-password: "%s"\nsmithoo4-ssh-key: |\n' "$HASH" \
  > secrets/secrets.yaml
sed 's/^/  /' /tmp/smithoo4_github_ed25519 >> secrets/secrets.yaml
```

Verify the file looks correct before encrypting:

```bash
cat secrets/secrets.yaml
```

The plaintext file should look like this (with your actual values):

```yaml
smithoo4-password: "$6$rounds=..."
smithoo4-ssh-key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXkAAAAA...
  -----END OPENSSH PRIVATE KEY-----
```

Now encrypt the file in-place. sops reads `.sops.yaml` to determine which
age keys to encrypt for:

```bash
sops -e -i secrets/secrets.yaml
```

Verify the file is encrypted:

```bash
cat secrets/secrets.yaml
```

The file should now contain an encrypted sops envelope — neither the password
hash nor the private key is readable. The file is safe to commit to version
control.

### 7.8 Copy the public key into the configuration directory

The public key is not a secret and is stored as a plain file in the repository.
Home Manager will place it at `~/.ssh/smithoo4_github_ed25519.pub` on the
installed system.

```bash
cp /tmp/smithoo4_github_ed25519.pub secrets/smithoo4_github_ed25519.pub
```

Verify it is in place:

```bash
cat secrets/smithoo4_github_ed25519.pub
```

### 7.9 Exit the temporary shell

```bash
exit
```

---

## 8. Run the Installation

### Partition, format, and mount the disk

Run Disko to partition, format, and mount the disk using the configuration
defined in your flake:

```bash
sudo nix --extra-experimental-features "nix-command flakes" \
  run disko -- \
  --mode destroy,format,mount --flake .#oneohm
```

Verify the disk is mounted:

```bash
lsblk
```

You should see your partitions with `/mnt` and `/mnt/boot` mount points.

### Place the server age key

sops-nix expects the server's age private key to exist at
`/var/lib/sops-nix/key.txt` before the first boot. Copy it into the mounted
system now:

```bash
sudo mkdir -p /mnt/var/lib/sops-nix
sudo cp /tmp/sops-nix/key.txt /mnt/var/lib/sops-nix/key.txt
sudo chown root:root /mnt/var/lib/sops-nix/key.txt
sudo chmod 600 /mnt/var/lib/sops-nix/key.txt
```

> [!NOTE]
> This key never leaves the server. It is what NixOS uses at boot to decrypt
> `secrets/secrets.yaml` and make secrets available to services and users.

### Install NixOS

With the disk mounted and the key in place, install NixOS from the flake:

```bash
sudo nixos-install --flake .#oneohm --root /mnt --no-root-passwd
```

This will take several minutes. It downloads packages from the NixOS binary
cache and installs the full system closure into `/mnt`.

When complete you will see:

```
installation finished!
```

---

## 9. Copy the Configuration to the Installed System

Copy the entire `nixos-config` directory into smithoo4's home directory on the
installed system. This ensures the configuration — including the encrypted
`secrets/secrets.yaml` and `.sops.yaml` — is available after first boot
without needing to re-create it.

```bash
cp -a ~/nixos-config /mnt/home/smithoo4/nixos-config
```

#### Optional: copy the admin key to the installed system

Placing the admin key on the new system allows you to edit and re-encrypt
secrets directly from the server using `sops`. This is convenient during early
development but is not recommended for a long-running server — the admin key
grants the ability to decrypt all secrets, so limit where it lives.

```bash
mkdir -p /mnt/home/smithoo4/.config/sops/age/
cp /tmp/sops-admin/key.txt /mnt/home/smithoo4/.config/sops/age/keys.txt
chmod 600 /mnt/home/smithoo4/.config/sops/age/keys.txt
```

sops automatically reads keys from `~/.config/sops/age/keys.txt`, so this is
all that is needed for `sops` commands to work as smithoo4.

Then reboot:

```bash
sudo reboot
```

---

## 10. First Boot

Boot the machine. After a few seconds systemd-boot will hand off to NixOS and
the system will come up.

### 10.1 SSH in as smithoo4

The server's SSH host key changed when NixOS was installed. Remove the
installer's key from your workstation's known hosts before connecting,
otherwise SSH will refuse the connection:

```bash
ssh-keygen -R <server-ip>
```

Then connect as smithoo4:

```bash
ssh -o StrictHostKeyChecking=accept-new smithoo4@<server-ip>
```

Your SSH key will authenticate automatically — no password prompt.

### 10.2 Put the configuration under version control

The configuration is already in `~/nixos-config`. Initialise a Git repository
so every change is tracked and reversible:

```bash
cd ~/nixos-config
git init
git add .
git commit -m "Initial oneohm configuration"
```

The encrypted `secrets/secrets.yaml` and the plain-text
`secrets/smithoo4_github_ed25519.pub` are both safe to include in the
repository. The next step connects this repository to GitHub and pushes it.

### 10.3 Add the public key to GitHub and push the repository

The GitHub SSH private key is decrypted by sops-nix at boot and placed at
`/run/secrets/smithoo4-ssh-key`, owned by smithoo4 with mode `0600`. The SSH
client configuration written by Home Manager points `github.com` at that path
automatically. The public key was placed at `~/.ssh/smithoo4_github_ed25519.pub`
by Home Manager.

Display the public key:

```bash
cat ~/.ssh/smithoo4_github_ed25519.pub
```

Copy the entire output. In GitHub, go to **Settings → SSH and GPG keys →
New SSH key**, give it a title such as `oneohm`, paste the key, and save.

Test the connection:

```bash
ssh -T git@github.com
```

Expected output:

```
Hi smithoo4! You've successfully authenticated, but GitHub does not provide shell access.
```

> [!NOTE]
> If you see `Permission denied (publickey)`, confirm the full public key was
> pasted correctly into GitHub and that the sops secret decrypted successfully.
> Run `ls -la /run/secrets/smithoo4-ssh-key` to confirm the file exists and
> is owned by smithoo4.

Add the remote and push. Replace `<repo-name>` with the name of the empty
repository you created on GitHub:

```bash
cd ~/nixos-config
git remote add origin git@github.com:smithoo4/<repo-name>.git
git push -u origin main
```

The configuration is now backed up off-site. Any future machine can restore
it by cloning this repository and following the installation steps from Step 5.

---

All maintenance commands are run on the server as `smithoo4` from inside
`~/nixos-config`:

```bash
cd ~/nixos-config
```

### Update the flake

Updating the flake pulls the latest revisions of `nixpkgs`, `disko`,
`sops-nix`, and `home-manager` into `flake.lock`. This is separate from
rebuilding — you control when updates are applied.

```bash
nix flake update
```

Review what changed in the lock file, then apply:

```bash
sudo nixos-rebuild switch --flake .#oneohm
```

Commit the updated `flake.lock` to keep your version history accurate:

```bash
git add flake.lock
git commit -m "flake: update inputs"
```

### Rebuild after a configuration change

Edit any file under `~/nixos-config`, then:

```bash
# Test the new config — takes effect immediately but reverts on next boot
# if you do not follow up with switch.
sudo nixos-rebuild test --flake .#oneohm

# Apply permanently and create a new generation.
sudo nixos-rebuild switch --flake .#oneohm
```

### Edit or add secrets

Use `sops` to decrypt, edit, and re-encrypt a secrets file in one step:

```bash
sops secrets/secrets.yaml
```

sops opens the decrypted file in your `$EDITOR`. Save and quit to re-encrypt
automatically. You will need the admin key available at
`~/.config/sops/age/keys.txt` for this to work.

After editing secrets, rebuild to apply the changes:

```bash
sudo nixos-rebuild switch --flake .#oneohm
```

Commit the updated encrypted file:

```bash
git add secrets/secrets.yaml
git commit -m "secrets: update smithoo4-password"
```

### Roll back a bad rebuild

NixOS keeps old generations. If a rebuild breaks something, reboot and select
the previous generation from the systemd-boot menu. Once back in the working
state, make the rollback permanent:

```bash
sudo nixos-rebuild switch --rollback
```

### Clean up old generations

```bash
sudo nix-collect-garbage --delete-older-than 30d
# Refresh the bootloader menu to remove old entries.
sudo nixos-rebuild boot --flake .#oneohm
```

---

## Reference

| Resource | Link |
|---|---|
| NixOS & Flakes Book | https://nixos-and-flakes.thiscute.world |
| Disko repository | https://github.com/nix-community/disko |
| sops-nix repository | https://github.com/Mic92/sops-nix |
| age repository | https://github.com/FiloSottile/age |
| Home Manager repository | https://github.com/nix-community/home-manager |
| Home Manager options search | https://home-manager-options.extranix.com |
| NixOS package search | https://search.nixos.org/packages |
| NixOS option search | https://search.nixos.org/options |
| NixOS manual | https://nixos.org/manual/nixos/stable |

---

**Version:** 1.6 | **Last Updated:** March 2026
