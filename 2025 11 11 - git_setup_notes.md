# Git Initial Setup Notes

## 1. SSH Key Setup for GitHub

This setup uses the recommended `ed25519` key type.

* **Generate Key:**
    ```bash
    ssh-keygen -t ed25519 -C "13087392+Smithoo4@users.noreply.github.com"
    ```
    *(Note: Use a strong passphrase when prompted.)*

* **Start Agent:**
    ```bash
    eval "$(ssh-agent -s)"
    ```

* **Add Key to Agent:**
    ```bash
    ssh-add ~/.ssh/id_ed25519
    ```

* **Copy Public Key:**
    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```
    *(Action: Copy the entire output string starting with `ssh-ed25519...`.)*

* **Upload to GitHub:**
    Go to **GitHub** $\rightarrow$ **Settings** $\rightarrow$ **SSH and GPG keys** $\rightarrow$ **New SSH key**. Paste the key you just copied.

* **Test Connection:**
    ```bash
    ssh -T git@github.com
    ```
    *(Verification: You should see a successful authentication message.)*

---

## 2. Global Git Configuration

Run these commands *once* on a new machine to set your preferred identity and default branch name.

```bash
# Set Identity (Using GitHub No-Reply Email and Name)
git config --global user.name "Morgan Smith"
git config --global user.email 13087392+Smithoo4@users.noreply.github.com

# Set Default Branch Name (Best Practice is 'main')
git config --global init.defaultBranch main

# Verification Step: View all global settings
git config --global --list
