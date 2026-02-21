# 🐧 Linux-Sprint

> **Technical documentation** of the full configuration process for a Linux environment running Samba as an Active Directory Domain Controller. This project is structured across four sprints — each one building on the previous — covering server promotion, client integration, storage management, and inter-domain trust relationships.

This document serves as a step-by-step reference guide, including commands, configuration files, expected outputs, and screenshots at every key stage of the process.

<br>

--- 

## 📋 Table of Contents

- [Sprint 1 — Promote Domain Controller](#️-sprint-1--promote-domain-controller)
- [Sprint 2 — Linux and Windows Client Integration](#-sprint-2--linux-and-windows-client-integration)
- [Sprint 3 — Storage, Shares & Permissions](#-sprint-3--storage-shares--permissions)
- [Sprint 4 — Domain Trust](#-sprint-4--domain-trust)

---

<br>

# 🖥️ SPRINT 1 — Promote Domain Controller

> **Goal:** Configure a fresh Ubuntu Server installation as a fully functional Samba Active Directory Domain Controller (AD DC). This involves setting the hostname correctly, configuring both network interfaces with static IPs, installing all required packages, provisioning the Samba AD domain, setting up time synchronization, and verifying that all services — DNS, Kerberos, LDAP, and SMB — are working correctly before any clients are added.

<br>

---

## ⚙️ Step 1 — Hostname Configuration

The hostname is the foundation of the server's identity within the network. In an Active Directory environment, the hostname becomes part of the server's **Fully Qualified Domain Name (FQDN)** — for example, `ls12.lab12.lan`. Both Kerberos authentication and DNS rely heavily on this name being set correctly and consistently from the very beginning. A hostname set incorrectly at this stage will cause cascade errors in later steps.

The following command sets the hostname permanently at the system level:

```bash
sudo hostnamectl set-hostname ls12
```

> This command writes the new hostname to `/etc/hostname` and updates the running system immediately — no reboot is required. The name `ls12` stands for **Linux Server 12**, which will later resolve to `ls12.lab12.lan` when DNS is configured.

<br>

The screenshot below confirms the hostname was set and is active. Notice that the shell prompt has updated to reflect the new name:

<br>

<img width="474" height="71" alt="imagen" src="https://github.com/user-attachments/assets/6932197e-3e60-4bac-ad15-ba54634bc41a" />

<br>

---

## 📄 Step 2 — Edit the /etc/hosts File

The `/etc/hosts` file is the system's local DNS resolver — it maps hostnames to IP addresses without needing to query an external DNS server. This is especially important during the early stages of AD setup, because Samba's provisioning tool needs to resolve its own FQDN before the DNS service is running.

We open the file with:

```bash
sudo nano /etc/hosts
```

Inside the file, we need to add a line that maps the server's static IP address to both its FQDN and its short hostname. The format is:

```
192.168.30.40   ls12.lab12.lan   ls12
```

> Make sure the loopback entry `127.0.0.1 localhost` remains intact. The new line should use the **real static IP** of the server, not `127.0.0.1`. If the FQDN is not resolvable from `/etc/hosts`, the Samba provisioning step will fail.

<br>

The screenshot below shows the hosts file open in the nano editor, ready to be modified:

<br>

<img width="801" height="179" alt="imagen" src="https://github.com/user-attachments/assets/23de9c27-387b-4207-8132-0e11b7a36283" />

<br>

After saving the file with `Ctrl+O` and exiting with `Ctrl+X`, you can verify the hostname resolves correctly by running `hostname -f`, which should return the full FQDN `ls12.lab12.lan`.

<br>

---

## 🌐 Step 3 — Network Configuration (Netplan)

The Domain Controller needs **static IP addresses** on all network interfaces. Without static IPs, the server's address could change between reboots, breaking DNS records and causing client authentication to fail. Ubuntu uses **Netplan** as its network configuration layer, which writes to YAML files and passes the result to either `networkd` or `NetworkManager`.

This server has two interfaces:
- `enp0s3` — connects to the **external/WAN** network for internet access and upstream DNS
- `enp0s8` — connects to the **internal AD network**, where domain clients will connect

We open the configuration file:

```bash
nano /etc/netplan/00-installer-config.yaml
```

<br>

We configure both interfaces with the following YAML:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses:
        - 172.30.20.77/25
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.0.0.1
      routes:
        - to: default
          via: 172.30.20.1
          metric: 100
    enp0s8:
      addresses:
        - 192.168.30.40/24
      nameservers:
        addresses:
          - 127.0.0.1
```

> **Why does `enp0s8` point DNS to `127.0.0.1`?** Once Samba AD is running, it takes over the DNS service for the internal domain. By pointing the internal interface at `127.0.0.1`, we ensure that the DC resolves its own domain records using its own DNS service — which is essential for Kerberos and LDAP to function correctly. The external interface still uses Cloudflare DNS (`1.1.1.1`) for internet access.

<br>

The screenshot below shows the Netplan file with the correct configuration for both interfaces. Note the indentation — YAML is sensitive to whitespace errors:

<br>

<img width="978" height="396" alt="imagen" src="https://github.com/user-attachments/assets/a107fee4-0764-4043-90f0-0b62c1d6cabd" />

<br>

After saving the file, we check whether the configuration is valid before applying it. An invalid YAML file could disconnect the server from the network:

<br>

<img width="795" height="102" alt="imagen" src="https://github.com/user-attachments/assets/495f7dc0-e14f-49c1-9405-8be5337216b9" />

<br>

If the validation passes with no errors, we apply the changes:

```bash
sudo netplan apply
```

> If the command returns silently with no output, the configuration was applied successfully. You can confirm the new IP addresses are active by running `ip a` or `ip addr show`.

<br>

The following screenshot shows the network interfaces after the configuration is applied. Both `enp0s3` and `enp0s8` now have their assigned static IPs:

<br>

<img width="631" height="131" alt="imagen" src="https://github.com/user-attachments/assets/e77da347-3d6d-4ac4-b232-ef60bb549406" />

<br>

We also verify the routing table to confirm the default gateway is set correctly via `enp0s3`:

<br>

<img width="798" height="341" alt="imagen" src="https://github.com/user-attachments/assets/09f7ee55-1976-4699-abee-e882b24fada9" />

<br>

A quick connectivity test confirms the server can reach the internet through the external interface:

<br>

<img width="623" height="53" alt="imagen" src="https://github.com/user-attachments/assets/683f6c83-ddb9-4143-b90e-cfaa1ea63084" />

<br>

---

## 🔗 Step 4 — Fix DNS Resolution (/etc/resolv.conf)

By default, Ubuntu links `/etc/resolv.conf` to a file managed by `systemd-resolved`. While useful in most scenarios, this symlink can be overwritten at runtime by network events, DHCP responses, or VPN connections — all of which would break our carefully configured DNS setup.

The solution is to delete the symlink, create a static file, and then mark it as **immutable** so that no process can modify it.

<br>

**Step 4.1 — Remove the symbolic link:**

```bash
sudo unlink /etc/resolv.conf
```

This removes the link without deleting the target file. The system now has no `/etc/resolv.conf` until we create a new one.

<br>

<img width="382" height="20" alt="imagen" src="https://github.com/user-attachments/assets/596162ac-cb9f-49ff-94d7-e7bc392f9728" />

<br>

**Step 4.2 — Create a new static resolv.conf file:**

```bash
sudo nano /etc/resolv.conf
```

Inside the file, add the domain's nameserver (which is this server itself on the internal interface) and the search domain so that short hostnames resolve automatically:

```
nameserver 1.1.1.1
search lab12.lan
```

<br>

The screenshot below shows the completed file with the correct DNS entries:

<br>

<img width="936" height="70" alt="image" src="https://github.com/user-attachments/assets/82e8aef7-1c60-4078-b748-0c8523dafaa8" />


<br>

**Step 4.3 — Make the file immutable:**

```bash
sudo chattr +i /etc/resolv.conf
```

The `+i` flag (immutable) prevents any user, service, or process from modifying, renaming, or deleting this file — even as root. This guarantees our DNS configuration survives across reboots and network events.

> To undo the immutable flag in the future (e.g., to update the nameserver), run: `sudo chattr -i /etc/resolv.conf`

<br>

<img width="407" height="28" alt="imagen" src="https://github.com/user-attachments/assets/772b9b80-3d01-4288-8f97-1aaa1ab82880" />

<br>

We can verify the immutable flag is set by running `lsattr /etc/resolv.conf`. The output will show `----i---------` in the attributes column, confirming the file is now protected:

<br>

<img width="674" height="234" alt="imagen" src="https://github.com/user-attachments/assets/d2ea2c58-ae52-4299-82a5-d5eac2c8902e" />

<br>

---

## 📦 Step 5 — Install Samba and All Required Packages

This is the largest installation step of Sprint 1. We install Samba itself along with a complete set of supporting packages that cover every role the Domain Controller needs to fulfill: authentication (Kerberos), user/group resolution (Winbind), DNS management (dnsutils), time synchronization (Chrony), and filesystem access controls (ACL).

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 \
  krb5-config krb5-user dnsutils chrony net-tools
```

Here is what each package group provides:

| Package | Purpose |
|---------|---------|
| `samba` | The core Samba server — provides SMB and AD functionality |
| `samba-dsdb-modules` | Directory Services Database modules required for AD DC mode |
| `samba-vfs-modules` | Virtual Filesystem modules for ACL and extended attribute support |
| `smbclient` | Command-line SMB client for testing shares and connectivity |
| `winbind` | Maps Windows domain users/groups to Linux UIDs/GIDs |
| `libpam-winbind` | PAM module for authenticating Linux logins against AD |
| `libnss-winbind` | NSS module so the OS can resolve AD users and groups |
| `libpam-krb5` | PAM module for Kerberos-based authentication |
| `krb5-config / krb5-user` | Kerberos client configuration and command-line tools |
| `dnsutils` | DNS query tools (host, dig, nslookup) for verification |
| `chrony` | NTP time synchronization daemon |
| `acl / attr` | Linux filesystem ACL and extended attribute tools |
| `net-tools` | Classic network tools (ifconfig, netstat) |

> During the installation, you will be prompted interactively to enter the **Kerberos realm**. Type it in **ALL UPPERCASE** — for example: `LAB12.LAN`. This must match exactly what you will use when provisioning Samba. Entering it in lowercase will cause Kerberos authentication to fail.

<br>

The apt package manager begins resolving and downloading all dependencies:

<br>

<img width="797" height="40" alt="imagen" src="https://github.com/user-attachments/assets/74169420-b712-4ac3-87f7-0ebb8f2ca49d" />

<br>

The Kerberos realm configuration prompt appears during the installation process. Enter `LAB12.LAN` when asked for the default realm:

<br>

<img width="766" height="222" alt="imagen" src="https://github.com/user-attachments/assets/814abeca-9769-4d4e-88b5-cb9654f418b9" />

<br>

Enter the KDC (Key Distribution Center) hostname — this is the FQDN of our Domain Controller itself:

<br>

<img width="719" height="177" alt="imagen" src="https://github.com/user-attachments/assets/10b47588-1973-4418-bb87-273d9ca4d8d5" />

<br>

Enter the administrative server for the Kerberos realm — again, this is the DC's FQDN:

<br>

<img width="721" height="180" alt="imagen" src="https://github.com/user-attachments/assets/f9cc1d19-d2c8-4180-92ca-478be067531e" />

<br>

The Samba packages are downloaded and extracted. This may take a few minutes depending on network speed:

<br>

<img width="801" height="183" alt="imagen" src="https://github.com/user-attachments/assets/a93d94f4-6e17-4ccc-8af1-0ba59aab0c09" />

<br>

Winbind and related PAM/NSS integration modules are installed:

<br>

<img width="802" height="126" alt="imagen" src="https://github.com/user-attachments/assets/303dc108-ff3a-42bf-a8e3-03280eb874c8" />

<br>

The installation completes successfully with no errors:

<br>

<img width="796" height="68" alt="imagen" src="https://github.com/user-attachments/assets/62c396aa-772e-42d2-b5ec-b89512604af5" />

<br>

Before starting the service, we stop and disable the three Samba daemons that are used in member-server mode (`smbd`, `nmbd`, and `winbind`). On a Domain Controller, we only need `samba-ad-dc`. Running both at the same time will cause port conflicts:

<br>

<img width="578" height="39" alt="imagen" src="https://github.com/user-attachments/assets/faaa0ea9-a377-43bd-8c2c-7582223b0b7f" />

<br>

We also unmask `samba-ad-dc`, which is masked by default to prevent accidental startup before provisioning:

<br>

<img width="635" height="116" alt="imagen" src="https://github.com/user-attachments/assets/fc49ebbd-a635-4b78-bd18-c135e22feee5" />

<br>

We back up the default `/etc/samba/smb.conf` file, as the provisioning step will generate a new one. If the existing file is present, Samba's provisioning tool will refuse to proceed:

<br>

<img width="515" height="28" alt="imagen" src="https://github.com/user-attachments/assets/7419dc0d-2085-4af4-ad25-f76d9b3a06cd" />

<br>

Now we run the provisioning command. This is the central step that creates the AD database, configures DNS zones, generates Kerberos keys, and writes the final `smb.conf`:

<br>

<img width="590" height="22" alt="imagen" src="https://github.com/user-attachments/assets/8e86ed15-e96c-49a4-9be7-f273cfba777e" />

<br>

**Now we start the Samba AD DC service and check its status:**

```bash
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```

<br>

The service starts and reports `active (running)`. The status output also shows the Samba version and lists the active sub-processes (smbd, winbindd, etc.):

<br>

<img width="425" height="43" alt="imagen" src="https://github.com/user-attachments/assets/bf81ba96-adbf-41c5-98e7-aae772bca9ca" />

<br>

A more detailed status view confirms all sub-processes are running and no errors appear in the journal:

<br>

<img width="805" height="578" alt="imagen" src="https://github.com/user-attachments/assets/cf3a6fef-17a1-4848-b401-f5a1131e90be" />

<br>

---

## 🕐 Step 6 — Time Synchronization (Chrony)

Time synchronization is not optional in a Kerberos-based environment — it is a hard requirement. Kerberos uses timestamps as part of its security model to prevent replay attacks. If the clock difference between the DC and any client exceeds **5 minutes**, authentication tickets will be rejected and logins will fail with cryptic errors.

We use **Chrony** as our NTP daemon because it integrates natively with Samba's NTP signing socket, allowing domain-joined clients to verify time synchronization is coming from a trusted source.

<br>

First, we need to change the ownership and permissions on the Samba NTP socket directory so that Chrony can access it:

<br>

<img width="551" height="36" alt="imagen" src="https://github.com/user-attachments/assets/679df16d-dcf3-4dc0-a57d-f90c370f1872" />

<br>

This grants the `_chrony` group read access to the NTP signing socket that Samba creates at `/var/lib/samba/ntp_signd/`. Without this, Chrony will start but won't be able to sign NTP responses for domain clients.

<br>

**Open the Chrony configuration file:**

```bash
nano /etc/chrony/chrony.conf
```

We add three important lines:
- `bindcmdaddress` — Binds Chrony's command interface to the internal AD network IP
- `allow` — Permits NTP requests from all hosts on the `192.168.30.0/24` subnet
- `ntpsigndsocket` — Points Chrony to Samba's signing socket so responses are cryptographically verified

The configuration file after our changes:

<br>

<img width="802" height="547" alt="imagen" src="https://github.com/user-attachments/assets/c45bda87-c6a5-475a-b388-c3a6e9b1752a" />

<br>

**Restart Chrony to apply the new configuration:**

```bash
sudo systemctl restart chronyd
```

<br>

**Verify the service is running and has no errors:**

```bash
sudo systemctl status chronyd
```

<br>

The status output shows Chrony is active and has successfully connected to upstream NTP sources:

<br>

<img width="403" height="44" alt="imagen" src="https://github.com/user-attachments/assets/b5083434-c798-4624-af01-14ffc8df1922" />

<br>

A detailed status view shows the NTP sources Chrony is synchronized to, the current time offset, and the number of clients being served:

<br>

<img width="802" height="352" alt="imagen" src="https://github.com/user-attachments/assets/9ae21ca1-d1d7-4ce7-b330-76e6b7dfc0b1" />

<br>

---

## ✅ Step 7 — Verify the Domain

With all services running, we now perform a comprehensive set of verification checks. This confirms that every layer of the Active Directory stack — DNS, Kerberos, LDAP, and SMB — is working correctly before we move on to connecting clients.

<br>

**7.1 — Verify DNS A records**

The domain itself must resolve to the DC's IP address:

```bash
host -t A lab12.lan
```

<br>

A successful response returns the IP address of the domain controller, confirming that Samba's internal DNS is resolving domain-level queries:

<br>

<img width="310" height="54" alt="imagen" src="https://github.com/user-attachments/assets/247487de-c970-45fb-9eba-92fb4864e103" />

<br>

The server's own FQDN must also resolve:

```bash
host -t A ls12.lab12.lan
```

<br>

This confirms that the host (A) record for the DC's FQDN was created correctly during provisioning:

<br>

<img width="346" height="57" alt="imagen" src="https://github.com/user-attachments/assets/ea8f1764-f027-491b-8985-17e6bb6a413d" />

<br>

**7.2 — Verify Kerberos and LDAP SRV records**

Service records (SRV) tell clients where to find key AD services. These records are created automatically during provisioning. If they are missing, clients will not be able to discover the DC:

```bash
host -t SRV _kerberos._udp.lab12.lan
```

<br>

The response should show the Kerberos service pointing to our DC at port 88:

<br>

<img width="515" height="38" alt="imagen" src="https://github.com/user-attachments/assets/302eea64-0d36-40fb-b392-70df9eec6fde" />

<br>

```bash
host -t SRV _ldap._tcp.clockwork.local
```

<br>

The LDAP SRV record confirms clients can locate the directory service at port 389:

<br>

<img width="491" height="41" alt="imagen" src="https://github.com/user-attachments/assets/2d2826f8-0a65-426a-8e34-a5bebe87294e" />

<br>

**7.3 — Verify SMB shares**

The presence of the default AD shares (`netlogon`, `sysvol`, and `IPC$`) confirms that Samba is running in Domain Controller mode:

```bash
smbclient -L clockwork.local -N
```

<br>

<img width="556" height="161" alt="imagen" src="https://github.com/user-attachments/assets/1d5e33aa-b4e7-43ef-9471-98e892ab747c" />

<br>

**7.4 — Verify Kerberos authentication**

Request a Kerberos Ticket-Granting Ticket (TGT) for the domain administrator. If this succeeds, the entire Kerberos authentication chain is working:

```bash
kinit administrator@LAB12.LAN
```

<br>

<img width="603" height="58" alt="imagen" src="https://github.com/user-attachments/assets/65c2f14b-cfbd-412e-a0db-99299a01f084" />

<br>

Run `klist` to display the cached tickets and confirm the TGT was issued with the correct realm and expiry time:

<br>

<img width="541" height="121" alt="imagen" src="https://github.com/user-attachments/assets/a5c53eb9-445a-4918-b86b-3d110ae71864" />

<br>

**7.5 — Log in via SMB**

Using the cached Kerberos ticket, authenticate to the `netlogon` share. Success here confirms that Kerberos, SMB, and the AD database are all communicating correctly:

```bash
sudo smbclient //localhost/netlogon -U 'administrator'
```

<br>

<img width="591" height="78" alt="imagen" src="https://github.com/user-attachments/assets/e43de933-1169-4b0c-8d7e-92bc33ab8730" />

<br>

**7.6 — Set administrator password**

Set a strong password for the domain administrator account. This account has full control over the domain, so it must be protected:

```bash
sudo samba-tool user setpassword administrator
```

<br>

<img width="710" height="40" alt="imagen" src="https://github.com/user-attachments/assets/b3070cd2-72e4-494f-bd14-5ac07cbbdc5a" />

<br>

**7.7 — Validate Samba configuration**

`testparm` reads the `smb.conf` file and checks it for syntax errors and configuration warnings. It also prints a summary of all active settings:

```bash
testparm
```

<br>

<img width="450" height="666" alt="imagen" src="https://github.com/user-attachments/assets/b18f299c-b956-4958-a56d-4489de8dbbb8" />

<br>

**7.8 — Check domain functional level**

The functional level determines which AD features are available. It should report at least `Windows 2008` for a modern Samba AD DC:

```bash
sudo samba-tool domain level show
```

<br>

<img width="500" height="107" alt="imagen" src="https://github.com/user-attachments/assets/6b95ef8c-59e5-4d96-870c-7acd9c31af06" />

<br>

Sprint 1 is now complete. The server is a fully operational Samba Active Directory Domain Controller, ready to accept client connections.

<br>

---

<br>

# 💻 SPRINT 2 — Linux and Windows Client Integration

> **Goal:** Join Linux desktop and server machines to the `lab12.lan` Samba Active Directory domain. This sprint walks through installing the required client-side packages, configuring Kerberos and Winbind, modifying PAM and NSS to authenticate domain users at the OS level, creating Organizational Units and user accounts, and enforcing domain-wide security policies through Group Policy Objects (GPOs).

<br>

---

## 🔄 Step 1 — Update the System

Before installing any packages on the client machine, we bring the system up to date. This ensures we install the latest available versions of all packages and avoids potential dependency conflicts caused by outdated package lists.

```bash
sudo apt update
```

<br>

The package manager contacts the repositories and downloads the updated index:

<br>

<img width="718" height="150" alt="imagen" src="https://github.com/user-attachments/assets/6f4f6456-baf5-4cd4-997c-9f9a413a53fa" />

<br>

Once the index is refreshed, we upgrade any installed packages that have newer versions available:

```bash
sudo apt upgrade
```

<br>

The upgrade process downloads and installs all available updates. On a fresh install this is usually quick, but on an older system it may take a few minutes:

<br>

<img width="1190" height="362" alt="imagen" src="https://github.com/user-attachments/assets/8a12c418-ac51-4d48-aa86-faed7b35be2c" />

<br>

---

## 🔐 Step 2 — Install and Verify SSH

We install the OpenSSH server so that the client machine can be managed remotely. In a real environment, you would typically manage all machines remotely via SSH rather than working directly at the console. This is also useful for copying and pasting configuration files between machines.

```bash
sudo apt-get install ssh
```

<br>

<img width="708" height="185" alt="imagen" src="https://github.com/user-attachments/assets/8745d1ea-bfbc-4469-8254-554181492278" />

<br>

After installation, we verify the SSH service is active and running. The `systemctl status` output should show `active (running)` in green:

```bash
sudo systemctl status ssh
```

<br>

The output below confirms SSH is listening on port 22 and is set to start automatically on boot:

<br>

<img width="717" height="303" alt="imagen" src="https://github.com/user-attachments/assets/c11c775c-0c94-4d51-851d-04b056c48ee0" />

<br>

---

## ⚙️ Step 3 — Configure the Linux Client Machine

### 3.1 — Set the Hostname

Just like the Domain Controller, the client machine needs a unique, meaningful hostname. This hostname will become the machine's computer account name in Active Directory. We set it to `lc12` (Linux Client 12):

```bash
sudo hostnamectl set-hostname lc12
hostname -f
```

The `hostname -f` command prints the Fully Qualified Domain Name to verify the change took effect correctly:

<br>

<img width="460" height="72" alt="imagen" src="https://github.com/user-attachments/assets/25069b5f-c4d4-47de-9ac9-8b4045a32060" />

<br>

### 3.2 — Configure /etc/hosts

Before the client has been configured to use the domain's DNS server, it needs to know where to find the Domain Controller. We add the DC's FQDN and IP to `/etc/hosts` as a fallback resolution method:

```bash
nano /etc/hosts
```

<br>

The command opens the file in the nano editor:

<br>

<img width="347" height="23" alt="imagen" src="https://github.com/user-attachments/assets/3ee4ed8b-172b-4ece-8d48-652026784e09" />

<br>

We add a line for the Domain Controller, mapping its IP to its FQDN and short name. The result should look similar to this — the client's own hostname is also mapped, which prevents warnings from some commands:

<br>

<img width="716" height="243" alt="imagen" src="https://github.com/user-attachments/assets/c05ac32f-439f-407f-9eb9-3d8ac8382dce" />

<br>

### 3.3 — Test Connectivity to the Domain

Before proceeding with any installation, we confirm the client can reach the Domain Controller over the network. A failed ping here would indicate a routing or firewall issue that must be resolved first:

```bash
ping lab12.lan
```

<br>

The ping resolves `lab12.lan` to the DC's IP and receives replies, confirming both DNS resolution and basic network reachability:

<br>

<img width="663" height="152" alt="image" src="https://github.com/user-attachments/assets/845a1d93-cd53-4fb8-860e-e07a93489425" />

<br>

---

## 🕐 Step 4 — Synchronize Time with the Domain Controller

Time synchronization must be established before attempting to authenticate via Kerberos. If the client's clock differs from the DC by more than 5 minutes, every authentication attempt will fail with a `Clock skew too great` error.

We install `ntpdate`, a lightweight tool for one-shot time synchronization:

```bash
sudo apt-get install ntpdate
```

<br>

<img width="713" height="427" alt="imagen" src="https://github.com/user-attachments/assets/7dc08504-a255-4d04-8e5c-2c6236345ab2" />

<br>

We first query the DC's NTP service without making any changes, to see the current offset between the client and the DC:

```bash
sudo ntpdate -q lab12.lan
```

Then we actually synchronize the client clock to match the DC:

```bash
sudo ntpdate lab12.lan
```

<br>

The output shows the time offset and confirms the clock has been adjusted:

<br>

<img width="767" height="94" alt="imagen" src="https://github.com/user-attachments/assets/7d76e4a1-b8c6-4ea3-b908-e80dd6eaa96f" />

<br>

---

## 📦 Step 5 — Install Domain Integration Packages

Now we install the full set of packages required for the client to join and authenticate against the Samba AD domain:

```bash
sudo apt-get install samba krb5-config krb5-user winbind libpam-winbind libnss-winbind
```

Here is what each package does on the client side:

| Package | Purpose on Client |
|---------|------------------|
| `samba` | Provides the `net` command-line tool used to join the domain |
| `krb5-config` | Sets up the Kerberos client configuration (`/etc/krb5.conf`) |
| `krb5-user` | Provides the `kinit` and `klist` tools for testing Kerberos |
| `winbind` | Maps domain users and groups to local Linux user IDs |
| `libpam-winbind` | Lets PAM authenticate local logins using domain credentials |
| `libnss-winbind` | Lets the OS resolve domain users/groups via `getent` |

<br>

<img width="882" height="23" alt="imagen" src="https://github.com/user-attachments/assets/3c583e93-afa7-4d85-b9e2-b92c7ff5609b" />

<br>

The installer again prompts for the Kerberos realm. Enter `LAB12.LAN` in uppercase:

<br>

<img width="768" height="199" alt="imagen" src="https://github.com/user-attachments/assets/b702a6f7-29b0-49a7-8575-866cc959bcf5" />

<br>

Enter the KDC hostname (the FQDN of our Domain Controller):

<br>

<img width="617" height="196" alt="imagen" src="https://github.com/user-attachments/assets/c52ba31b-8a79-4cc6-a450-1f96b6dba191" />

<br>

Enter the administrative Kerberos server (again, our DC's FQDN):

<br>

<img width="616" height="184" alt="imagen" src="https://github.com/user-attachments/assets/7c6d9a3c-66ec-4238-b29b-788c83a1c6fe" />

<br>

---

## 🔑 Step 6 — Kerberos Authentication Test

Before joining the domain, we verify that the Kerberos client is configured correctly by manually requesting a Ticket-Granting Ticket (TGT) for the domain administrator. This is a standalone test that does not require any domain membership:

```bash
kinit administrator@LAB12.LAN
```

<br>

If Kerberos is configured correctly, the command will prompt for the password and return silently on success:

<br>

<img width="594" height="57" alt="imagen" src="https://github.com/user-attachments/assets/bbe0e8d6-a145-4305-9334-7fcaffa4f749" />

<br>

We list the cached Kerberos tickets to confirm a TGT was granted and check its validity period:

```bash
klist
```

<br>

The ticket cache shows the TGT for the `administrator@LAB12.LAN` principal, with its issue time and expiration time. Kerberos tickets are valid for 10 hours by default:

<br>

<img width="520" height="119" alt="imagen" src="https://github.com/user-attachments/assets/0405eebe-9aeb-4fe1-9341-9ea3aec70157" />

<br>

---

## 📝 Step 7 — Configure Samba as a Domain Member (smb.conf)

The client machine's Samba configuration must tell it to operate as a **domain member** (`security = ADS`) rather than a standalone server. We start by preserving the default configuration:

```bash
mv /etc/samba/smb.conf /etc/samba/smb.conf.initial
```

<br>

<img width="491" height="20" alt="imagen" src="https://github.com/user-attachments/assets/1e1fde2e-991e-472c-b80e-cea68e656273" />

<br>

We create a fresh, minimal `smb.conf` file with only the settings needed for domain membership:

```bash
nano /etc/samba/smb.conf
```

<br>

<img width="422" height="17" alt="imagen" src="https://github.com/user-attachments/assets/6f795203-931c-4f2b-ac78-7b1a0f005ff3" />

<br>

Add the following configuration, which tells Samba to authenticate against Active Directory, sets up Winbind for user and group mapping, and enables ACL support:

```ini
[global]
    workgroup = LAB12
    realm = LAB12.LAN
    netbios name = ud101
    security = ADS
    dns forwarder = 192.168.30.40

idmap config * : backend = tdb
idmap config *:range = 50000-1000000

    template homedir = /home/%D/%U
    template shell = /bin/bash
    winbind use default domain = true
    winbind offline logon = false
    winbind nss info = rfc2307
    winbind enum users = yes
    winbind enum groups = yes

    vfs objects = acl_xattr
    map acl inherit = Yes
    store dos attributes = Yes
```

Key settings explained:
- **`security = ADS`** — Instructs Samba to authenticate via Active Directory (Kerberos + LDAP)
- **`template homedir = /home/%D/%U`** — Creates home directories like `/home/LAB12/alice` for domain users
- **`winbind use default domain = true`** — Users can log in as `alice` instead of `LAB12\alice`
- **`idmap config *:range`** — Defines the UID/GID range reserved for domain accounts, avoiding conflicts with local users

<br>

<img width="732" height="386" alt="imagen" src="https://github.com/user-attachments/assets/a1ad0024-b608-4669-b422-be1df542a250" />

<br>

---

## 🔄 Step 8 — Manage Samba Services on the Client

On a client machine (as opposed to the DC), we run `smbd` and `nmbd` for file sharing, but we do **not** run `samba-ad-dc`. We restart the file-sharing daemons to apply the new `smb.conf`:

```bash
sudo systemctl restart smbd nmbd
```

<br>

<img width="448" height="22" alt="imagen" src="https://github.com/user-attachments/assets/f31b2edb-98b2-4852-8404-7b7d35c33d5e" />

<br>

We explicitly stop `samba-ad-dc` since it must not run on a client machine — it is only needed on the Domain Controller itself:

```bash
sudo systemctl stop samba-ad-dc
```

<br>

<img width="443" height="23" alt="imagen" src="https://github.com/user-attachments/assets/f3d6f5b9-de2e-4c7e-a46e-1dccca5e6ce2" />

<br>

We enable `smbd` and `nmbd` to start on every boot so the file-sharing services are always available:

```bash
sudo systemctl enable smbd nmbd
```

<br>

The systemctl enable command creates the symbolic links in the systemd startup directories:

<br>

<img width="789" height="115" alt="imagen" src="https://github.com/user-attachments/assets/40e8baea-f2ca-4ccf-8d62-90a8cb3c05d8" />

<br>

---

## 🔗 Step 9 — Join the Domain

This is the pivotal step of Sprint 2. The `net ads join` command communicates with the Domain Controller over Kerberos, creates a **computer account** for this machine in AD, and generates a machine secret that will be used for future authentications:

```bash
sudo net ads join -U administrator
```

<br>

You will be prompted for the administrator password. On success, you should see the message `Joined 'LCXX' to dns domain 'lab12.lan'`, confirming the machine account was created and the machine is now a member of the domain:

<br>

<img width="508" height="26" alt="imagen" src="https://github.com/user-attachments/assets/3b2bb6cc-2744-430e-a2d7-ccd375f9a1bc" />

<br>

**Verify from the Domain Controller** that the computer account was created. We can list all computer objects in the domain using `samba-tool`:

```bash
sudo samba-tool computer list
```

<br>

The client machine's name should now appear in the list alongside the DC itself:

<br>

<img width="378" height="69" alt="imagen" src="https://github.com/user-attachments/assets/a01d1e3c-f5d4-4ff6-86ee-fa2bb2c84eaa" />

<br>

**Verify from the client side** that the domain join is valid. This command performs a quick sanity check against the AD server without requiring credentials:

```bash
net ads testjoin
```

<br>

The expected output is simply `Join is OK`, confirming the machine's credentials are valid and the join is still active:

<br>

<img width="366" height="44" alt="imagen" src="https://github.com/user-attachments/assets/a041644d-4ec4-4a4c-8994-9337983a8d8c" />

<br>

---

## 🛠️ Step 10 — Configure AD Account Authentication

### 10.1 — Edit the NSS Configuration

The **Name Service Switch (NSS)** configuration tells the operating system where to look up users, groups, and other system information. By default, it only consults local files (`/etc/passwd`, `/etc/group`). We need to add `winbind` as an additional source so domain users become visible to the OS:

```bash
sudo nano /etc/nsswitch.conf
```

Find the lines starting with `passwd:` and `group:`, and add `winbind` at the end of each:

```
passwd:   files systemd winbind
group:    files systemd winbind
```

<br>

<img width="425" height="28" alt="imagen" src="https://github.com/user-attachments/assets/b54fbe04-95e4-4777-ae2f-ee24d24b1a54" />

<br>

The full `nsswitch.conf` file after our modifications. The `winbind` entries in the `passwd` and `group` lines are what enable domain user resolution:

<br>

<img width="793" height="433" alt="imagen" src="https://github.com/user-attachments/assets/66a97ad6-0543-4690-a904-fe784cc8aa82" />

<br>

### 10.2 — Restart Winbind

After modifying NSS, we restart Winbind so it reconnects to the Domain Controller and picks up the new configuration:

```bash
sudo systemctl restart winbind
```

<br>

<img width="433" height="19" alt="imagen" src="https://github.com/user-attachments/assets/6bd46a2c-e4e6-4c6f-9b01-5192b10c9838" />

<br>

### 10.3 — Verify Domain User and Group Integration

`wbinfo` is the command-line tool for querying Winbind directly. We use it to verify that domain users and groups are visible to the client:

```bash
wbinfo -u    # List all domain users
wbinfo -g    # List all domain groups
```

<br>

The output lists all domain user accounts and groups retrieved from the DC. If this is empty, Winbind is not connecting to the domain successfully:

<br>

<img width="322" height="364" alt="imagen" src="https://github.com/user-attachments/assets/c7bcc3ed-fb1e-4cea-932d-955fc72ce5eb" />

<br>

We also verify using `getent`, which queries NSS and should now return domain accounts alongside local ones:

```bash
sudo getent passwd | grep administrator
```

<br>

The `getent` output for the administrator account shows a Unix-formatted entry with the domain-assigned UID:

<br>

<img width="517" height="33" alt="imagen" src="https://github.com/user-attachments/assets/95b19adc-a219-4149-b6e1-7b84199324b3" />

<br>

```bash
sudo getent group | grep 'domain admins'
```

<br>

The `Domain Admins` group is visible with its assigned GID:

<br>

<img width="493" height="36" alt="imagen" src="https://github.com/user-attachments/assets/29377847-b312-491b-b851-243d74657580" />

<br>

```bash
id administrator
```

<br>

The `id` command shows the administrator's UID, primary GID, and all supplementary groups — including `Domain Admins`:

<br>

<img width="795" height="75" alt="imagen" src="https://github.com/user-attachments/assets/e54074c0-8cd9-4c56-baf7-712fb0b5cf42" />

<br>

---

## 👥 Step 11 — Configure PAM for Domain Login

**PAM (Pluggable Authentication Modules)** is the Linux subsystem responsible for authentication. We configure it to allow domain users to log in and to automatically create their home directory on first login.

```bash
sudo pam-auth-update
```

<br>

<img width="350" height="21" alt="imagen" src="https://github.com/user-attachments/assets/af2a2bf9-b546-4041-9e0f-0dd874cf5000" />

<br>

An interactive menu appears. We select the Winbind and automatic home directory options using the spacebar, then confirm with Enter:

<br>

<img width="757" height="445" alt="imagen" src="https://github.com/user-attachments/assets/9550ef33-07bc-4535-a19a-4452f580b0a6" />

<br>

We also manually add home directory auto-creation to the PAM account file:

```bash
sudo nano /etc/pam.d/common-account
```

<br>

<img width="487" height="24" alt="imagen" src="https://github.com/user-attachments/assets/d62dd0bb-be12-4df5-a74a-24218b4482de" />

<br>

Add this line at the very end of the file. It creates the user's home directory from `/etc/skel` on first login with secure permissions:

```
session required pam_mkhomedir.so skel=/etc/skel/ umask=0022
```

<br>

The full `common-account` file after adding the `pam_mkhomedir` line at the bottom:

<br>

<img width="786" height="532" alt="imagen" src="https://github.com/user-attachments/assets/cc293e06-9297-4a46-b06e-c7eb03e7379c" />

<br>

**Test terminal login with a domain account:**

```bash
su administrator
```

<br>

The `su` command switches to the domain administrator account. If PAM and Winbind are correctly configured, you will be prompted for the domain password and dropped into a shell as the domain user:

<br>

<img width="314" height="56" alt="imagen" src="https://github.com/user-attachments/assets/09dc6311-0fe8-4b0a-9486-51756e6f3be9" />

<br>

**Test graphical (GUI) login with domain credentials:**

The following screenshots show the GUI login flow on Ubuntu Desktop. The login screen now accepts domain usernames. We enter the domain administrator credentials:

<br>

<img width="376" height="304" alt="imagen" src="https://github.com/user-attachments/assets/f93ecbd0-f72b-45df-8b78-7bd7f78fc32b" />

<br>

The password prompt accepts domain credentials, and the session is created:

<br>

<img width="368" height="228" alt="imagen" src="https://github.com/user-attachments/assets/2201485a-1c2e-4724-939d-57872451b735" />

<br>

The home directory `/home/LAB12/administrator` is automatically created on first login:

<br>

<img width="489" height="74" alt="imagen" src="https://github.com/user-attachments/assets/1405ea13-b647-4546-9346-570b5dd731b6" />

<br>

The desktop loads successfully as the domain administrator user. Notice the username displayed in the top-right corner reflects the domain account:

<br>

<img width="613" height="483" alt="imagen" src="https://github.com/user-attachments/assets/8be61fb0-a733-41c2-a4b2-7a765df495f0" />

<br>

Additional GUI verification — the file manager opens and the home directory is populated with default skeleton files:

<br>

<img width="609" height="481" alt="imagen" src="https://github.com/user-attachments/assets/5ea4733b-6137-4e58-b857-f14c1d1ab896" />

<br>

System settings reflect the domain user identity:

<br>

<img width="612" height="481" alt="imagen" src="https://github.com/user-attachments/assets/70fd1e12-8126-4fb4-834f-8b3f26bb1c26" />

<br>

The user's desktop preferences are initialized from the skeleton configuration:

<br>

<img width="611" height="477" alt="imagen" src="https://github.com/user-attachments/assets/a82d7d9a-8a55-4cd6-b5c3-d98ce2220a9b" />

<br>

User account details visible in system settings:

<br>

<img width="407" height="482" alt="imagen" src="https://github.com/user-attachments/assets/d8c410d4-c0b4-4dfb-9b68-5cd976326a1d" />

<br>

---

## 👤 Step 12 — Create Users and Groups in Active Directory

Now we populate the domain with a realistic structure of user accounts, security groups, and Organizational Units (OUs). This mirrors how a real organization would structure its directory.

<br>

### 12.1 — Create User Accounts

We create three users — one assigned to IT, and two assigned to the Students OU:

```bash
sudo samba-tool user create Alice --userou="OU=IT_Department"
```

<br>

Alice is created successfully. Her account is placed in the IT_Department OU:

<br>

<img width="663" height="77" alt="imagen" src="https://github.com/user-attachments/assets/23515350-6a24-4771-aafb-fa9e324e0f74" />

<br>

```bash
sudo samba-tool user create Bob --userou="OU=Students"
```

<br>

Bob is created and placed in the Students OU:

<br>

<img width="601" height="70" alt="imagen" src="https://github.com/user-attachments/assets/44ad168f-9be1-4157-9505-365f9fc04114" />

<br>

```bash
sudo samba-tool user create Charlie --userou="OU=Students"
```

<br>

Charlie is created alongside Bob in the Students OU:

<br>

<img width="626" height="68" alt="imagen" src="https://github.com/user-attachments/assets/5946eb66-0bab-4c26-b946-3d219d400e91" />

<br>

### 12.2 — Create Security Groups

Security groups are used to assign permissions to multiple users at once. Instead of granting permissions user by user, we grant them to a group and then add users to that group:

```bash
sudo samba-tool group add IT_Admins
```

<br>

<img width="435" height="25" alt="image" src="https://github.com/user-attachments/assets/551a9e87-754e-4e29-839c-983f36c265ad" />

<br>

```bash
sudo samba-tool group add Students
```

<br>

<img width="427" height="23" alt="image" src="https://github.com/user-attachments/assets/8cdadbf0-1d3b-4066-b87d-35a172025cc7" />

<br>

### 12.3 — Add Users to Groups

With groups and users created, we assign memberships:

```bash
sudo samba-tool group addmembers IT_Admins Alice
```

<br>

Alice is added to the IT_Admins group. She will now inherit any permissions assigned to that group:

<br>

<img width="542" height="27" alt="image" src="https://github.com/user-attachments/assets/982c6b7c-ad7d-4d5d-a216-e356fe25215e" />

<br>

```bash
sudo samba-tool group addmembers Students Bob,Charlie
```

<br>

Both Bob and Charlie are added to the Students group in a single command:

<br>

<img width="585" height="22" alt="image" src="https://github.com/user-attachments/assets/70b59cc6-e834-4119-9cc0-ca415971a4c1" />

<br>

### 12.4 — Create the Organizational Unit (OU) Hierarchy

OUs are the building blocks of directory structure in Active Directory. They act as folders that contain users, groups, computers, and even other OUs. A well-designed OU structure makes it easy to delegate administrative control and apply GPOs to specific departments:

```bash
sudo samba-tool ou add "OU=IT_Department,DC=lab12,DC=lan"
```

<br>

The IT_Department OU is created at the root of the domain:

<br>

<img width="628" height="36" alt="image" src="https://github.com/user-attachments/assets/6fe6093e-7216-42ad-8c34-1d458ac5f202" />

<br>

```bash
sudo samba-tool ou add "OU=Students,DC=lab12,DC=lan"
```

<br>

<img width="593" height="32" alt="image" src="https://github.com/user-attachments/assets/c8d3c92f-49c6-4597-9eaf-11d89a28683c" />

<br>

```bash
sudo samba-tool ou add "OU=HR_Department,DC=lab12,DC=lan"
```

<br>

<img width="632" height="35" alt="image" src="https://github.com/user-attachments/assets/66847513-2859-458c-96dd-81113129f2d1" />

<br>

---

## 🔒 Step 13 — Group Policy Objects (GPOs)

Group Policies allow administrators to enforce security settings and restrictions across all users and machines in the domain from a single, central location. We implement two key policies: a password complexity policy and an account lockout policy.

<br>

### 13.1 — Password Policy

We set a domain-wide minimum password length of 8 characters and require passwords to meet complexity requirements (mix of uppercase, lowercase, numbers, and symbols):

```bash
sudo samba-tool domain passwordsettings set --min-pwd-length=8 --complexity=on
```

<br>

The command reports success and shows the updated policy values:

<br>

<img width="784" height="130" alt="imagen" src="https://github.com/user-attachments/assets/4d484726-1ff2-42cd-a799-8e92e9b7e8e8" />

<br>

### 13.2 — Account Lockout Policy

The account lockout policy protects against brute-force password attacks. After 3 failed login attempts, the account is locked for 5 minutes — enough to stop automated attacks while minimizing disruption for legitimate users who mistype their password.

**Set the lockout threshold to 3 failed attempts:**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-threshold=3
```

<br>

<img width="742" height="55" alt="imagen" src="https://github.com/user-attachments/assets/1585bde3-d978-49ba-ace5-3532cb3874c3" />

<br>

**Set the lockout duration to 5 minutes.** During this period, the account cannot be used even with the correct password:

```bash
sudo samba-tool domain passwordsettings set --account-lockout-duration=5
```

<br>

<img width="731" height="53" alt="imagen" src="https://github.com/user-attachments/assets/f3d5a0e6-1a31-4a10-b6f2-5c880cb3b099" />

<br>

**Set the reset counter to 5 minutes.** This means the failed-attempt counter resets after 5 minutes of no failed attempts, so a user who makes 2 mistakes and then waits 5 minutes starts fresh:

```bash
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=5
```

<br>

<img width="731" height="51" alt="imagen" src="https://github.com/user-attachments/assets/7880d848-5a6c-4534-a8c2-bc1a031c7efb" />

<br>

### 13.3 — Policy Verification and Testing

**Account Lockout Test — Brute Force Simulation:**

To confirm the lockout policy is active, we deliberately attempted to log in as **Bob** with an incorrect password. The steps were:

1. Entered Bob's username with a wrong password on the login screen.
2. Repeated the failed attempt 3 times to hit the threshold.
3. On the 4th attempt, the system refused to show a password prompt and displayed a lockout error:

<br>

<img width="796" height="719" alt="imagen" src="https://github.com/user-attachments/assets/82c44e16-1e08-4aea-97e2-f7cdc99302b3" />

<br>

4. After waiting exactly 5 minutes, Bob's account was automatically unlocked and a login with the correct password succeeded — confirming both the lockout duration and the auto-unlock mechanism are working correctly.

<br>

**Verify the OU structure is intact:**

```bash
sudo samba-tool ou list
```

<br>

All three OUs are listed, confirming they were created successfully:

<br>

<img width="358" height="104" alt="imagen" src="https://github.com/user-attachments/assets/6ffa1252-2d2b-4f6d-9c2f-05d5d44005c0" />

<br>

**Audit user placement using Distinguished Names:**

The Distinguished Name (DN) uniquely identifies an object's full path in the directory tree. We verify that Alice was correctly placed under the IT_Department OU:

```bash
sudo samba-tool user show Alice | grep dn
```

<br>

The DN output confirms Alice's full path in the AD hierarchy:

<br>

<img width="507" height="106" alt="imagen" src="https://github.com/user-attachments/assets/e89984d2-9745-4b93-9303-36af4480e04e" />

<br>

**Validate group memberships:**

```bash
sudo samba-tool group listmembers IT_Admins
sudo samba-tool group listmembers Students
```

<br>

The output confirms Alice is in IT_Admins, and both Bob and Charlie are in Students:

<br>

<img width="591" height="86" alt="imagen" src="https://github.com/user-attachments/assets/ae88b763-ca0b-4359-b2f5-72bc19de5bca" />

<br>

**Display the full security policy summary:**

```bash
sudo samba-tool domain passwordsettings show
```

<br>

This shows all active password and lockout policy values in one view — a useful audit trail:

<br>

<img width="513" height="205" alt="imagen" src="https://github.com/user-attachments/assets/49879fd8-6c84-42f0-87e1-858cfdb4bffc" />

<br>

---

<br>

# 💾 SPRINT 3 — Storage, Shares & Permissions

> **Goal:** Expand the Domain Controller's storage by adding a dedicated data disk, create a department-based directory hierarchy on that disk, configure Samba network shares with group-based access controls for each department, install ACL support for fine-grained permission management, and learn the key Linux tools for monitoring and managing system processes.

<br>

---

## 🔍 Connectivity Verification

Before making any changes to the server, we verify that the domain is still healthy and the DC is responding correctly.

**Check the domain status directly:**

```bash
sudo samba-tool domain info 127.0.0.1
```

<br>

The output shows the domain name, realm, and DC role. This confirms the AD services are running correctly:

<br>

<img width="458" height="148" alt="image" src="https://github.com/user-attachments/assets/ae1e256c-0dde-47e5-83f4-61b10f8e999d" />

<br>

**Verify DNS resolution with a ping:**

```bash
ping -c 3 lab12.lan
```

<br>

Three successful ping replies confirm that DNS is resolving the domain name correctly and the network is reachable:

<br>

<img width="597" height="152" alt="image" src="https://github.com/user-attachments/assets/6c2d4c83-c52f-43b6-b225-90d6383f42aa" />

<br>

---

## 1. 🗄️ Storage Management

### Step 1.1 — Create the New Virtual Disk

In the hypervisor (VirtualBox or VMware), we add a new virtual hard disk to the Domain Controller VM. This disk will be dedicated exclusively to Samba file shares, separating data from the system disk for better performance and easier backups. A size of **20GB** is sufficient for a lab environment:

<br>

<img width="1025" height="595" alt="image" src="https://github.com/user-attachments/assets/361a78ba-7147-4173-ab4d-44d88ed749b7" />

<br>

### Step 1.2 — Identify the New Disk in the OS

After adding the disk in the hypervisor and booting the VM, Linux automatically detects the new disk. We use `lsblk` to list all block devices and confirm the new disk appeared:

```bash
lsblk
```

<br>

The output shows `sda` (the existing system disk) and the newly added `sdb` (20GB, unpartitioned). This confirms the hypervisor passed the disk to the VM successfully:

<br>

<img width="582" height="230" alt="image" src="https://github.com/user-attachments/assets/c4e68074-9f56-497b-b1a0-f4975e2638fb" />

<br>

### Step 1.3 — Create a Partition Table with fdisk

`fdisk` is the traditional Linux disk partitioning tool. We use it to initialize the new disk and create a single partition that spans the entire disk:

```bash
sudo fdisk /dev/sdb
```

Follow the interactive prompt in sequence:

| Command | What it does |
|---------|--------------|
| `n` | Start creating a new partition |
| `p` | Choose "primary" partition type |
| `1` | Set partition number to 1 |
| `Enter` | Accept the default first sector (beginning of disk) |
| `Enter` | Accept the default last sector (end of disk — uses full disk) |
| `w` | Write the partition table to disk and exit |

<br>

The screenshot below shows the full fdisk session. Note the confirmation message at the end that the partition table was written successfully:

<br>

<img width="677" height="397" alt="image" src="https://github.com/user-attachments/assets/78e3d80b-87b1-4671-bbd2-7cd360d7f362" />

<br>

### Step 1.4 — Format the Partition as EXT4

Samba requires the **EXT4** filesystem for two critical reasons: it natively supports **POSIX ACLs** (Access Control Lists) and **extended attributes** (xattrs), both of which Samba uses to store Windows permission data on Linux. FAT or NTFS-based filesystems are not suitable here.

```bash
sudo mkfs.ext4 /dev/sdb1
```

<br>

The formatter creates the filesystem, generates a UUID, writes the superblock, and allocates inodes. This UUID will be used to permanently mount the disk:

<br>

<img width="520" height="182" alt="image" src="https://github.com/user-attachments/assets/aaa52e39-9562-45f1-9d2f-fd68049fb510" />

<br>

### Step 1.5 — Configure a Permanent Mount Point

We want this disk to be automatically mounted at the same location every time the system boots. Using the disk's UUID (rather than its device path `/dev/sdb1`) makes the mount configuration resilient to disk order changes.

**Create the mount directory:**

```bash
sudo mkdir -p /srv/samba/Data
```

<br>

<img width="391" height="27" alt="image" src="https://github.com/user-attachments/assets/631bf1b6-dcb7-47c5-8f64-019291f3122c" />

<br>

**Get the UUID of the new partition:**

```bash
sudo blkid /dev/sdb1
```

<br>

The output shows the UUID, filesystem type, and label. Copy the UUID string — we will paste it into `/etc/fstab` in the next step:

<br>

<img width="802" height="55" alt="image" src="https://github.com/user-attachments/assets/43f930b2-e609-4df1-904d-330d894943a0" />

<br>

**Add the mount entry to /etc/fstab:**

```bash
sudo nano /etc/fstab
```

Add this line at the end of the file, replacing `your-uuid-here` with the UUID from the previous step:

```
UUID=your-uuid-here /srv/samba/Data ext4 user_xattr,acl,barrier=1 1 1
```

The mount options are important:
- `user_xattr` — Enables user-defined extended attributes (required for Samba to store Windows metadata)
- `acl` — Enables POSIX Access Control Lists (required for Samba to enforce AD permissions)
- `barrier=1` — Enables write barriers for data integrity protection

<br>

<img width="931" height="283" alt="image" src="https://github.com/user-attachments/assets/621a52c0-f4cd-41e6-b168-72ab2261e54c" />

<br>

**Apply the fstab configuration immediately:**

```bash
sudo mount -a
df -h /srv/samba/Data
```

<br>

`mount -a` mounts all entries in `/etc/fstab`. The `df` command then confirms the new disk is mounted and shows the available space:

<br>

<img width="490" height="62" alt="image" src="https://github.com/user-attachments/assets/4b3b45c1-ff0c-4920-88fd-f18b69079795" />

<br>

### Storage Verification Checks

We run a series of checks to confirm the storage is set up correctly before using it for shares.

**Verify block devices and partition structure:**

```bash
lsblk
```

<br>

`sdb1` now shows as a mounted partition under `/srv/samba/Data`, confirming the mount is active:

<br>

<img width="620" height="298" alt="image" src="https://github.com/user-attachments/assets/17e3fa2e-3e02-481e-8e41-f37aa518d58e" />

<br>

**Check available disk space at the mount point:**

```bash
df -h /srv/samba/Data
```

<br>

The output shows the total size, used space, available space, and usage percentage:

<br>

<img width="510" height="66" alt="image" src="https://github.com/user-attachments/assets/d3e1f9a9-211c-4e5a-a253-958bdb5def92" />

<br>

**Confirm the fstab entry is correct:**

```bash
cat /etc/fstab | grep Data
```

<br>

The fstab entry with the UUID and correct mount options is displayed, confirming the disk will remount automatically after a reboot:

<br>

<img width="851" height="40" alt="image" src="https://github.com/user-attachments/assets/3c3c326b-c035-432a-8f5b-ea7394c57ef4" />

<br>

**Validate the ACL and extended attribute flags are active:**

```bash
findmnt -n -o OPTIONS /srv/samba/Data
```

<br>

The output lists all active mount options. We confirm `acl` and `user_xattr` are present — without these, Samba will not be able to enforce AD permissions on files:

<br>

<img width="517" height="47" alt="image" src="https://github.com/user-attachments/assets/ef509aaa-9e02-4ea4-93d9-5f1e422a4a53" />

<br>

---

## 2. 📁 Directory Structure and Network Shares

### Step 2.1 — Create the Department Directory Hierarchy

We create a folder for each department that will have its own Samba share with different access rights:

```bash
sudo mkdir -p /srv/samba/Data/FinanceDocs
sudo mkdir -p /srv/samba/Data/HRDocs
sudo mkdir -p /srv/samba/Data/Public
```

<br>

All three directories are created on the newly mounted data disk:

<br>

<img width="555" height="86" alt="image" src="https://github.com/user-attachments/assets/aadd8f37-c0b1-4d56-96fa-7dfe4c1051aa" />

<br>

**Verify the directory structure:**

```bash
ls -l /srv/samba/Data
```

<br>

The listing shows all three directories with their current ownership (root:root) and permissions. We will configure more specific ownership and ACLs in the next sprint:

<br>

<img width="482" height="125" alt="image" src="https://github.com/user-attachments/assets/9379cb03-c380-4bd4-988b-35ac13e3f3d1" />

<br>

### Step 2.2 — Configure Samba Network Shares

**Open the Samba configuration file:**

```bash
sudo nano /etc/samba/smb.conf
```

<br>

Add the following share definitions at the bottom of the file. Each share section specifies the path, which groups can access it, and what level of access they have:

```ini
[FinanceDocs]
    comment = Finance Department Documents
    path = /srv/samba/FinanceDocs
    valid users = @Students, @"Domain Admins"
    read only = no
    browseable = yes
    create mask = 0660
    directory mask = 0770

[HRDocs]
    comment = HR Department Documents
    path = /srv/samba/HRDocs
    valid users = @IT_Admins, @"Domain Admins"
    read only = no
    browseable = yes
    create mask = 0660
    directory mask = 0770

[Public]
    comment = Public Shared Documents (Read-Only)
    path = /srv/samba/Public
    valid users = @"Domain Users"
    read only = yes
    browseable = yes
    write list = @"Domain Admins"
```

The `create mask` and `directory mask` settings define the maximum permissions new files and folders can have. For example, `0660` means files are readable and writable by owner and group, but not accessible to others. The `write list` override in `[Public]` allows Domain Admins to write even though the share is set to read-only for everyone else.

<br>

<img width="562" height="862" alt="image" src="https://github.com/user-attachments/assets/ba062069-d25b-43fa-963d-30b9f811896f" />

<br>

**Validate the configuration syntax before restarting:**

```bash
testparm
```

<br>

`testparm` parses the configuration file and reports any errors or warnings. A clean run with the message `Loaded services file OK` confirms the file is valid:

<br>

<img width="467" height="132" alt="image" src="https://github.com/user-attachments/assets/13e1c51c-e60d-4585-bc3d-dcd2fb24af43" />

<br>

**Restart Samba to apply the changes:**

```bash
sudo systemctl restart samba-ad-dc
```

<br>

<img width="481" height="22" alt="image" src="https://github.com/user-attachments/assets/9356308d-3935-4b3f-8143-f7e59e4e8d83" />

<br>

### Step 2.3 — Verify the Network Shares

**List all available shares on the server:**

```bash
smbclient -L localhost -N
```

<br>

The output lists all shares, including the new `FinanceDocs`, `HRDocs`, and `Public` shares alongside the built-in `netlogon`, `sysvol`, and `IPC$` shares:

<br>

<img width="627" height="247" alt="image" src="https://github.com/user-attachments/assets/3e607ae5-571a-4f67-ac73-04cec65797ed" />

<br>

**Check directory ownership and permission bits:**

```bash
ls -ld /srv/samba/Data/*
```

<br>

The listing shows the current owner, group, and permission string for each share directory. These will be refined further when ACLs are applied:

<br>

<img width="632" height="103" alt="image" src="https://github.com/user-attachments/assets/1d984dde-e425-4f85-9b01-3aad1b874891" />

<br>

---

## 3. 🔐 Advanced Permissions and ACLs

### Step 3.1 — Install ACL Support

POSIX ACLs extend the standard Unix permission model (owner/group/others) to allow per-user and per-group permissions on individual files and directories. This is essential for Samba to correctly enforce the granular Windows-style permissions that AD administrators expect:

```bash
sudo apt update && sudo apt install acl -y
```

With ACL support installed, administrators can use `setfacl` to grant or revoke permissions for specific users or groups, and `getfacl` to read the current permission set on any file or directory. This goes far beyond what standard `chmod` can achieve.

<br>

---

## 4. ⚙️ Task and Process Management

> Understanding how to monitor and control running processes is a core Linux administration skill. On a Domain Controller, it is especially important to know which Samba processes are running, how to identify resource-hungry processes, and how to gracefully manage service restarts and process signals.

<br>

### 4.1 — The `top` Utility

`top` is the most widely used tool for real-time process monitoring. It updates automatically every few seconds and shows all running processes ranked by CPU usage by default.

Key columns:
- **PID** — The unique process identifier assigned by the kernel
- **USER** — Which user owns this process
- **%CPU** — Percentage of one CPU core being consumed
- **%MEM** — Percentage of physical RAM in use
- **TIME+** — Total CPU time consumed since the process started
- **COMMAND** — The name of the executable

```bash
top
```

<br>

<img width="185" height="20" alt="imagen" src="https://github.com/user-attachments/assets/14bed00a-c0bd-45e3-80cc-4b15ac4fb3b3" />

<br>

The `top` output shows a live view of all processes. Look for `samba`, `smbd`, and `winbindd` processes to confirm the AD services are running:

<br>

<img width="801" height="371" alt="imagen" src="https://github.com/user-attachments/assets/197661f5-aee2-4a27-9fe2-88422b196d36" />

<br>

### 4.2 — The `htop` Utility

`htop` is a modern, color-coded replacement for `top`. It provides a much more intuitive interface with mouse support, horizontal scrolling to see full command names, and built-in process kill/signal functionality. It is the preferred tool for interactive system monitoring.

```bash
htop
```

<br>

<img width="634" height="134" alt="imagen" src="https://github.com/user-attachments/assets/0b79537a-2c61-432f-8539-d9dd3d074ab0" />

<br>

The `htop` interface showing CPU meters, memory usage, and the process list with color coding:

<br>

<img width="798" height="364" alt="imagen" src="https://github.com/user-attachments/assets/8ffee18a-3ef0-4983-9f06-b322e4dc6b58" />

<br>

**Key interactive commands in htop:**

| Key | Action |
|-----|--------|
| `F2` | Open Setup to customize columns, colors, and meters |
| `F3` | Search for a process by name |
| `F5` | Toggle between tree view and flat list |
| `F6` | Sort processes by any column |
| `F9` | Send a signal to the selected process (kill/stop/resume) |
| `F10` | Quit htop |

<br>

### 4.3 — The `ps` Command

While `top` and `htop` provide live interactive views, `ps` (Process Status) takes a one-time snapshot of the process table. This makes it ideal for use in shell scripts, log files, and piping to `grep`. The most common invocation is `ps aux`:

- **`a`** — Show processes from all users, not just the current user
- **`u`** — Display in user-friendly format, showing username, CPU, and memory
- **`x`** — Include processes not attached to a terminal (background daemons)

```bash
# View all running processes
ps aux

# Filter specifically for Samba-related processes
ps aux | grep samba
```

<br>

The filtered output shows all Samba processes including their PIDs, which can be used to send signals directly:

<br>

<img width="795" height="245" alt="imagen" src="https://github.com/user-attachments/assets/d9dad8a9-70d3-4342-ad55-df73a9328e02" />

<br>

### 4.4 — Service Management with systemctl

`systemctl` is the central tool for managing services in modern Linux (systemd-based) systems. It handles starting, stopping, restarting, enabling, and querying the status of services.

**Check the detailed status of the Samba AD DC service:**

```bash
sudo systemctl status samba-ad-dc
```

<br>

The status output includes: service state (active/inactive), PID, uptime, recent journal entries, and sub-process details:

<br>

<img width="802" height="399" alt="imagen" src="https://github.com/user-attachments/assets/adb7f9ee-c7d6-47f2-9dff-fb81baa2f535" />

<br>

**Restart a service that is hung or needs a configuration reload:**

```bash
sudo systemctl restart samba-ad-dc
```

<br>

A clean restart with no errors — the command returns silently:

<br>

<img width="426" height="20" alt="imagen" src="https://github.com/user-attachments/assets/d2921b5e-1fba-40a5-b692-1cf94f4d64a5" />

<br>

**Enable the service so it starts automatically on every boot:**

```bash
sudo systemctl enable samba-ad-dc
```

<br>

The enable command creates symbolic links in the systemd unit directories, ensuring the service starts automatically:

<br>

<img width="800" height="69" alt="imagen" src="https://github.com/user-attachments/assets/fbbb5ded-cde1-4a74-8aad-7525276ac856" />

<br>

**List all currently active services to see the full picture of what is running:**

```bash
systemctl list-units --type=service --state=running
```

<br>

The output shows every active service on the system. This is useful for identifying unexpected services that may be consuming resources or causing conflicts:

<br>

<img width="798" height="469" alt="imagen" src="https://github.com/user-attachments/assets/7302992d-1513-424d-8528-9a03744609bb" />

<br>

### 4.5 — Sending Signals to Processes

Linux uses **signals** as the primary mechanism for inter-process communication. When a process becomes unresponsive, we send it a signal to tell it what to do. The most important signals for system administration are:

```bash
# SIGTERM (signal 15) — Graceful shutdown, allows the process to save state and clean up
kill <PID>

# SIGKILL (signal 9) — Immediate, unconditional termination. No cleanup possible.
kill -9 <PID>

# SIGSTOP (signal 19) — Pause a process without terminating it
kill -19 <PID>

# SIGCONT (signal 18) — Resume a previously stopped process
kill -18 <PID>

# Kill all instances of a process by name
sudo killall smbd
```

> Always try `SIGTERM` first. `SIGKILL` is a last resort — it cannot be caught or ignored by the process, so the process has no chance to release resources or flush data to disk.

<br>

### 4.6 — Practical Exercise: Pausing and Resuming a Process

This exercise demonstrates process control signals using the `sl` program (a playful terminal animation of a steam locomotive). It is a harmless way to practice the `kill` command in a lab setting.

**Step 1: Run the animation:**

<br>

<img width="797" height="433" alt="imagen" src="https://github.com/user-attachments/assets/81ee956f-97c3-4203-880c-674755a4aa2b" />

<br>

**Step 2: Find its PID using `pgrep`:**

```bash
pgrep -sl
```

<br>

`pgrep` searches the process table by name and returns the matching PID:

<br>

<img width="268" height="126" alt="imagen" src="https://github.com/user-attachments/assets/acc2b3c5-6bd7-4472-8a9d-ede1c98a371a" />

<br>

**Step 3: Send SIGSTOP to freeze the animation:**

```bash
kill -19 <PID>
```

Signal 19 is `SIGSTOP`. The locomotive animation freezes mid-screen, but the process is still alive in memory:

<br>

<img width="288" height="26" alt="imagen" src="https://github.com/user-attachments/assets/612c9297-1d6d-4246-817a-aa366baa9cef" />

<br>

The terminal shows the frozen animation state — the process is paused but has not been terminated:

<br>

<img width="657" height="291" alt="imagen" src="https://github.com/user-attachments/assets/1fb29639-cc40-4409-aaa2-a540d5a5485c" />

<br>

**Step 4: Send SIGCONT to resume the process:**

```bash
kill -18 <PID>
```

Signal 18 is `SIGCONT` (continue). The animation immediately resumes from exactly where it stopped:

<br>

<img width="1563" height="473" alt="imagen" src="https://github.com/user-attachments/assets/80e19292-b10e-41c4-8ebf-1dba21e7f671" />

<br>

---

<br>

# 🤝 SPRINT 4 — Domain Trust

> **Goal:** Establish a **bidirectional forest trust** between two completely independent Samba Active Directory domains — `lab12.lan` (our original DC from Sprint 1) and a new domain `lab120.lan` (a second DC set up specifically for this sprint). A forest trust allows users from either domain to authenticate and access resources in the other domain, enabling cross-organizational collaboration without merging the two domains into one.

<br>

---

## Step 1 — Install and Configure the Second Domain Controller (LS120)

We provision a brand-new Ubuntu Server as a separate, independent AD DC for the `lab120.lan` domain. This server goes through the same installation and configuration steps as Sprint 1, but with different domain and realm names. The two DCs must be able to reach each other over the network.

<br>

The new virtual machine is created in the hypervisor:

<br>

<img width="801" height="512" alt="image" src="https://github.com/user-attachments/assets/e485bfc4-b333-436a-a402-f5399eea63a8" />

<br>

The operating system installation begins on the new VM:

<br>

<img width="801" height="512" alt="image" src="https://github.com/user-attachments/assets/52e791f3-6942-4936-b1fb-8b16f111e492" />

<br>

The installation completes and the system boots to a command prompt:

<br>

<img width="798" height="320" alt="image" src="https://github.com/user-attachments/assets/440d3c14-49c0-4242-8bf5-fe312c9f06e1" />

<br>

Basic network connectivity is verified after the OS installation — the new server can reach the internet:

<br>

<img width="799" height="595" alt="imagen" src="https://github.com/user-attachments/assets/d5b3436b-e980-4fff-9db1-c5cb8feb5bc4" />

<br>

---

## Step 2 — Network Configuration for LS120

The second DC needs a static IP on its internal interface and must be able to reach `ls12.lab12.lan` over the network. We follow the same Netplan process as in Sprint 1.

**Edit the Netplan configuration:**

```bash
nano /etc/netplan/00-installer-config.yaml
```

<br>

The configuration assigns a static IP (`192.168.30.42`) to the internal interface. This IP is in the same subnet as the first DC so they can communicate:

<br>
```
network:
  ethernets:
    enp0s3:
      addresses:
        - 192.168.1.80/24
      nameservers:
        addresses:
          - 8.8.8.8        # Added Google DNS for public resolution
          - 1.1.1.1        # Added Cloudflare DNS as backup
        search: []
      routes:
        - to: default
          via: 192.168.1.1
    enp0s8:
      addresses: [192.168.30.42/24]
      dhcp4: false
  version: 2

```





<img width="616" height="168" alt="imagen" src="https://github.com/user-attachments/assets/f3debf6c-2daf-4552-848e-e246bb7df90d" />

<br>

**Apply the network changes:**

```bash
sudo netplan apply
```

<br>

The network is reconfigured with the new static IP:

<br>

<img width="799" height="231" alt="imagen" src="https://github.com/user-attachments/assets/d2d54bb5-0b57-452e-ae68-22265328c44d" />

<br>

We verify the IP assignment was applied correctly by checking the interface details:

<br>

<img width="800" height="339" alt="imagen" src="https://github.com/user-attachments/assets/8cb90cd3-b4d4-4c00-97ab-d1c1bb2ca4a7" />

<br>

**Remove the systemd-managed symlink and create a static resolv.conf:**

```bash
sudo unlink /etc/resolv.conf
```

<br>

<img width="415" height="18" alt="imagen" src="https://github.com/user-attachments/assets/6e52f4d5-7961-4d9a-847f-49aca9786fe1" />

<br>

The static resolv.conf is created with the new server's own IP as the nameserver (since it will be its own DNS server after provisioning):

<br>

<img width="796" height="598" alt="imagen" src="https://github.com/user-attachments/assets/c0077bf7-e2dc-4c54-9d8f-e212ce34dd27" />

<br>

The immutable flag is applied to prevent the file from being overwritten:

<br>

<img width="437" height="21" alt="imagen" src="https://github.com/user-attachments/assets/1622c6e4-9825-4d69-a832-b88d187f2251" />

<br>

---

## Step 3 — Install Samba on LS120

We run the same package installation command as in Sprint 1. The Kerberos configuration prompts will now ask for `LAB120.LAN` as the realm:

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 \
  krb5-config krb5-user dnsutils chrony net-tools
```

<br>

The Kerberos realm prompt — enter `LAB120.LAN` in uppercase to match the new domain:

<br>

<img width="907" height="285" alt="image" src="https://github.com/user-attachments/assets/36ce743a-b587-41fc-acf4-e92bc4e56a56" />

<br>

The KDC hostname prompt — enter `ls120.lab120.lan`, the FQDN of the second DC:

<br>

<img width="907" height="218" alt="image" src="https://github.com/user-attachments/assets/91e70db1-92a4-4ed2-9f7c-d3cd18b7b24f" />

<br>

The administrative server prompt — enter the same FQDN:

<br>

<img width="913" height="222" alt="image" src="https://github.com/user-attachments/assets/51399d25-3dd9-420e-9884-83beac40af2c" />

<br>

---

## Step 4 — Disable Unneeded Services and Enable the AD DC

On a Domain Controller, only `samba-ad-dc` should run. The member-server daemons (`smbd`, `nmbd`, `winbind`) must be stopped and disabled to prevent port conflicts:

```bash
sudo systemctl disable --now smbd nmbd winbind
```

<br>

All three services are stopped and disabled. The output confirms the symlinks were removed from the systemd startup directories:

<br>

<img width="923" height="138" alt="image" src="https://github.com/user-attachments/assets/2d76573f-b3ca-4d3d-804a-e1c93f3f336e" />

<br>

**Unmask and enable `samba-ad-dc`:**

The service is masked by default (to prevent accidental start before provisioning). We unmask it and then enable it for automatic start on boot:

```bash
sudo systemctl unmask samba-ad-dc
```

<br>

<img width="516" height="45" alt="image" src="https://github.com/user-attachments/assets/e8d4dfe9-3051-4d91-b1b5-12adfb666e77" />

<br>

```bash
sudo systemctl enable samba-ad-dc
```

<br>

<img width="937" height="125" alt="image" src="https://github.com/user-attachments/assets/6fa38068-74eb-4a50-878a-c4d01a6b407b" />

<br>

---

## Step 5 — Provision Samba AD on LS120

**Back up the default smb.conf** (the provisioning tool requires this file to not exist or be empty):

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
```

<br>

<img width="678" height="21" alt="image" src="https://github.com/user-attachments/assets/490745f2-8e2f-4c00-8ff3-ba3fb8676700" />

<br>

**Run the interactive provisioning wizard:**

```bash
sudo samba-tool domain provision
```

Enter the following values when prompted. These define the new, separate domain:

```
Realm:       LAB120.LAN
Domain:      LAB120
Server Role: dc
DNS backend: SAMBA_INTERNAL
DNS forwarder IP address: (leave blank or set to 1.1.1.1 for internet)
```

<br>

The provisioning tool creates the AD database, generates the Kerberos keytab, sets up the internal DNS zones, and writes the final `smb.conf` configuration file:

<br>

<img width="941" height="257" alt="image" src="https://github.com/user-attachments/assets/ab455bb6-4c07-404a-be91-d7d74d35a1f3" />

<br>

**Replace the system Kerberos configuration with Samba's generated one.** Samba creates a tailored `krb5.conf` during provisioning that is pre-configured with the correct realm settings:

```bash
sudo mv /etc/krb5.conf /etc/krb5.conf.orig
```

<br>

<img width="597" height="20" alt="image" src="https://github.com/user-attachments/assets/7b9042f4-a44a-4e0b-94d9-67e4d5a1774e" />

<br>

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

<br>

<img width="702" height="23" alt="image" src="https://github.com/user-attachments/assets/105b68aa-5f7b-4c10-a53d-fce162c01af3" />

<br>

**Start the Samba AD DC service:**

```bash
sudo systemctl start samba-ad-dc
```

<br>

<img width="488" height="28" alt="image" src="https://github.com/user-attachments/assets/a70b9a6f-3402-4af8-9ec3-8d38c84b5c31" />

<br>

**Verify the service is running and healthy:**

```bash
sudo systemctl status samba-ad-dc
```

<br>

The status output shows `active (running)` for the `lab120.lan` domain controller:

<br>

<img width="942" height="577" alt="image" src="https://github.com/user-attachments/assets/de81b232-1c96-4c49-938c-f9e7807fec44" />

<br>

---

## Step 6 — Time Synchronization for LS120

The second DC also needs Chrony configured correctly. Kerberos will reject any authentication requests between the two domains if their clocks differ by more than 5 minutes.

**Set permissions on the NTP signing socket:**

```bash
sudo chown root:_chrony /var/lib/samba/ntp_signd/
```

<br>

<img width="657" height="30" alt="image" src="https://github.com/user-attachments/assets/6f13422d-1863-4b56-a9d8-8229e61592f5" />

<br>

```bash
sudo chmod 750 /var/lib/samba/ntp_signd/
```

<br>

<img width="570" height="25" alt="image" src="https://github.com/user-attachments/assets/8c92ed38-2fec-4fbf-b56f-fc0617e40785" />

<br>

**Edit the Chrony configuration:**

```bash
sudo nano /etc/chrony/chrony.conf
```

<br>

<img width="513" height="22" alt="image" src="https://github.com/user-attachments/assets/40ab6841-c6cd-4f9a-a5f4-91ef064a2a63" />

<br>

Add the NTP server configuration lines. Note that `bindcmdaddress` uses `192.168.30.42` — the IP of the second DC:

```
bindcmdaddress 192.168.30.42
allow 192.168.30.0/24
ntpsigndsocket /var/lib/samba/ntp_signd
```

<br>

The full Chrony configuration file for LS120 after our additions:

<br>

<img width="941" height="933" alt="image" src="https://github.com/user-attachments/assets/e996678b-853e-47f6-9871-6bcdd8d6384f" />

<br>

**Restart Chrony and confirm it is running:**

```bash
sudo systemctl restart chronyd
```

<br>

<img width="472" height="23" alt="image" src="https://github.com/user-attachments/assets/fa7f0092-fb8a-428a-9453-ff4a6a8ce5e2" />

<br>

```bash
sudo systemctl status chronyd
```

<br>

Chrony is active, synchronized to upstream NTP sources, and listening for requests from domain clients:

<br>

<img width="937" height="512" alt="image" src="https://github.com/user-attachments/assets/079a9d29-1568-448d-943b-fc6481f47d0b" />

<br>

---

## Step 7 — Verify Samba AD on LS120

We run the same full set of verification checks on the new DC to confirm everything is working before attempting to create the trust relationship.

**Verify DNS A records for the new domain:**

```bash
host -t A lab120.lan
```

<br>

The domain name resolves to the second DC's IP:

<br>

<img width="393" height="71" alt="image" src="https://github.com/user-attachments/assets/0dfaf3ae-23aa-4d4c-9c98-1d069682c802" />

<br>

```bash
host -t A ls120.lab120.lan
```

<br>

The second DC's FQDN resolves correctly:

<br>

<img width="452" height="67" alt="image" src="https://github.com/user-attachments/assets/6ca673af-87bf-4442-8f39-328e771a457b" />

<br>

**Verify Kerberos and LDAP SRV records:**

```bash
host -t SRV _kerberos._udp.lab120.lan
```

<br>

The Kerberos service record points to the second DC on port 88:

<br>

<img width="613" height="42" alt="image" src="https://github.com/user-attachments/assets/78cb578b-0773-4205-b170-34928b50ba79" />

<br>

```bash
host -t SRV _ldap._tcp.lab120.lan
```

<br>

The LDAP service record points to the second DC on port 389:

<br>

<img width="582" height="51" alt="image" src="https://github.com/user-attachments/assets/fce9dde7-6dff-4d0b-ad72-8cef956b7fea" />

<br>

**Verify SMB shares are available:**

```bash
smbclient -L clockwork.local -N
```

<br>

The default shares (`netlogon`, `sysvol`, `IPC$`) are present, confirming AD DC mode is active:

<br>

<img width="617" height="191" alt="image" src="https://github.com/user-attachments/assets/5cc241c6-10da-4984-a95a-362c73358c51" />

<br>

**Test Kerberos authentication on the new domain:**

```bash
kinit administrator@CLOCKWORK.LOCAL
```

<br>

<img width="617" height="191" alt="image" src="https://github.com/user-attachments/assets/3219284b-18bc-41f7-9c07-b8f588f77723" />

<br>

```bash
klist
```

<br>

The TGT for the `lab120.lan` administrator is displayed with valid timestamps:

<br>

<img width="606" height="142" alt="image" src="https://github.com/user-attachments/assets/dbeaec9c-ad91-46cb-8deb-8934279c3c5a" />

<br>

**Test SMB login:**

```bash
sudo smbclient //localhost/netlogon -U 'administrator'
```

<br>

<img width="737" height="71" alt="image" src="https://github.com/user-attachments/assets/7ab6af6d-e4e2-4970-8942-4531356fc927" />

<br>

**Set the administrator password for the new domain:**

```bash
sudo samba-tool user setpassword administrator
```

<br>

<img width="640" height="85" alt="image" src="https://github.com/user-attachments/assets/1f37899d-e578-47e9-881c-0a5a9c577d2b" />

<br>

**Validate the Samba configuration:**

```bash
testparm
```

<br>

The configuration passes validation with no errors:

<br>

<img width="507" height="787" alt="image" src="https://github.com/user-attachments/assets/f9cf7707-31fb-4bf7-8c74-6ea10b76ca7d" />

<br>

**Check the domain functional level of the new domain:**

```bash
sudo samba-tool domain level show
```

<br>

Both forest and domain functional levels report `Windows 2008 R2` or higher, confirming the new DC is fully provisioned:

<br>

<img width="556" height="128" alt="image" src="https://github.com/user-attachments/assets/e6e0b557-bfd9-4bda-a92b-14b4fa6df8b5" />

<br>

---

## Step 8 — Establish the Bidirectional Forest Trust

With both DCs verified and operational, we create the trust. A **forest trust** is the highest level of trust between two AD domains — it grants users in either domain the ability to be authenticated by the other domain's DC and access resources across the boundary.

**Run this command from the LS12 Domain Controller (the `lab12.lan` side):**

```bash
sudo samba-tool domain trust create lab120.lan \
  --type=forest \
  --direction=both \
  -U administrator@lab120.lan
```

You will be prompted for the `lab120.lan` administrator password. The command then connects to the second DC, negotiates the trust relationship, and registers the trust objects in both AD databases.

Flags explained:
- **`--type=forest`** — Creates a forest-level trust (highest level, allows full transitivity)
- **`--direction=both`** — Bidirectional trust: `lab12.lan` trusts `lab120.lan` AND vice versa
- **`-U administrator@lab120.lan`** — Authenticates as the remote domain's admin to authorize the trust from both sides

<br>

The trust creation output shows the trust being established step by step across both domains:

<br>

<img width="1242" height="490" alt="image" src="https://github.com/user-attachments/assets/0f5c35d3-2fee-49f3-8247-526ef7358161" />

<br>

**List all configured trusts to confirm the new trust was recorded:**

```bash
sudo samba-tool domain trust list
```

<br>

The `lab120.lan` domain appears in the trust list with direction `Both` and type `Forest`:

<br>

<img width="606" height="48" alt="image" src="https://github.com/user-attachments/assets/1bda3171-d848-4119-9ab9-48b153324984" />

<br>

**Inspect the detailed trust attributes:**

```bash
sudo samba-tool domain trust show lab120.lan
```

<br>

The detailed output shows the trust type, direction, flags, and the partner domain's SID. This is the full picture of the trust object stored in the AD database:

<br>

<img width="935" height="323" alt="image" src="https://github.com/user-attachments/assets/8843d7d7-185d-42f3-b6db-477c3c94f0b3" />

<br>

**Validate the trust is functionally operational:**

```bash
sudo samba-tool domain trust validate lab06.lan
```

<br>

A successful validation performs a full end-to-end check — it authenticates across the trust boundary and confirms the trust is not just configured, but actively working:

<br>

<img width="942" height="142" alt="image" src="https://github.com/user-attachments/assets/98bab33f-f4fc-4316-970b-09e3d21b7737" />

<br>

---

## Step 9 — Create Test Users on LAB120

To fully test cross-domain authentication, we create two standard test user accounts on the `lab120.lan` domain. These accounts will be used to verify that a user from one domain can authenticate on a machine joined to the other domain:

```bash
sudo samba-tool user create testuser admin_21 --given-name=Test --surname=User
```

<br>

The first test user `testuser` is created successfully in the `lab120.lan` domain:

<br>

<img width="901" height="27" alt="image" src="https://github.com/user-attachments/assets/9cded784-0c95-415f-b00e-1e1954ae4d52" />

<br>

```bash
sudo samba-tool user create testuser2 admin_21 --given-name=Test --surname=User
```

<br>

The second test user `testuser2` is also created:

<br>

<img width="925" height="32" alt="image" src="https://github.com/user-attachments/assets/fb3bb7dc-f757-454b-8710-b628281453b5" />

<br>

**Verify both users appear in the domain's user list:**

```bash
sudo samba-tool user list
```

<br>

The output shows all users in the `lab120.lan` domain, including the two newly created test accounts alongside the default `Administrator` account:

<br>

<img width="427" height="123" alt="image" src="https://github.com/user-attachments/assets/dc929015-e441-4b7e-98d0-21296d75ced8" />

<br>

---

## Step 10 — Cross-Domain Authentication Verification

This final step is the definitive proof that the forest trust is working end-to-end. We log in on a machine that is joined to `lab12.lan` and attempt to authenticate using credentials from the `lab120.lan` domain — something that would have been impossible before the trust was established.

**On the LS12 Domain Controller, request a Kerberos ticket for a LAB120 user:**

```bash
kinit testuser@LAB120.LAN
```

When you press Enter, Kerberos contacts the `lab12.lan` KDC, which recognizes `LAB120.LAN` as a trusted domain and redirects the authentication request to the `lab120.lan` KDC. The remote KDC validates the credentials and issues a ticket that is honored by `lab12.lan`.

<br>

The command prompts for the password and, on success, returns silently — the ticket is now cached:

<br>

<img width="608" height="57" alt="image" src="https://github.com/user-attachments/assets/4a316fce-5e16-4827-b079-e2ee65ca71d1" />

<br>

**Display the cached Kerberos tickets to confirm the cross-domain ticket was issued:**

```bash
klist
```

<br>

The ticket cache shows a TGT for `testuser@LAB120.LAN` — this is a foreign-realm ticket stored on a machine in `lab12.lan`. This is definitive proof that cross-domain Kerberos authentication is working through the forest trust:

<br>

<img width="555" height="110" alt="image" src="https://github.com/user-attachments/assets/9e8ffd67-0d5c-4a92-8b48-959fac61efbe" />

<br>

**Clean up — destroy all cached tickets:**

```bash
kdestroy
```

`kdestroy` removes all Kerberos tickets from the credential cache. This is good security practice after testing — you do not want sensitive tickets left in memory when testing is complete.

<br>

<img width="223" height="31" alt="image" src="https://github.com/user-attachments/assets/97fecc7c-f200-47d0-a8b3-1c815c27da76" />

<br>

The forest trust is fully operational. Users from `lab120.lan` can now authenticate on machines joined to `lab12.lan`, and vice versa — completing the final sprint of this project.

<br>

---

<br>

*📄 Linux-Sprint — Technical Documentation for Samba Active Directory on Ubuntu Server*
