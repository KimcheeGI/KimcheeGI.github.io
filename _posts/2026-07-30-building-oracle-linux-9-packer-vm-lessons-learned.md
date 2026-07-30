---
layout: post
title: "Building an Oracle Linux 9 Packer VM on Windows: Lessons Learned"
subtitle: "Boot timing, kickstart pitfalls, SSH auth on RHEL9, and Ansible playbook fixes for an air-gapped controller"
date: 2026-07-30
author: "Charles"
tags: [packer, oracle-linux, vmware, kickstart, ansible, air-gapped, stig, devsecops]
---

# Building an Oracle Linux 9 Packer VM on Windows: Lessons Learned

What looks like a straightforward Packer build — boot from ISO, run kickstart, SSH in, done — turned into a four-day deep-dive when every assumption about the Oracle Linux 9 installer, RHEL9's SSH daemon configuration, and Ansible module resolution turned out to be wrong. This post documents every failure, the root cause, and the exact fix.

The goal was a fully automated VMware Workstation VM build on Windows using Packer, producing a hardened Oracle Linux 9 Ansible controller that can operate in an air-gapped environment.

---

## The Stack

- **Host:** Windows 11, VMware Workstation 17.5
- **Packer:** v1.16.0 with `github.com/hashicorp/vmware` plugin ≥ 1.0.0
- **Guest OS:** Oracle Linux 9.8 (`OracleLinux-R9-U8-x86_64-dvd.iso`)
- **Purpose:** Ansible controller for STIG automation in air-gapped environments

---

## Problem 1: Anaconda never saw the kickstart — `boot_wait = "60s"`

### Symptom

The VM booted, Anaconda loaded the graphical installer, and showed the Installation Summary screen with red "Kickstart insufficient!" warnings for Root Password and Installation Destination. After 13+ minutes Packer gave up waiting for SSH.

### Root Cause

The Oracle Linux 9 isolinux boot menu has a **60-second countdown** before it automatically boots the top entry. The Packer config had `boot_wait = "60s"` — meaning Packer waited the full 60 seconds before sending the `<tab>` keystroke. The menu had already timed out and auto-booted into a standard graphical install *without* the kickstart URL.

### Fix

```hcl
boot_wait = "10s"   # catch the menu before it auto-boots
boot_command = [
  "<tab><wait>",
  " inst.ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ks.cfg inst.text<enter><wait>"
]
```

Two changes:
1. **`boot_wait = "10s"`** — sends the keystroke while the isolinux menu is still visible
2. **`inst.text`** — uses text-mode Anaconda, which is faster and more reliable for unattended installs

---

## Problem 2: BIOS vs. EFI — `firmware = "bios"` required

### Symptom

Even after fixing `boot_wait`, the `<tab>` boot command was only working intermittently. On some runs the installer loaded correctly; on others it ignored the keystroke entirely.

### Root Cause

VMware Workstation creates VMs in EFI mode by default for `guest_os_type = "rhel9-64"`. EFI boots into GRUB2, which uses a completely different menu interaction model — pressing `<tab>` does nothing. The isolinux `<tab>` trick only works in **BIOS/legacy** mode.

### Fix

```hcl
firmware = "bios"
```

This forces the VM to boot in BIOS mode, which uses isolinux where `<tab>` edits the selected boot entry's kernel command line.

---

## Problem 3: Kickstart "insufficient" — missing explicit `rootpw` and partitioning

### Symptom

Anaconda showed the installation summary screen with red icons on "Root Password" and "Installation Destination" even when the kickstart URL was correctly injected.

### Root Cause

The early kickstart file used `autopart --type=lvm`, which Anaconda 34 (OL9) sometimes fails to parse correctly, and was missing an explicit `rootpw` directive entirely. Anaconda treats both as "not configured" and requires manual completion.

### Fix

```
rootpw --plaintext CyberSecurity123!
zerombr
clearpart --all --initlabel --disklabel=msdos
part / --fstype=ext4 --size=1 --grow
```

Simple ext4 over a single MBR partition. No LVM, no complexity. Anaconda handles it without hesitation.

---

## Problem 4: SSH authentication failure after successful install — OL9's `sshd_config.d` override

### Symptom

After a successful 13-minute installation and reboot, Packer connected to SSH and immediately got:

```
ssh: unable to authenticate, attempted methods [none publickey password],
no supported methods remain
```

Password auth was definitely enabled — it was in the kickstart `%post`. The root password was definitely set.

### Root Cause

Oracle Linux 9 (and all RHEL 9 derivatives) ship a **drop-in override file** at `/etc/ssh/sshd_config.d/50-redhat.conf` that sets:

```
PermitRootLogin prohibit-password
PasswordAuthentication no
```

Files in `sshd_config.d/` with a **lower** filename prefix take precedence over those with a higher one, and **all drop-ins override the main `sshd_config`**. Editing `/etc/ssh/sshd_config` with `sed` had no effect — the drop-in was winning.

### Fix

Add a `01-packer.conf` drop-in in the kickstart `%post` section. Files prefixed `01-` load before `50-redhat.conf` and override it:

```bash
mkdir -p /etc/ssh/sshd_config.d
cat > /etc/ssh/sshd_config.d/01-packer.conf << 'EOF'
PermitRootLogin yes
PasswordAuthentication yes
EOF
chmod 600 /etc/ssh/sshd_config.d/01-packer.conf
```

---

## Problem 5: Packer provisioner calling `yum -y update` — network blocked

### Symptom

After SSH finally connected, the Packer shell provisioner ran `yum -y update` and failed:

```
Curl error (7): Couldn't connect to server for
https://yum.oracle.com/repo/OracleLinux/OL9/appstream/x86_64/...
[Failed to connect to yum.oracle.com port 443: Connection refused]
```

