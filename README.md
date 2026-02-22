# 🐧 Linux-Sprint — Samba Active Directory

*Alejandro Marti Garcia*

Full deployment of a **Samba Active Directory** environment on Ubuntu Server, structured across 4 progressive sprints.

---

## 📋 Sprints Overview

| Sprint | Goal |
|--------|------|
| **Sprint 1** | Promote the server as an Active Directory Domain Controller |
| **Sprint 2** | Join Linux and Windows clients to the domain |
| **Sprint 3** | Storage management, network shares, and ACL permissions |
| **Sprint 4** | Cross-domain Forest Trust between two AD domains |

---

## 🖥️ Sprint 1 — Domain Controller

Configure `ls12.lab12.lan` as a Samba AD DC.

**Key steps:**
- Hostname + `/etc/hosts` + Netplan static IPs
- Immutable `/etc/resolv.conf` using `chattr +i`
- Package install: `samba`, `krb5`, `winbind`, `chrony`
- Domain provisioning: `samba-tool domain provision`
- NTP sync with Chrony + Samba NTP signing socket
- Verification: DNS (`host`), Kerberos (`kinit`), SMB (`smbclient`)

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
sudo systemctl enable --now samba-ad-dc
```

---

## 💻 Sprint 2 — Client Integration

Join client machines to `lab12.lan` and configure AD authentication.

**Key steps:**
- Time sync with `ntpdate lab12.lan`
- Package install: `samba`, `winbind`, `krb5-user`, `libpam-winbind`, `libnss-winbind`
- Configure `smb.conf` with `security = ADS`
- Join the domain: `net ads join -U administrator`
- NSS (`/etc/nsswitch.conf`) + PAM (`pam_mkhomedir`)
- Create users, groups, and OUs via `samba-tool`
- GPOs: password complexity and account lockout policies

```bash
sudo net ads join -U administrator
sudo samba-tool domain passwordsettings set --min-pwd-length=8 --complexity=on
```

---

## 💾 Sprint 3 — Storage, Shares & ACLs

Dedicated disk for department-based network shares with granular permissions.

**Key steps:**
- New disk → `fdisk` → `mkfs.ext4` → persistent mount in `/etc/fstab`
- Mount options: `user_xattr,acl,barrier=1`
- Samba shares in `smb.conf`: `FinanceDocs`, `HRDocs`, `Public`
- ACLs with `setfacl` including default inheritance (`-d` flag)
- Audit with `getfacl` and access tests via `smbclient`
- Process management: `top`, `htop`, `ps`, `kill`, `systemctl`

```bash
sudo setfacl -m g:Students:rwx /srv/samba/Data/FinanceDocs
sudo setfacl -d -m g:Students:rwx /srv/samba/Data/FinanceDocs
```

| Share | Groups | Permissions |
|-------|--------|-------------|
| `FinanceDocs` | Students, Domain Admins | `rwx` |
| `HRDocs` | IT_Admins, Domain Admins | `rwx` |
| `Public` | Domain Users / Domain Admins | `r-x` / `rwx` |

---

## 🤝 Sprint 4 — Domain Trust

Bidirectional forest trust between `lab12.lan` and `lab120.lan`.

**Key steps:**
- Provision second DC `ls120.lab120.lan` with independent domain `LAB120.LAN`
- Configure network, `resolv.conf`, Kerberos and Chrony on LS120
- Run `samba-tool domain provision` for the new domain
- Cross-reference `/etc/hosts` on both servers for mutual resolution
- Create and validate the trust relationship

```bash
sudo samba-tool domain trust create lab120.lan \
  --type=forest --direction=both \
  -U administrator@lab120.lan

sudo samba-tool domain trust validate lab120.lan
```

---

## 🔧 Requirements

- Ubuntu Server 22.04 / 24.04
- Samba 4.x
- Two network interfaces: one external (WAN) + one internal (AD network)
- Internal DNS pointing to the DC itself (`127.0.0.1`)

---

## 📁 Full Documentation

> See `Sprint.md` for the complete step-by-step guide with screenshots and detailed explanations of every command.

---

*📄 Linux-Sprint · Alejandro Marti Garcia*
