# Finding the Right Hypervisor: A Comparison for a Home Lab VM Server

**Date:** April 2026  
**Hardware tested:**
- HP EliteDesk 800 G3 SFF — used for Incus on Debian, XCP-ng, and KVM + Cockpit
- HP EliteDesk 800 G4 DM — used for IncusOS and SmartOS (has TPM 2.0 natively)

**Goal:** Headless hypervisor with a web UI for VM testing on a spare hardware

---

## The Starting Point

I had a few spare HP EliteDesk sitting around and wanted to turn it into a dedicated VM test server. The requirements were simple:

- Headless (no monitor/keyboard)
- Web UI for overview, start/stop, and basic management
- CLI-friendly — I like editing config files
- Free to use

---

## The Options Considered

I cast a wide net and looked at nine platforms:

| Platform | Type | Tested? |
|---|---|---|
| **Proxmox VE** | Bare-metal KVM + LXC | ❌ Not this round (used previously) |
| **XCP-ng** (with XO Lite) | Bare-metal Xen | ✅ Tested |
| **Incus on Debian** | KVM + LXC + OCI on Linux | ✅ Tested |
| **IncusOS** | Immutable Incus appliance | ✅ Tested (on HP G4 DM) |
| **KVM + Virt-Manager + Cockpit** | KVM on Linux | ✅ Already using |
| **SmartOS** | Illumos-based (Zones + KVM) | ✅ Tested (on HP G4 DM) |
| **oVirt** | Enterprise KVM management | ❌ Dismissed — too heavy |
| **Nutanix Community Edition** | Enterprise HCI | ❌ Dismissed — 32 GB RAM + 3 drives required |
| **Harvester** | Kubernetes-native HCI | ❌ Dismissed — 32 GB RAM, overkill for single node |

**oVirt**, **Nutanix CE**, and **Harvester** were dropped early. oVirt needs a separate 16 GB engine VM. Nutanix requires 32 GB RAM and three dedicated drives. Harvester eats 9–16 GB just to run its stack before a single VM exists. All three are designed for multi-node clusters, not a single spare desktop.

---

## The Shortlist — What I Actually Tested

### 🥇 IncusOS — The Clear Winner

**IncusOS won, and it wasn't close.**

IncusOS is a purpose-built, immutable operating system designed to run Incus.

- **The best web UI of any platform I tested.** Clean, intuitive, does everything I wanted. The killer feature: an **Edit YAML** button that lets you view and modify the full instance configuration directly in the browser. This is exactly how I like to work — a UI that shows me the config and lets me change it.
- **In-browser graphical console is one of the best.** It worked flawlessly and was one of the best console experiences across all the platforms I tested.
- **Tons of pre-built images.** The Incus image server has hundreds of ready-to-launch OS images. Spinning up a test VM or container takes seconds. A feature I did not know I wanted.
- **YAML everywhere.** Profiles, instance configs, preseed files — everything is YAML. Create a profile once, reuse it across VMs. Export a config, version-control it, import it elsewhere. It just makes sense.
- **VMs and containers in one tool.** KVM virtual machines, LXC system containers, and OCI containers — all managed with the same CLI and API.
- **Immutable and self-updating.** IncusOS uses A/B atomic updates — the OS is read-only and updates are applied as whole images. No `apt`, no package drift, no "it worked until I updated something." If an update fails, it rolls back automatically.
- **Active development.** New stable releases every six weeks. The project is led by Stéphane Graber and the community is growing fast.
- **I'd replace Virt-Manager with Incus.** Next time I rebuild my workstation, my plan would be to install Incus over Virt-Manager.

**The downsides:**

- **Hardware requirements are strict.** IncusOS requires UEFI, Secure Boot, and TPM 2.0. My HP EliteDesk 800 G3 SFF only has TPM 1.2, and I couldn't get the firmware flash to 2.0 working (the tool only runs under Windows, which I'd already wiped, and I couldn't get a bootable Windows USB working). I ended up using an **HP EliteDesk 800 G4 DM 65W (TAA)** instead, which has TPM 2.0 natively.
- **Non-traditional installation.** You don't just flash an ISO. You generate a custom image via a web customizer tool with your TLS client certificate baked in. The system has no SSH and no local shell — the only way in is through the Incus API and web UI. It's a different paradigm, and if you lose your client certificate, you're reinstalling.
- **No VM performance metrics in the web UI.** I can see what's running and manage it, but I can't see CPU/RAM usage per VM in the browser. Would be a nice addition.