### Root Cause

This is an air-gapped build environment. The provisioner was trying to reach Oracle's public yum servers which are blocked.

### Fix

Remove the `yum -y update` provisioner entirely. The kickstart already installs everything needed from the local ISO during Anaconda installation (`@core`, `@system-tools`, `@standard`, `python3`, `ansible-core` via pip3). The Packer provisioner was redundant.

```hcl
provisioner "shell" {
  inline = [
    "echo '=== Build verification ==='",
    "python3 --version",
    "ansible --version | head -1",
    "echo '=== Build complete ==='"
  ]
}
```

---

## Problem 6: Ansible playbook — `firewalld` module not found

### Symptom

```
ERROR! couldn't resolve module/action 'firewalld'.
```

### Root Cause

`ansible-core` (installed via pip) does not include the `firewalld` module. It belongs to the `ansible.posix` collection which must be installed separately.

### Fix

```bash
ansible-galaxy collection install ansible.posix
```

And in the playbook, use the fully-qualified collection name:

```yaml
- name: Allow http/https through firewalld
  ansible.posix.firewalld:
    service: https
    permanent: yes
    state: enabled
  ignore_errors: yes
```

---

## Problem 7: `lookup(...) is not failed` — invalid Jinja2 test

### Symptom

```
The conditional check 'lookup('file', '/opt/offline-assets/assets.tar.gz',
errors='ignore') is not failed' failed.
The error was: The 'failed' test expects a dictionary
```

### Root Cause

The `is failed` / `is not failed` Jinja2 tests are designed for **task result dictionaries** (objects with a `failed` key), not for strings. `lookup('file', ...)` returns a string (file contents) or empty string on error — applying `is not failed` to a string throws a type error.

### Fix

Replace all `lookup('file', PATH, errors='ignore') is not failed` conditions with proper `stat` module pre-checks:

```yaml
# Add before the block:
- name: Check for offline bundle
  stat:
    path: /opt/offline-assets/assets.tar.gz
  register: offline_bundle_stat

- name: Import offline asset bundle (if present)
  block:
    - name: Ensure offline folder exists
      file:
        path: /opt/offline-assets
        state: directory
    - name: Unpack offline bundle
      unarchive:
        src: /opt/offline-assets/assets.tar.gz
        dest: /opt/offline-assets/
        remote_src: yes
  when: offline_bundle_stat.stat.exists
```

Six occurrences of this pattern existed across `controller.yml` and `securityonion.yml`. All replaced.

---

## Problem 8: `import_role` with `when:` — condition ignored at parse time

### Symptom

```
ERROR! the role 'stig-manager' was not found in ...
```

Even though the task had `when: have_stig_manager` and `have_stig_manager` was `false`.

### Root Cause

`import_role` is **statically resolved at parse time**, before any `when:` conditions are evaluated. Ansible tries to load the role into the play graph immediately — and fails if the role directory doesn't exist, regardless of the condition.

### Fix

Use `include_role` instead, which is **dynamically resolved at runtime** and correctly skips when the condition is false:

```yaml
- name: Run stig-manager role if present
  include_role:
    name: "stig-manager"
  when: have_stig_manager
```

---

## Manual Intervention Steps Required

Even with a fully automated Packer build, the following steps currently require manual action:

### 1. Re-attach the ISO after Packer completes

Packer detaches the ISO at the end of the build (`Detaching ISO from CD-ROM device ide0:0`). If you need to install additional packages from the DVD (e.g., nginx for air-gapped deployments), you must re-attach it in VMware Workstation settings and reboot the VM — the kernel requires a reboot to detect new optical media in VMware.

### 2. SSH keyboard input from VS Code terminal

VS Code integrated terminals cannot forward keystrokes to interactive SSH password prompts. Solutions:
- **SSH key auth** — generate a key pair, paste the public key into the VM console, then use `-i keyfile` with scp/ssh
- **sshpass via WSL** — `sshpass -p 'password' scp ...` works reliably
- **External terminal** (Windows Terminal, cmd.exe) — handles interactive prompts fine

### 3. Install `ansible.posix` collection before running the playbook

```bash
ansible-galaxy collection install ansible.posix
```

The collection is not included with `ansible-core` and there is no offline-first fallback in the current playbook.

---

## Final Build Outcome

After all fixes, the Packer build completes in ~20 minutes:

```
==> Connected to SSH!
==> === Build verification ===
==> Python 3.9.25
==> ansible [core 2.15.13]
==> === Build complete ===
Build finished after 20 minutes 7 seconds.
Artifacts: VM files in directory: output-oracle_linux
```

The controller playbook runs to:

```
PLAY RECAP
localhost: ok=25  changed=8  failed=0  skipped=12  ignored=1
```

nginx is running as the HTTPS offline asset server, the ansible user is configured with passwordless sudo, and SELinux context is applied to `/opt/offline-assets`.

---

## Key Takeaways

| Assumption | Reality |
|---|---|
| `boot_wait = "60s"` gives Anaconda time to load | It gives the isolinux menu time to time out and auto-boot without the kickstart |
| VMware Workstation uses BIOS by default | It uses EFI for RHEL9; `<tab>` only works with isolinux (BIOS) |
| Editing `/etc/ssh/sshd_config` enables root login | OL9's `50-redhat.conf` drop-in overrides it; write a `01-` prefixed drop-in instead |
| Packer provisioners can update packages | Not in air-gapped environments; do all installs in the kickstart from the local ISO |
| `import_role` respects `when:` | It doesn't — use `include_role` for conditional role execution |
| `lookup(...) is not failed` checks file existence | It's a type error; use the `stat` module |

---

The full project is on GitHub: [KimcheeGI/oracle-stig-so-platform](https://github.com/KimcheeGI/oracle-stig-so-platform)
