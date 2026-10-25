# KDE neon Incus Remote CLI Setup Guide

**Hostname:** `proteus`  
**User:** `morgan`  
**IncusOS server:** `192.168.0.50:8443`  
**Remote aliases:** `incusos` (TLS client certificate, daily use) and `incusos-sso` (SSO, bootstrap only)

A workstation setup guide for adding the Incus repository, installing the Incus CLI and `incus-migrate`, and connecting KDE neon to an existing IncusOS server on the same LAN. This guide continues from the supplied **KDE neon User Edition Workstation Setup Guide** and uses native DEB packages.

> **Starting point:** KDE neon is already installed using the existing workstation guide. This is a client workstation, not a new Incus server. Do not run `incus admin init` or configure local Incus storage or networking.
>
> **Access model:**
>
> - **Web interface:** SSO.
> - **CLI on this workstation:** a TLS client certificate. This is the CLI's own keypair, created automatically by the Incus client, and made trusted by the server through a one-time trust token.
> - **Bootstrap and recovery:** SSO is used once to create that trust token. It is also the way back in if the CLI credentials are ever lost.
>
> This means no `client.pfx` file is needed and nothing needs to be imported into Chrome.

## Contents

1. [Check the Operating System Base](#1-check-the-operating-system-base)
2. [Confirm SSO Access](#2-confirm-sso-access)
3. [Add the Incus Repository](#3-add-the-incus-repository)
4. [Install the Incus CLI Tools](#4-install-the-incus-cli-tools)
5. [Connect to IncusOS](#5-connect-to-incusos)
6. [Verify Remote Access](#6-verify-remote-access)
7. [Review and Revoke Certificates](#7-review-and-revoke-certificates)

---

## 1. Check the Operating System Base

### 1a. Complete the Existing Workstation Setup

Start with KDE neon installed and updated using the existing workstation guide. Retain its hostname and user settings:

- **Hostname:** `proteus`
- **User:** `morgan`

The local KVM/libvirt packages in the original guide are not required for managing a remote IncusOS server. Existing installations of those tools do not need to be removed.

### 1b. Check the Base and Repository Suite

The Zabbly repository is organised by the **Ubuntu or Debian base** of the system, not by a KDE neon release name. Open **Konsole** and run:

```bash
cat /etc/os-release
dpkg --print-architecture
( . /etc/os-release && echo "Repository suite: ${UBUNTU_CODENAME:-${VERSION_CODENAME}}" )
```

The last line prints the suite that Section 3 will use. It reads `UBUNTU_CODENAME` when the system sets one (Ubuntu and Ubuntu-based systems) and otherwise falls back to `VERSION_CODENAME` (Debian). Section 3 reads the same value directly from `/etc/os-release` when it writes the repository file, so there is nothing to copy or set by hand.

Confirm that the suite and architecture are ones Zabbly publishes for. At the time of writing these are: [1]

- **Ubuntu:** `jammy` (22.04), `noble` (24.04), `resolute` (26.04)
- **Debian:** `bookworm` (12), `trixie` (13)
- **Architectures:** `amd64`, `arm64`

If the suite or architecture is not listed, stop and check the Zabbly page before continuing.

## 2. Confirm SSO Access

This section confirms that your SSO login works in the web interface. Section 5 relies on it.

1. Open the [IncusOS web interface](https://192.168.0.50:8443) in Google Chrome.
2. If Chrome reports an untrusted server certificate, continue to the site only for this check. Do not disable browser certificate checks globally.
3. Sign in using SSO.
4. Confirm that the interface opens and that you can view the server's instances.

Incus gives any user who authenticates through its configured OIDC provider full access to Incus, so a successful SSO login should be enough to create the trust token in Section 5. [3]

---

## 3. Add the Incus Repository

This guide uses the **Zabbly stable repository** for the client tools. Zabbly publishes Ubuntu-compatible Incus packages and a separate client-only package. Use one Zabbly Incus release channel, not stable and daily together. [1]

### 3a. Install Repository Prerequisites

```bash
sudo apt update
sudo apt install ca-certificates curl
```

### 3b. Install the Signing Key

Save the Zabbly signing key: [1]

```bash
sudo mkdir -p /etc/apt/keyrings/
sudo curl -fsSL https://pkgs.zabbly.com/key.asc -o /etc/apt/keyrings/zabbly.asc
```

### 3c. Add the Stable Repository

The suite and architecture are read from the operating system when the command runs, using the same logic as the check in Section 1b: [1]

```bash
sudo tee /etc/apt/sources.list.d/zabbly-incus-stable.sources > /dev/null <<EOF_REPO
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-${VERSION_CODENAME}}")
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/zabbly.asc
EOF_REPO

sudo apt update
```

Confirm that APT completes without signature or repository errors.

---

## 4. Install the Incus CLI Tools

The required packages are:

- **`incus-client`:** the Incus command-line client for managing remote servers. [1]
- **`incus-extra`:** additional tools, including `incus-migrate`. [2]

### 4a. Install the Tools

```bash
sudo apt install --no-install-recommends incus-client incus-extra
```

> **Client-only check:** Do not install `incus` or `incus-base`, which provide server components. If APT proposes either package, stop and inspect the dependency requirements rather than accepting a local server installation.

### 4b. Verify the Commands

```bash
command -v incus
command -v incus-migrate
incus --version
incus-migrate --help
```

Both commands should be found. `incus-migrate --help` should display the migration tool's usage information.

No local Incus initialization, `incus-admin` group membership, or local virtualization configuration is needed for this remote-client workflow.

> **Scope:** This guide installs and verifies `incus-migrate`; it does not perform a migration or create a VM.

---

## 5. Connect to IncusOS

Run the following commands as `morgan`, **without `sudo`**. This keeps the CLI configuration and credentials associated with the workstation user.

The process has three steps: sign in once with SSO, create a trust token, then use the token to register this workstation's CLI certificate with the server.

### 5a. Add a Temporary SSO Remote

```bash
incus remote add incusos-sso https://192.168.0.50:8443 --auth-type=oidc
```

When Incus displays the server certificate fingerprint, accept it to continue.

Follow the browser sign-in prompt shown in the terminal, confirm the device code if one is shown, and sign in with SSO. Then confirm that the remote works:

```bash
incus list incusos-sso:
```

> **Note:** On first use the Incus client generates its own keypair in `~/.config/incus/`. For this SSO remote that keypair only secures the connection. The server does not need to trust it yet.

### 5b. Create a Trust Token

Create a token for this workstation. The name is a label that will appear in the server's trust list, so use something you will recognize later:

```bash
incus config trust add incusos-sso:proteus-cli
```

Incus prints a token. Treat it as a credential: use it only on this workstation, do not paste it into messages, and use it promptly. [3]

If you create a token by mistake, revoke it as shown in Section 7b.

### 5c. Add the TLS Remote

```bash
incus remote add incusos https://192.168.0.50:8443 --auth-type=tls
```

Accept the server certificate fingerprint when it is displayed. When the server asks for a trust token, paste the token from Section 5b. The server then adds this workstation's client certificate to its trust store, and the connection is granted. [3]

The alias `incusos` is a local name for the remote server. It does not rename the server.

---

## 6. Verify Remote Access

### 6a. List Configured Remotes

```bash
incus remote list
```

Confirm that `incusos` points to `https://192.168.0.50:8443` and that its authentication type is `tls`. The temporary `incusos-sso` remote should show `oidc`.

### 6b. Test Read-Only Server Queries

```bash
incus info incusos:
incus list incusos:
incus storage list incusos:
incus network list incusos:
incus config trust list incusos:
```

Confirm that the server responds and that the listed resources match the existing server. The trust list should now include an entry named `proteus-cli`. An empty instance list is valid if no instances exist in the selected project. Permission errors may indicate project restrictions rather than a failed connection.

These commands inspect the remote server without creating instances or changing its storage or network configuration.

### 6c. Make IncusOS the Default Remote

```bash
incus remote switch incusos
incus remote get-default
incus list
```

The default remote should be `incusos`. Commands without a remote prefix now target this server. [4]

> **Important:** Before running any future create, delete, or configuration command, confirm the intended remote. Use an explicit `incusos:` prefix when you want to make the target unambiguous.

### 6d. Remove the Temporary SSO Remote

Once Sections 6a to 6c all work over `incusos`, remove the bootstrap remote:

```bash
incus remote remove incusos-sso
```

This removes only this workstation's configuration for that remote. It does not affect the server, the trusted `proteus-cli` certificate, or your SSO access to the web interface. You can add the SSO remote again at any time to create another token.

---

## 7. Review and Revoke Certificates

Use this section periodically, and after replacing a workstation or recovering from lost credentials. If the CLI's key was lost, first repeat Section 5 to create a fresh identity, then use this section to remove the old, unusable certificate. Run the commands without `sudo`. Unless noted, they use the `incusos:` remote.

> **Before you remove anything:** Confirm you have a second way into the server. The web interface over SSO is the usual one. Removing the certificate this workstation is currently using locks the CLI out until you repeat Section 5.

### 7a. List the Certificates Trusted by the Server

```bash
incus config trust list incusos:
```

This lists the client certificates the server trusts. [3] Typical columns are the name, common name, a shortened fingerprint, type, and expiry date, though the exact columns can vary by version. For full details, add `--format yaml`.

To inspect one entry:

```bash
incus config trust show incusos:<fingerprint>
```

### 7b. Check for Unused Tokens

Tokens that were created but never used are credentials waiting to be redeemed. List them, then revoke any you no longer need, using the name you gave in Section 5b:

```bash
incus config trust list-tokens incusos:
incus config trust revoke-token incusos:<name>
```

### 7c. Decide What to Remove

Candidates for removal:

- A certificate from the original, lost `client.pfx` identity, if it cannot be recovered and nothing else uses it.
- Certificates left from old or replaced workstations.
- Test or temporary certificates.
- Expired certificates.

Do not remove:

- The entry for this workstation's certificate (normally `proteus-cli`).
- A certificate used by another machine or service that you still rely on.

The trust list does not show when a certificate was last used. If you are unsure whether something is still in use, check the name and expiry date with `incus config trust show`, and confirm with the other machines before removing it.

### 7d. Remove a Certificate

```bash
incus config trust remove incusos:<fingerprint>
```

This revokes the certificate's access to the server. [3] Use the fingerprint from the list. If the shortened form is rejected, use the full fingerprint from `incus config trust show`. The change takes effect immediately and affects only trust on the server. It does not delete the key or certificate files on the machine that holds them.

Afterwards, confirm the entry is gone and that this workstation still works:

```bash
incus config trust list incusos:
incus info incusos:
```

To restore access for a removed certificate, repeat the token process in Section 5.

### 7e. Clean Up the Local CLI

List the remotes configured on this workstation:

```bash
incus remote list
```

Remove any that are no longer needed:

```bash
incus remote remove <name>
```

Removing a remote affects only local configuration. After removing one, look through `~/.config/incus/` for leftover files belonging to it, such as saved server certificates or SSO session data, and remove them if you want a thorough cleanup. Do not delete `client.crt` and `client.key` unless you intend to create a new identity.

If you followed an earlier version of this guide and created a `client.pfx` copy, delete it once you no longer need it:

```bash
rm -f "$HOME/Downloads/client.pfx"
rm -f "$HOME/.local/share/incus-credentials/client.pfx"
```

### 7f. SSO Access Is Managed Separately

SSO access does not appear in `incus config trust list`. To revoke a person's SSO access to the server, remove or disable them in your identity provider.

---

## References

- Supplied **KDE neon User Edition Workstation Setup Guide**, version 0.2 Draft, September 30, 2026.
- **[1]** [Zabbly Incus package repository and installation instructions](https://github.com/zabbly/incus). Source of the key and repository commands, and of the list of supported releases.
- **[2]** [Zabbly incus-extra package file manifest](https://github.com/zabbly/incus/blob/daily/debian/incus-extra.install). This is the repository's daily-branch manifest; check which packages APT installs from the stable repository.
- **[3]** [Incus remote API authentication](https://linuxcontainers.org/incus/docs/main/authentication/). Covers trust tokens, OIDC authentication, and `incus config trust` list, add, and remove.
- **[4]** [Incus client documentation](https://linuxcontainers.org/incus/docs/main/client/).

**Version:** 0.5 Draft  
**Last updated:** October 25, 2026