#### Incus on Debian — The Fallback

If your hardware doesn't meet IncusOS's requirements (TPM 2.0, Secure Boot, UEFI), **Incus on Debian** gives you the exact same Incus experience — same VMs, same containers, same CLI, same web UI, same YAML. You just manage the host OS yourself (`apt update`, SSH access, etc.).

#### 🇨🇦 Shout-out to Stéphane Graber

A fellow Canadian! Stéphane is based in Shefford, Quebec and has been working on container technology for about 15 years. He's the project leader of Linux Containers (Incus, LXC, LXCFS, Distrobuilder), owner of Zabbly (the company that provides the Incus package repository and infrastructure), and CTO of FuturFusion — a company building an Incus-based private cloud aimed at replacing VMware. In his spare time, he's also the VP of Infrastructure for NorthSec, a cybersecurity conference and CTF in Montreal that runs hundreds of VMs and thousands of containers for its challenges.

The entire Incus ecosystem — packages, CI/CD, image building, hosting — runs through Zabbly. If you use Incus and want to support its continued development, Stéphane accepts donations via [GitHub Sponsors](https://github.com/sponsors/stgraber), [Ko-fi](https://ko-fi.com/stgraber), and [Patreon](https://www.patreon.com/stgraber).

**Verdict:** IncusOS is my new go-to for dedicated VM servers. It fits my workflow perfectly — config files for the real work, the best web UI with in-browser graphical console and YAML editing, tons of pre-built images, and the flexibility to grow into clustering in the future.

---

### 🥈 SmartOS — The Surprise

**I did not expect to like SmartOS as much as I did.**

- **Incredibly lightweight.** The resource usage was amazingly low. The OS boots entirely into RAM from a USB stick, and almost all system resources go to your VMs and zones.
- **Easy to set up.** Boot from USB, answer a few prompts, SSH in, done. Quicker than I expected.
- **JSON config files.** VMs are defined by JSON manifests passed to `vmadm`. Different from YAML but the same idea — your infrastructure is defined in config files.
- **ZFS built-in.** Illumos has first-class ZFS support (it's where ZFS was born). Snapshots, compression, scrubs — it all just works.

**The downsides:**

- **Web UI is limiting.** The SmartOS web UI on port 4443 can list instances and start/stop them, but that's about it. No VNC console in the browser, no JSON config editing, no update management. Almost everything is CLI.
- **Uncertain future.** SmartOS is based on Illumos (Solaris lineage), which is a niche ecosystem. The company behind it (MNX Cloud, formerly Joyent) has been bought and sold multiple times. Development continues but the year-over-year commit activity is declining. The community is small and there's no clear market position to sustain it long-term.
- **Not Linux.** Commands are different (`vmadm`, `imgadm`, `dladm`, `svcadm`), packaging is different (`pkgin` in zones only), and persistence works differently (`/root` is tmpfs!). It's a learning experience, which is both a pro and a con.

**Verdict:** SmartOS is a fascinating platform that I genuinely enjoyed using. For a weekend deep-dive into something completely different, it's excellent. But the uncertain future and limited web UI make it hard to commit to for the long term.

---

### 🥉 XCP-ng with XO Lite — Worth Watching

- **Easy to set up.** ISO to USB, boot, install, done. XO Lite is accessible immediately at `https://<ip>` with zero extra setup.
- **XO Lite is promising but incomplete.** It handles basic VM lifecycle (create, start, stop, console) but is missing patch management, ISO uploads, advanced networking, and DHCP config. The interesting thing: the UI itself has notes/placeholders where upcoming features will go — it's clearly under heavy active development.
- **Full Xen Orchestra fills the gaps** but requires building from source (free) or purchasing from Vates. For a single test box, that's a lot of overhead.

**Verdict:** XO Lite isn't quite there yet for my needs, but the pace of development is encouraging. Worth revisiting in 6–12 months to see where it lands. If Vates delivers on the roadmap, XCP-ng + XO Lite could become a very strong turnkey option.

---

### KVM + Virt-Manager (+ Cockpit) — The Incumbent

This is what I was already using on my workstation (minus Cockpit). I gave Cockpit a try alongside it.

- **Virt-Manager feels dated.** It works, but the XML-heavy approach is cumbersome.
- **Cockpit was OK but limited.** The web UI is clean for basic tasks, but it lacks configurability. You can't disable Secure Boot from the UI, for example. And there's no way to view or edit the underlying XML — which defeats the purpose for me. If I'm going to use the CLI anyway, I'd rather use a tool with better CLI ergonomics.

**Verdict:** Functional but uninspiring. Cockpit is a decent web UI for system administration but not a great VM management tool. The short answer is Incus just seemed newer and better to me.

---

### Proxmox VE — Skipped This Round

I've used Proxmox before and chose not to test it this time. It's a great platform — I understand why it's so popular. But it's not for me.

I had a hard time quantifying what doesn't work for me. I think it has to do with **Proxmox being too much of GUI-driven tool for my taste.** I prefer config files, and when there is a UI, I want to see how it maps back to a file I can edit. Incus's YAML-everywhere approach and SmartOS's JSON manifests are more aligned with how I think. Proxmox hides the config behind a web interface, and while you *can* edit files on the backend, it's not designed to work that way.

---

## Final Comparison — The Tested Five

| | **IncusOS / Incus** | **SmartOS** | **XCP-ng** | **KVM + Cockpit** | **Proxmox** |
|---|---|---|---|---|---|
| **Web UI Quality** | 🏆 Best | ⭐⭐ | ⭐⭐⭐ (XO Lite) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Config File Approach** | ✅ YAML | ✅ JSON | ❌ GUI-focused | ⚠️ XML (clunky) | ❌ GUI-focused |
| **Edit Config from UI** | ✅ YAML editor | ❌ CLI only | ❌ | ❌ | ❌ |
| **In-Browser Graphical Console** | ✅ Excellent | ❌ External VNC | ✅ VNC | ✅ VNC | ✅ noVNC + SPICE |
| **Resource Usage** | Low | 🏆 Lowest | Medium | Low | Medium |
| **Containers + VMs** | ✅ Both + OCI | Zones + KVM | VMs only | VMs only | ✅ Both |
| **Pre-built Images** | 🏆 Hundreds | Dozens | Few | None | Templates/CTs |
| **Community & Future** | 🟢 Growing fast | 🔴 Uncertain | 🟢 Solid (Vates) | 🟢 Stable | 🟢 Dominant |
| **Learning Curve** | Medium | High (Illumos) | Low–Medium | Low | Low |
| **Immutable OS Option** | ✅ IncusOS | ✅ Boots into RAM | ❌ | ❌ | ❌ |

---

## The Verdict

**IncusOS is the clear winner for me.** It fits my workflow perfectly — YAML config files, the best web UI of any platform I tested (with in-browser graphical console and YAML editing), tons of pre-built images, an immutable self-updating OS, and a path forward for clustering when I'm ready. For hardware that meets the requirements (UEFI + Secure Boot + TPM 2.0), IncusOS is the answer. For hardware that doesn't, Incus on Debian gets you 95% of the way there.

**SmartOS was the surprise.** I went in expecting a novelty and came out genuinely impressed. The resource efficiency is remarkable and the JSON-config approach appeals to me. But the limited web UI and uncertain future keep it as a "fun to explore" option rather than a daily driver.

**XO Lite is worth watching.** Give it another year of development and revisit. If the feature gaps get filled, XCP-ng becomes a much stronger contender.

**Everything else** — oVirt, Nutanix CE, Harvester — is designed for enterprise/multi-node deployments and is overkill for a single spare desktop.

---

*Written April 2026. Hardware: HP EliteDesk 800 G3 SFF and HP EliteDesk 800 G4 DM 65W (TAA). Your mileage may vary — try them yourself and see what clicks.*
