# 🐧 Linux-Sprint

> **Technical documentation** of the full configuration process for a Linux environment with Samba Active Directory, client integration, storage management, and domain administration. This project covers four sprints, each building on the previous one to create a complete, production-ready AD infrastructure.

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

> **Goal:** Configure a fresh Ubuntu Server installation as a fully functional Samba Active Directory Domain Controller (AD DC). This involves setting the hostname, configuring network interfaces, installing the required packages, provisioning Samba AD, and verifying all services are operational.

<br>

---

## ⚙️ Step 1 — Hostname Configuration

The first step is to assign a proper, permanent hostname to the server. This name will be part of the Fully Qualified Domain Name (FQDN) and is critical for Kerberos and DNS to work correctly.

```bash
sudo hostnamectl set-hostname ls12
```

> After running this command, the hostname change takes effect immediately without requiring a reboot.

<br>

<img width="474" height="71" alt="imagen" src="https://github.com/user-attachments/assets/6932197e-3e60-4bac-ad15-ba54634bc41a" />

<br>

---

## 📄 Step 2 — Edit the /etc/hosts File

The `/etc/hosts` file must be updated to map the server's FQDN and short hostname to its static IP address. This is essential for local DNS resolution before any external DNS service is available.

```bash
sudo nano /etc/hosts
```

> Add a line in the format: `<IP_ADDRESS>  <FQDN>  <SHORT_HOSTNAME>`
> Example: `192.168.30.40  ls12.lab12.lan  ls12`

<br>

<img width="801" height="179" alt="imagen" src="https://github.com/user-attachments/assets/23de9c27-387b-4207-8132-0e11b7a36283" />

<br>

---

## 🌐 Step 3 — Network Configuration (Netplan)

The server requires a **static IP address** on both network interfaces. We configure this using Netplan, Ubuntu's default network configuration tool. One interface (`enp0s3`) connects to the external network, and the other (`enp0s8`) serves the internal AD network.

```bash
nano /etc/netplan/00-installer-config.yaml
```

<br>

Add or modify the file with the following configuration:

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

> **Note:** `enp0s8` uses `127.0.0.1` as its DNS, because once Samba AD is running, the DC itself will act as the DNS server for the internal domain.

<br>

<img width="978" height="396" alt="imagen" src="https://github.com/user-attachments/assets/a107fee4-0764-4043-90f0-0b62c1d6cabd" />

<br>

<img width="795" height="102" alt="imagen" src="https://github.com/user-attachments/assets/495f7dc0-e14f-49c1-9405-8be5337216b9" />

<br>

Apply the network changes to make them active:

```bash
sudo netplan apply
```

> If there are no errors, the command will return silently. You can verify the IP addresses with `ip a`.

<br>

<img width="631" height="131" alt="imagen" src="https://github.com/user-attachments/assets/e77da347-3d6d-4ac4-b232-ef60bb549406" />

<br>

<img width="798" height="341" alt="imagen" src="https://github.com/user-attachments/assets/09f7ee55-1976-4699-abee-e882b24fada9" />

<br>

<img width="623" height="53" alt="imagen" src="https://github.com/user-attachments/assets/683f6c83-ddb9-4143-b90e-cfaa1ea63084" />

<br>

---

## 🔗 Step 4 — Fix DNS Resolution (/etc/resolv.conf)

By default, Ubuntu manages `/etc/resolv.conf` via a systemd symlink, which can override our static DNS settings. We need to break the symlink, manually create the file, and then make it immutable.

<br>

**Remove the existing symbolic link:**

```bash
sudo unlink /etc/resolv.conf
```

<img width="382" height="20" alt="imagen" src="https://github.com/user-attachments/assets/596162ac-cb9f-49ff-94d7-e7bc392f9728" />

<br>

**Recreate the file manually with the correct DNS entries:**

```bash
sudo nano /etc/resolv.conf
```

> Add the domain's nameserver and search domain. For example:
> ```
> nameserver 192.168.30.40
> search lab12.lan
> ```

<br>

<img width="802" height="110" alt="imagen" src="https://github.com/user-attachments/assets/736ebab6-1fa6-430b-85e5-c3229df50664" />

<br>

**Make the file immutable to prevent any service from overwriting it:**

```bash
sudo chattr +i /etc/resolv.conf
```

> The `+i` flag sets the immutable attribute. To edit it again in the future, you must first run `sudo chattr -i /etc/resolv.conf`.

<br>

<img width="407" height="28" alt="imagen" src="https://github.com/user-attachments/assets/772b9b80-3d01-4288-8f97-1aaa1ab82880" />

<br>

<img width="674" height="234" alt="imagen" src="https://github.com/user-attachments/assets/d2ea2c58-ae52-4299-82a5-d5eac2c8902e" />

<br>

---

## 📦 Step 5 — Install Samba and All Required Packages

This command installs Samba and all the components needed for a fully functional Active Directory Domain Controller, including Kerberos (authentication), Winbind (user/group resolution), Chrony (time sync), and DNS utilities.

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 \
  krb5-config krb5-user dnsutils chrony net-tools
```

> During the installation, you may be prompted to enter the Kerberos realm. Enter it in **ALL CAPS** (e.g., `LAB12.LAN`).

<br>

<img width="797" height="40" alt="imagen" src="https://github.com/user-attachments/assets/74169420-b712-4ac3-87f7-0ebb8f2ca49d" />

<br>

<img width="766" height="222" alt="imagen" src="https://github.com/user-attachments/assets/814abeca-9769-4d4e-88b5-cb9654f418b9" />

<br>

<img width="719" height="177" alt="imagen" src="https://github.com/user-attachments/assets/10b47588-1973-4418-bb87-273d9ca4d8d5" />

<br>

<img width="721" height="180" alt="imagen" src="https://github.com/user-attachments/assets/f9cc1d19-d2c8-4180-92ca-478be067531e" />

<br>

<img width="801" height="183" alt="imagen" src="https://github.com/user-attachments/assets/a93d94f4-6e17-4ccc-8af1-0ba59aab0c09" />

<br>

<img width="802" height="126" alt="imagen" src="https://github.com/user-attachments/assets/303dc108-ff3a-42bf-a8e3-03280eb874c8" />

<br>

<img width="796" height="68" alt="imagen" src="https://github.com/user-attachments/assets/62c396aa-772e-42d2-b5ec-b89512604af5" />

<br>

<img width="578" height="39" alt="imagen" src="https://github.com/user-attachments/assets/faaa0ea9-a377-43bd-8c2c-7582223b0b7f" />

<br>

<img width="635" height="116" alt="imagen" src="https://github.com/user-attachments/assets/fc49ebbd-a635-4b78-bd18-c135e22feee5" />

<br>

<img width="515" height="28" alt="imagen" src="https://github.com/user-attachments/assets/7419dc0d-2085-4af4-ad25-f76d9b3a06cd" />

<br>

<img width="590" height="22" alt="imagen" src="https://github.com/user-attachments/assets/8e86ed15-e96c-49a4-9be7-f273cfba777e" />

<br>

**Initialize the Samba AD DC service and check its status:**

```bash
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```

<br>

<img width="425" height="43" alt="imagen" src="https://github.com/user-attachments/assets/bf81ba96-adbf-41c5-98e7-aae772bca9ca" />

<br>

<img width="805" height="578" alt="imagen" src="https://github.com/user-attachments/assets/cf3a6fef-17a1-4848-b401-f5a1131e90be" />

<br>

---

## 🕐 Step 6 — Time Synchronization (Chrony)

Kerberos authentication is extremely time-sensitive — a difference of more than 5 minutes between the DC and any client will cause authentication failures. We configure Chrony as our NTP server.

<br>

First, change the permissions on the Samba NTP signing socket so that the Chrony service can access it:

<img width="551" height="36" alt="imagen" src="https://github.com/user-attachments/assets/679df16d-dcf3-4dc0-a57d-f90c370f1872" />

<br>

**Open the Chrony configuration file:**

```bash
nano /etc/chrony/chrony.conf
```

<br>

Add the following lines to point to the DNS server and allow NTP requests from the local subnet:

<img width="802" height="547" alt="imagen" src="https://github.com/user-attachments/assets/c45bda87-c6a5-475a-b388-c3a6e9b1752a" />

<br>

**Restart the service and verify it is running correctly:**

```bash
sudo systemctl restart chronyd
sudo systemctl status chronyd
```

<br>

<img width="403" height="44" alt="imagen" src="https://github.com/user-attachments/assets/b5083434-c798-4624-af01-14ffc8df1922" />

<br>

<img width="802" height="352" alt="imagen" src="https://github.com/user-attachments/assets/9ae21ca1-d1d7-4ce7-b330-76e6b7dfc0b1" />

<br>

---

## ✅ Step 7 — Verify the Domain

Once all services are running, we perform a series of verification checks to confirm that DNS, Kerberos, LDAP, and SMB are all working as expected.

<br>

**Verify that the domain A records resolve correctly:**

```bash
host -t A lab12.lan
```

<img width="310" height="54" alt="imagen" src="https://github.com/user-attachments/assets/247487de-c970-45fb-9eba-92fb4864e103" />

<br>

```bash
host -t A ls12.lab12.lan
```

<img width="346" height="57" alt="imagen" src="https://github.com/user-attachments/assets/ea8f1764-f027-491b-8985-17e6bb6a413d" />

<br>

**Verify that the Kerberos and LDAP SRV records point to the DC's FQDN:**

```bash
host -t SRV _kerberos._udp.lab12.lan
```

<img width="515" height="38" alt="imagen" src="https://github.com/user-attachments/assets/302eea64-0d36-40fb-b392-70df9eec6fde" />

<br>

```bash
host -t SRV _ldap._tcp.clockwork.local
```

<img width="491" height="41" alt="imagen" src="https://github.com/user-attachments/assets/2d2826f8-0a65-426a-8e34-a5bebe87294e" />

<br>

**List the available network shares on the AD server:**

```bash
smbclient -L clockwork.local -N
```

> You should see the default shares: `netlogon`, `sysvol`, and `IPC$`. Their presence confirms Samba is operating as an AD DC.

<br>

<img width="556" height="161" alt="imagen" src="https://github.com/user-attachments/assets/1d5e33aa-b4e7-43ef-9471-98e892ab747c" />

<br>

**Verify Kerberos authentication by requesting a ticket for the administrator:**

```bash
kinit administrator@LAB12.LAN
```

<img width="603" height="58" alt="imagen" src="https://github.com/user-attachments/assets/65c2f14b-cfbd-412e-a0db-99299a01f084" />

<br>

<img width="541" height="121" alt="imagen" src="https://github.com/user-attachments/assets/a5c53eb9-445a-4918-b86b-3d110ae71864" />

<br>

**Log in to the server via SMB to confirm end-to-end authentication works:**

```bash
sudo smbclient //localhost/netlogon -U 'administrator'
```

<img width="591" height="78" alt="imagen" src="https://github.com/user-attachments/assets/e43de933-1169-4b0c-8d7e-92bc33ab8730" />

<br>

**Set a secure password for the administrator account:**

```bash
sudo samba-tool user setpassword administrator
```

<img width="710" height="40" alt="imagen" src="https://github.com/user-attachments/assets/b3070cd2-72e4-494f-bd14-5ac07cbbdc5a" />

<br>

**Check the Samba configuration file for any syntax errors:**

```bash
testparm
```

<img width="450" height="666" alt="imagen" src="https://github.com/user-attachments/assets/b18f299c-b956-4958-a56d-4489de8dbbb8" />

<br>

**Verify the domain functional level (should show Windows AD DC 2008 or higher):**

```bash
sudo samba-tool domain level show
```

<img width="500" height="107" alt="imagen" src="https://github.com/user-attachments/assets/6b95ef8c-59e5-4d96-870c-7acd9c31af06" />

<br>

---

<br>

# 💻 SPRINT 2 — Linux and Windows Client Integration

> **Goal:** Join Linux and Windows client machines to the Samba Active Directory domain. This sprint covers installing the required client packages, configuring authentication, creating Organizational Units (OUs), managing users and groups, and enforcing Group Policy Objects (GPOs).

<br>

---

## 🔄 Step 1 — Update the System

Before installing any packages, ensure the system package index is up to date to avoid version conflicts.

```bash
sudo apt update
```

<img width="718" height="150" alt="imagen" src="https://github.com/user-attachments/assets/6f4f6456-baf5-4cd4-997c-9f9a413a53fa" />

<br>

```bash
sudo apt upgrade
```

<img width="1190" height="362" alt="imagen" src="https://github.com/user-attachments/assets/8a12c418-ac51-4d48-aa86-faed7b35be2c" />

<br>

---

## 🔐 Step 2 — Install and Verify SSH

Installing SSH allows remote management of the client machine from any other host on the network, which is essential for administration.

```bash
sudo apt-get install ssh
```

<img width="708" height="185" alt="imagen" src="https://github.com/user-attachments/assets/8745d1ea-bfbc-4469-8254-554181492278" />

<br>

**Verify that the SSH service is active and running:**

```bash
sudo systemctl status ssh
```

<img width="717" height="303" alt="imagen" src="https://github.com/user-attachments/assets/c11c775c-0c94-4d51-851d-04b056c48ee0" />

<br>

---

## ⚙️ Step 3 — Configure the Linux Client Machine

### 3.1 — Set the Hostname

Assign a unique hostname to the client machine. This will be the computer account name in the domain.

```bash
sudo hostnamectl set-hostname lc12
hostname -f
```

> `hostname -f` displays the FQDN to confirm the change was applied correctly.

<br>

<img width="460" height="72" alt="imagen" src="https://github.com/user-attachments/assets/25069b5f-c4d4-47de-9ac9-8b4045a32060" />

<br>

### 3.2 — Configure /etc/hosts

Map the domain controller's IP to its FQDN in the local hosts file so the client can resolve it even before joining the domain.

```bash
nano /etc/hosts
```

<img width="347" height="23" alt="imagen" src="https://github.com/user-attachments/assets/3ee4ed8b-172b-4ece-8d48-652026784e09" />

<br>

<img width="716" height="243" alt="imagen" src="https://github.com/user-attachments/assets/c05ac32f-439f-407f-9eb9-3d8ac8382dce" />

<br>

### 3.3 — Test Connectivity to the Domain

Before proceeding, verify that the client can reach the domain controller over the network.

```bash
ping lab12.lan
```

> A successful ping confirms DNS resolution and basic network connectivity to the DC.

<br>

<img width="663" height="152" alt="image" src="https://github.com/user-attachments/assets/845a1d93-cd53-4fb8-860e-e07a93489425" />

<br>

---

## 🕐 Step 4 — Synchronize Time with the Domain Controller

Kerberos requires that all machines in the domain have their clocks synchronized. Install `ntpdate` to sync the client clock with the DC.

```bash
sudo apt-get install ntpdate
```

<img width="713" height="427" alt="imagen" src="https://github.com/user-attachments/assets/7dc08504-a255-4d04-8e5c-2c6236345ab2" />

<br>

**Query and sync the time from the domain controller:**

```bash
sudo ntpdate -q lab12.lan
sudo ntpdate lab12.lan
```

<img width="767" height="94" alt="imagen" src="https://github.com/user-attachments/assets/7d76e4a1-b8c6-4ea3-b908-e80dd6eaa96f" />

<br>

---

## 📦 Step 5 — Install Domain Integration Packages

Install all the packages needed to join the Linux client to the Samba Active Directory domain.

```bash
sudo apt-get install samba krb5-config krb5-user winbind libpam-winbind libnss-winbind
```

> - **samba** — Core Samba suite for SMB file sharing
> - **krb5-config / krb5-user** — Kerberos client configuration and tools
> - **winbind** — Resolves Windows domain users and groups on Linux
> - **libpam-winbind / libnss-winbind** — PAM and NSS integration for AD authentication

<br>

<img width="882" height="23" alt="imagen" src="https://github.com/user-attachments/assets/3c583e93-afa7-4d85-b9e2-b92c7ff5609b" />

<br>

<img width="768" height="199" alt="imagen" src="https://github.com/user-attachments/assets/b702a6f7-29b0-49a7-8575-866cc959bcf5" />

<br>

<img width="617" height="196" alt="imagen" src="https://github.com/user-attachments/assets/c52ba31b-8a79-4cc6-a450-1f96b6dba191" />

<br>

<img width="616" height="184" alt="imagen" src="https://github.com/user-attachments/assets/7c6d9a3c-66ec-4238-b29b-788c83a1c6fe" />

<br>

---

## 🔑 Step 6 — Kerberos Authentication Test

Before joining the domain, verify that Kerberos is configured correctly by requesting a ticket for the domain administrator.

```bash
kinit administrator@LAB12.LAN
```

<img width="594" height="57" alt="imagen" src="https://github.com/user-attachments/assets/bbe0e8d6-a145-4305-9334-7fcaffa4f749" />

<br>

**List the active Kerberos tickets to confirm the TGT was granted:**

```bash
klist
```

<img width="520" height="119" alt="imagen" src="https://github.com/user-attachments/assets/0405eebe-9aeb-4fe1-9341-9ea3aec70157" />

<br>

---

## 📝 Step 7 — Configure Samba as a Domain Member (smb.conf)

The client's Samba configuration must be set up as a domain member (not a standalone server). First, back up the existing config file.

```bash
mv /etc/samba/smb.conf /etc/samba/smb.conf.initial
```

<img width="491" height="20" alt="imagen" src="https://github.com/user-attachments/assets/1e1fde2e-991e-472c-b80e-cea68e656273" />

<br>

**Create a new smb.conf file:**

```bash
nano /etc/samba/smb.conf
```

<img width="422" height="17" alt="imagen" src="https://github.com/user-attachments/assets/6f795203-931c-4f2b-ac78-7b1a0f005ff3" />

<br>

**Add the following configuration:**

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

> - `security = ADS` — Sets Samba to authenticate against an Active Directory server
> - `template homedir` — Automatically creates home directories for domain users on first login
> - `winbind use default domain = true` — Allows users to log in without specifying the domain prefix

<br>

<img width="732" height="386" alt="imagen" src="https://github.com/user-attachments/assets/a1ad0024-b608-4669-b422-be1df542a250" />

<br>

---

## 🔄 Step 8 — Manage Samba Services

**Restart Samba daemons to apply the new configuration:**

```bash
sudo systemctl restart smbd nmbd
```

<img width="448" height="22" alt="imagen" src="https://github.com/user-attachments/assets/f31b2edb-98b2-4852-8404-7b7d35c33d5e" />

<br>

**Stop the AD DC service (not needed on the client):**

```bash
sudo systemctl stop samba-ad-dc
```

<img width="443" height="23" alt="imagen" src="https://github.com/user-attachments/assets/f3d6f5b9-de2e-4c7e-a46e-1dccca5e6ce2" />

<br>

**Enable smbd and nmbd to start automatically on boot:**

```bash
sudo systemctl enable smbd nmbd
```

<img width="789" height="115" alt="imagen" src="https://github.com/user-attachments/assets/40e8baea-f2ca-4ccf-8d62-90a8cb3c05d8" />

<br>

---

## 🔗 Step 9 — Join the Domain

This is the key step — we use `net ads join` to register the Ubuntu Desktop as a computer account in the Active Directory domain.

```bash
sudo net ads join -U administrator
```

> You will be prompted for the administrator password. On success, you'll see a message like `Joined 'LCxx' to dns domain 'lab12.lan'`.

<br>

<img width="508" height="26" alt="imagen" src="https://github.com/user-attachments/assets/3b2bb6cc-2744-430e-a2d7-ccd375f9a1bc" />

<br>

**Verify the computer account was created in the domain:**

```bash
sudo samba-tool computer list
```

<img width="378" height="69" alt="imagen" src="https://github.com/user-attachments/assets/a01d1e3c-f5d4-4ff6-86ee-fa2bb2c84eaa" />

<br>

**Confirm the domain join is valid from the client side:**

```bash
net ads testjoin
```

> Expected output: `Join is OK`

<br>

<img width="366" height="44" alt="imagen" src="https://github.com/user-attachments/assets/a041644d-4ec4-4a4c-8994-9337983a8d8c" />

<br>

---

## 🛠️ Step 10 — Configure AD Account Authentication

### 10.1 — Edit the NSS Configuration

NSS (Name Service Switch) needs to know to look up users and groups from Winbind (the AD source) in addition to local files.

```bash
sudo nano /etc/nsswitch.conf
```

> In the `passwd` and `group` lines, add `winbind` at the end. This tells the system to fall back to Active Directory for any user/group not found locally.

<br>

<img width="425" height="28" alt="imagen" src="https://github.com/user-attachments/assets/b54fbe04-95e4-4777-ae2f-ee24d24b1a54" />

<br>

<img width="793" height="433" alt="imagen" src="https://github.com/user-attachments/assets/66a97ad6-0543-4690-a904-fe784cc8aa82" />

<br>

### 10.2 — Restart Winbind

```bash
sudo systemctl restart winbind
```

<img width="433" height="19" alt="imagen" src="https://github.com/user-attachments/assets/6bd46a2c-e4e6-4c6f-9b01-5192b10c9838" />

<br>

### 10.3 — Verify Domain User and Group Integration

```bash
wbinfo -u    # List all domain users
wbinfo -g    # List all domain groups
```

<img width="322" height="364" alt="imagen" src="https://github.com/user-attachments/assets/c7bcc3ed-fb1e-4cea-932d-955fc72ce5eb" />

<br>

**Verify with getent that domain accounts are visible to the OS:**

```bash
sudo getent passwd | grep administrator
sudo getent group | grep 'domain admins'
id administrator
```

<img width="517" height="33" alt="imagen" src="https://github.com/user-attachments/assets/95b19adc-a219-4149-b6e1-7b84199324b3" />

<br>

<img width="493" height="36" alt="imagen" src="https://github.com/user-attachments/assets/29377847-b312-491b-b851-243d74657580" />

<br>

<img width="795" height="75" alt="imagen" src="https://github.com/user-attachments/assets/e54074c0-8cd9-4c56-baf7-712fb0b5cf42" />

<br>

---

## 👥 Step 11 — Configure PAM for Domain Login

PAM (Pluggable Authentication Modules) must be configured to allow domain users to log in and automatically create home directories on first login.

```bash
sudo pam-auth-update
```

> Select the options for Winbind and automatic home directory creation.

<br>

<img width="350" height="21" alt="imagen" src="https://github.com/user-attachments/assets/af2a2bf9-b546-4041-9e0f-0dd874cf5000" />

<br>

<img width="757" height="445" alt="imagen" src="https://github.com/user-attachments/assets/9550ef33-07bc-4535-a19a-4452f580b0a6" />

<br>

**Edit the /etc/pam.d/common-account file to automatically create home directories:**

```bash
sudo nano /etc/pam.d/common-account
```

<img width="487" height="24" alt="imagen" src="https://github.com/user-attachments/assets/d62dd0bb-be12-4df5-a74a-24218b4482de" />

<br>

Add this line at the end of the file:

```
session required pam_mkhomedir.so skel=/etc/skel/ umask=0022
```

> This ensures that when a domain user logs in for the first time, a personal home directory is automatically created using the default skeleton files.

<br>

<img width="786" height="532" alt="imagen" src="https://github.com/user-attachments/assets/cc293e06-9297-4a46-b06e-c7eb03e7379c" />

<br>

**Test login with a domain account via the terminal:**

```bash
su administrator
```

<img width="314" height="56" alt="imagen" src="https://github.com/user-attachments/assets/09dc6311-0fe8-4b0a-9486-51756e6f3be9" />

<br>

**Authenticate with GUI — the login screen should now accept domain credentials:**

<img width="376" height="304" alt="imagen" src="https://github.com/user-attachments/assets/f93ecbd0-f72b-45df-8b78-7bd7f78fc32b" />

<br>

<img width="368" height="228" alt="imagen" src="https://github.com/user-attachments/assets/2201485a-1c2e-4724-939d-57872451b735" />

<br>

<img width="489" height="74" alt="imagen" src="https://github.com/user-attachments/assets/1405ea13-b647-4546-9346-570b5dd731b6" />

<br>

<img width="613" height="483" alt="imagen" src="https://github.com/user-attachments/assets/8be61fb0-a733-41c2-a4b2-7a765df495f0" />

<br>

<img width="609" height="481" alt="imagen" src="https://github.com/user-attachments/assets/5ea4733b-6137-4e58-b857-f14c1d1ab896" />

<br>

<img width="612" height="481" alt="imagen" src="https://github.com/user-attachments/assets/70fd1e12-8126-4fb4-834f-8b3f26bb1c26" />

<br>

<img width="611" height="477" alt="imagen" src="https://github.com/user-attachments/assets/a82d7d9a-8a55-4cd6-b5c3-d98ce2220a9b" />

<br>

<img width="407" height="482" alt="imagen" src="https://github.com/user-attachments/assets/d8c410d4-c0b4-4dfb-9b68-5cd976326a1d" />

<br>

---

## 👤 Step 12 — Create Users and Groups in Active Directory

Now we populate the domain with user accounts, security groups, and Organizational Units (OUs) to represent our organization's structure.

<br>

### 12.1 — Create Users

```bash
sudo samba-tool user create Alice --userou="OU=IT_Department"
```

<img width="663" height="77" alt="imagen" src="https://github.com/user-attachments/assets/23515350-6a24-4771-aafb-fa9e324e0f74" />

<br>

```bash
sudo samba-tool user create Bob --userou="OU=Students"
```

<img width="601" height="70" alt="imagen" src="https://github.com/user-attachments/assets/44ad168f-9be1-4157-9505-365f9fc04114" />

<br>

```bash
sudo samba-tool user create Charlie --userou="OU=Students"
```

<img width="626" height="68" alt="imagen" src="https://github.com/user-attachments/assets/5946eb66-0bab-4c26-b946-3d219d400e91" />

<br>

### 12.2 — Create Groups

```bash
sudo samba-tool group add IT_Admins
```

<img width="435" height="25" alt="image" src="https://github.com/user-attachments/assets/551a9e87-754e-4e29-839c-983f36c265ad" />

<br>

```bash
sudo samba-tool group add Students
```

<img width="427" height="23" alt="image" src="https://github.com/user-attachments/assets/8cdadbf0-1d3b-4066-b87d-35a172025cc7" />

<br>

### 12.3 — Add Users to Groups

```bash
sudo samba-tool group addmembers IT_Admins Alice
sudo samba-tool group addmembers Students Bob,Charlie
```

<img width="542" height="27" alt="image" src="https://github.com/user-attachments/assets/982c6b7c-ad7d-4d5d-a216-e356fe25215e" />

<br>

<img width="585" height="22" alt="image" src="https://github.com/user-attachments/assets/70b59cc6-e834-4119-9cc0-ca415971a4c1" />

<br>

### 12.4 — Create the Organizational Unit (OU) Hierarchy

OUs are containers in Active Directory used to organize users, groups, and computers. They also allow GPOs to be applied selectively to different parts of the organization.

```bash
sudo samba-tool ou add "OU=IT_Department,DC=lab12,DC=lan"
sudo samba-tool ou add "OU=Students,DC=lab12,DC=lan"
sudo samba-tool ou add "OU=HR_Department,DC=lab12,DC=lan"
```

<img width="628" height="36" alt="image" src="https://github.com/user-attachments/assets/6fe6093e-7216-42ad-8c34-1d458ac5f202" />

<br>

<img width="593" height="32" alt="image" src="https://github.com/user-attachments/assets/c8d3c92f-49c6-4597-9eaf-11d89a28683c" />

<br>

<img width="632" height="35" alt="image" src="https://github.com/user-attachments/assets/66847513-2859-458c-96dd-81113129f2d1" />

<br>

---

## 🔒 Step 13 — Group Policy Objects (GPOs)

GPOs allow administrators to enforce security and configuration settings across all machines and users in the domain from a central location.

<br>

### 13.1 — Password Policy

Enforce a minimum password length of 8 characters with complexity requirements enabled for all domain users:

```bash
sudo samba-tool domain passwordsettings set --min-pwd-length=8 --complexity=on
```

<img width="784" height="130" alt="imagen" src="https://github.com/user-attachments/assets/4d484726-1ff2-42cd-a799-8e92e9b7e8e8" />

<br>

### 13.2 — Account Lockout Policy

To mitigate brute-force attacks, configure an account lockout policy: **3 failed login attempts trigger a 5-minute account lockout**.

**Step 1 — Set the lockout threshold (3 attempts):**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-threshold=3
```

<img width="742" height="55" alt="imagen" src="https://github.com/user-attachments/assets/1585bde3-d978-49ba-ace5-3532cb3874c3" />

<br>

**Step 2 — Set the lockout duration (5 minutes):**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-duration=5
```

<img width="731" height="53" alt="imagen" src="https://github.com/user-attachments/assets/f3d5a0e6-1a31-4a10-b6f2-5c880cb3b099" />

<br>

**Step 3 — Set the reset counter timer (5 minutes):**

```bash
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=5
```

<img width="731" height="51" alt="imagen" src="https://github.com/user-attachments/assets/7880d848-5a6c-4534-a8c2-bc1a031c7efb" />

<br>

### 13.3 — Verification and Testing

**Account Lockout Test:**

To validate that the policy is working, a brute-force simulation was performed:

1. Attempted to log in with user **Bob** using an incorrect password.
2. Repeated the failed login attempt **3 times** to trigger the lockout threshold.
3. On the **4th attempt**, the system displayed an account lockout message, confirming the policy is active:

<br>

<img width="796" height="719" alt="imagen" src="https://github.com/user-attachments/assets/82c44e16-1e08-4aea-97e2-f7cdc99302b3" />

<br>

4. After waiting **5 minutes**, the account was automatically unlocked — confirming the lockout duration setting is also functioning correctly.

<br>

**Verify the OU structure:**

```bash
sudo samba-tool ou list
```

<img width="358" height="104" alt="imagen" src="https://github.com/user-attachments/assets/6ffa1252-2d2b-4f6d-9c2f-05d5d44005c0" />

<br>

**Audit user placement and check Distinguished Names:**

```bash
sudo samba-tool user show Alice | grep dn
```

<img width="507" height="106" alt="imagen" src="https://github.com/user-attachments/assets/e89984d2-9745-4b93-9303-36af4480e04e" />

<br>

**Validate group memberships:**

```bash
sudo samba-tool group listmembers IT_Admins
sudo samba-tool group listmembers Students
```

<img width="591" height="86" alt="imagen" src="https://github.com/user-attachments/assets/ae88b763-ca0b-4359-b2f5-72bc19de5bca" />

<br>

**Review all active security policy settings:**

```bash
sudo samba-tool domain passwordsettings show
```

<img width="513" height="205" alt="imagen" src="https://github.com/user-attachments/assets/49879fd8-6c84-42f0-87e1-858cfdb4bffc" />

<br>

---

<br>

# 💾 SPRINT 3 — Storage, Shares & Permissions

> **Goal:** Add dedicated storage to the Domain Controller, create a structured directory hierarchy for departmental file shares, configure Samba network shares with appropriate access controls, and apply advanced ACL-based permissions to protect sensitive data.

<br>

---

## 🔍 Connectivity Verification

Before making any changes, confirm the domain is healthy and the DC is reachable.

```bash
sudo samba-tool domain info 127.0.0.1
```

<img width="458" height="148" alt="image" src="https://github.com/user-attachments/assets/ae1e256c-0dde-47e5-83f4-61b10f8e999d" />

<br>

**Ping the domain to verify DNS resolution is working:**

```bash
ping -c 3 lab12.lan
```

<img width="597" height="152" alt="image" src="https://github.com/user-attachments/assets/6c2d4c83-c52f-43b6-b225-90d6383f42aa" />

<br>

---

## 1. 🗄️ Storage Management

### Step 1.1 — Create the New Disk

Add a new virtual disk to the server via the hypervisor settings. We recommend **20GB or 40GB** for a lab environment.

<img width="1025" height="595" alt="image" src="https://github.com/user-attachments/assets/361a78ba-7147-4173-ab4d-44d88ed749b7" />

<br>

### Step 1.2 — Identify the New Disk

After adding the disk, confirm it is visible to the operating system. The new disk will typically appear as `/dev/sdb`.

```bash
lsblk
```

<img width="582" height="230" alt="image" src="https://github.com/user-attachments/assets/c4e68074-9f56-497b-b1a0-f4975e2638fb" />

<br>

### Step 1.3 — Create a Partition Table

Use `fdisk` to create a new partition on the disk.

```bash
sudo fdisk /dev/sdb
```

Follow the interactive prompt with these commands:

| Command | Action |
|---------|--------|
| `n` | Create a new partition |
| `p` | Set it as a primary partition |
| `1` | Assign partition number 1 |
| `Enter` × 2 | Accept the default start and end sectors (use the full disk) |
| `w` | Write the changes and exit |

<br>

<img width="677" height="397" alt="image" src="https://github.com/user-attachments/assets/78e3d80b-87b1-4671-bbd2-7cd360d7f362" />

<br>

### Step 1.4 — Format the Filesystem

Samba requires **EXT4** to correctly support ACLs (Access Control Lists) and extended attributes used by Active Directory.

```bash
sudo mkfs.ext4 /dev/sdb1
```

<img width="520" height="182" alt="image" src="https://github.com/user-attachments/assets/aaa52e39-9562-45f1-9d2f-fd68049fb510" />

<br>

### Step 1.5 — Configure a Permanent Mount Point

**1. Create the mount directory:**

```bash
sudo mkdir -p /srv/samba/Data
```

<img width="391" height="27" alt="image" src="https://github.com/user-attachments/assets/631bf1b6-dcb7-47c5-8f64-019291f3122c" />

<br>

**2. Find the disk UUID for persistent mounting:**

Using the UUID (rather than the device path `/dev/sdb1`) ensures the mount point remains consistent even if the disk order changes after a reboot.

```bash
sudo blkid /dev/sdb1
```

<img width="802" height="55" alt="image" src="https://github.com/user-attachments/assets/43f930b2-e609-4df1-904d-330d894943a0" />

<br>

**3. Add the entry to /etc/fstab:**

```bash
sudo nano /etc/fstab
```

Add the following line at the end of the file (replace `your-uuid-here` with the actual UUID):

```
UUID=your-uuid-here /srv/samba/Data ext4 user_xattr,acl,barrier=1 1 1
```

> The options `user_xattr` and `acl` are required by Samba for extended attribute and ACL support.

<br>

<img width="931" height="283" alt="image" src="https://github.com/user-attachments/assets/621a52c0-f4cd-41e6-b168-72ab2261e54c" />

<br>

**4. Mount all filesystems from fstab and verify:**

```bash
sudo mount -a
df -h /srv/samba/Data
```

<img width="490" height="62" alt="image" src="https://github.com/user-attachments/assets/4b3b45c1-ff0c-4920-88fd-f18b69079795" />

<br>

### Storage Verification Checks

**1. Verify devices and partitions:**

```bash
lsblk
```

<img width="620" height="298" alt="image" src="https://github.com/user-attachments/assets/17e3fa2e-3e02-481e-8e41-f37aa518d58e" />

<br>

**2. Check mount point status and available space:**

```bash
df -h /srv/samba/Data
```

<img width="510" height="66" alt="image" src="https://github.com/user-attachments/assets/d3e1f9a9-211c-4e5a-a253-958bdb5def92" />

<br>

**3. Confirm persistence (entry exists in fstab):**

```bash
cat /etc/fstab | grep Data
```

<img width="851" height="40" alt="image" src="https://github.com/user-attachments/assets/3c3c326b-c035-432a-8f5b-ea7394c57ef4" />

<br>

**4. Validate that ACL and extended attribute options are active:**

```bash
findmnt -n -o OPTIONS /srv/samba/Data
```

<img width="517" height="47" alt="image" src="https://github.com/user-attachments/assets/ef509aaa-9e02-4ea4-93d9-5f1e422a4a53" />

<br>

---

## 2. 📁 Directory Structure and Network Shares

### Step 2.1 — Create the Directory Hierarchy

Create separate directories for each department's file share. These folders will be the targets of our Samba share definitions.

```bash
sudo mkdir -p /srv/samba/Data/FinanceDocs
sudo mkdir -p /srv/samba/Data/HRDocs
sudo mkdir -p /srv/samba/Data/Public
```

<img width="555" height="86" alt="image" src="https://github.com/user-attachments/assets/aadd8f37-c0b1-4d56-96fa-7dfe4c1051aa" />

<br>

**Verify the directory structure was created correctly:**

```bash
ls -l /srv/samba/Data
```

<img width="482" height="125" alt="image" src="https://github.com/user-attachments/assets/9379cb03-c380-4bd4-988b-35ac13e3f3d1" />

<br>

### Step 2.2 — Configure Samba Network Shares

**1. Open the Samba configuration file:**

```bash
sudo nano /etc/samba/smb.conf
```

<br>

**2. Add the share definitions at the bottom of the file:**

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

> - `valid users` restricts access to specific groups
> - `create mask` and `directory mask` control the default permissions for new files and folders
> - `write list` in the `[Public]` share allows only Domain Admins to write, while all domain users can read

<br>

<img width="562" height="862" alt="image" src="https://github.com/user-attachments/assets/ba062069-d25b-43fa-963d-30b9f811896f" />

<br>

**3. Validate the configuration for syntax errors:**

```bash
testparm
```

<img width="467" height="132" alt="image" src="https://github.com/user-attachments/assets/13e1c51c-e60d-4585-bc3d-dcd2fb24af43" />

<br>

**4. Restart the Samba service to apply the changes:**

```bash
sudo systemctl restart samba-ad-dc
```

<img width="481" height="22" alt="image" src="https://github.com/user-attachments/assets/9356308d-3935-4b3f-8143-f7e59e4e8d83" />

<br>

### Step 2.3 — Verify the Network Shares

**List the available shares locally to confirm they appear:**

```bash
smbclient -L localhost -N
```

<img width="627" height="247" alt="image" src="https://github.com/user-attachments/assets/3e607ae5-571a-4f67-ac73-04cec65797ed" />

<br>

**Check directory ownership and permissions:**

```bash
ls -ld /srv/samba/Data/*
```

<img width="632" height="103" alt="image" src="https://github.com/user-attachments/assets/1d984dde-e425-4f85-9b01-3aad1b874891" />

<br>

---

## 3. 🔐 Advanced Permissions and ACLs

### Step 3.1 — Install ACL Support

ACLs (Access Control Lists) allow fine-grained control over file permissions beyond the standard Linux user/group/other model. This is required for proper Samba AD permission enforcement.

```bash
sudo apt update && sudo apt install acl -y
```

<br>

---

## 4. ⚙️ Task and Process Management

> This section covers the essential Linux tools for monitoring system activity, managing services, and controlling processes — all critical skills for maintaining a healthy Samba AD DC.

<br>

### 4.1 — The `top` Utility

`top` provides a dynamic, real-time view of all running processes and resource consumption. It updates every few seconds and is the quickest way to spot runaway processes.

Key columns to watch:
- **PID** — Unique process identifier
- **%CPU** — Percentage of CPU being consumed
- **%MEM** — Percentage of RAM being used
- **Command** — Name of the program or script

```bash
top
```

<img width="185" height="20" alt="imagen" src="https://github.com/user-attachments/assets/14bed00a-c0bd-45e3-80cc-4b15ac4fb3b3" />

<br>

<img width="801" height="371" alt="imagen" src="https://github.com/user-attachments/assets/197661f5-aee2-4a27-9fe2-88422b196d36" />

<br>

### 4.2 — The `htop` Utility

`htop` is an enhanced, more user-friendly alternative to `top`. It features color coding, mouse support, and the ability to scroll horizontally to see full command lines.

```bash
htop
```

<img width="634" height="134" alt="imagen" src="https://github.com/user-attachments/assets/0b79537a-2c61-432f-8539-d9dd3d074ab0" />

<br>

<img width="798" height="364" alt="imagen" src="https://github.com/user-attachments/assets/8ffee18a-3ef0-4983-9f06-b322e4dc6b58" />

<br>

**Key interactions in htop:**

| Key | Action |
|-----|--------|
| `F2` | Open Setup to customize columns and colors |
| `F6` | Sort processes by CPU, Memory, or Priority |
| `F9` | Kill a selected process by sending a signal |

<br>

### 4.3 — The `ps` Command

`ps` (Process Status) is a non-interactive snapshot of currently running processes. It's useful in scripts and for piping output to `grep`.

```bash
# Show all running processes in user-friendly format
ps aux

# Filter specifically for Samba-related processes
ps aux | grep samba
```

> `ps aux | grep samba` is particularly useful to verify all Samba daemons (smbd, nmbd, samba, winbindd) are running.

<br>

<img width="795" height="245" alt="imagen" src="https://github.com/user-attachments/assets/d9dad8a9-70d3-4342-ad55-df73a9328e02" />

<br>

### 4.4 — Service Management with systemctl

```bash
# Check if the service is active and review recent logs
sudo systemctl status samba-ad-dc
```

<img width="802" height="399" alt="imagen" src="https://github.com/user-attachments/assets/adb7f9ee-c7d6-47f2-9dff-fb81baa2f535" />

<br>

```bash
# Restart a service that is unresponsive or stuck
sudo systemctl restart samba-ad-dc
```

<img width="426" height="20" alt="imagen" src="https://github.com/user-attachments/assets/d2921b5e-1fba-40a5-b692-1cf94f4d64a5" />

<br>

```bash
# Ensure the service starts automatically after every reboot
sudo systemctl enable samba-ad-dc
```

<img width="800" height="69" alt="imagen" src="https://github.com/user-attachments/assets/fbbb5ded-cde1-4a74-8aad-7525276ac856" />

<br>

```bash
# List all currently active services on the system
systemctl list-units --type=service --state=running
```

<img width="798" height="469" alt="imagen" src="https://github.com/user-attachments/assets/7302992d-1513-424d-8528-9a03744609bb" />

<br>

### 4.5 — Sending Signals to Processes

Linux uses signals to communicate with processes. Here are the most common ones:

```bash
# Gracefully terminate a process (SIGTERM — allows cleanup)
kill <PID>

# Forcefully kill a process immediately (SIGKILL — no cleanup)
kill -9 <PID>

# Kill all processes with a specific name
sudo killall smbd
```

<br>

### 4.6 — Practical Exercise: Controlling a Process

**Run the `sl` command (Steam Locomotive — a fun terminal animation):**

<img width="797" height="433" alt="imagen" src="https://github.com/user-attachments/assets/81ee956f-97c3-4203-880c-674755a4aa2b" />

<br>

**Find the PID of the running process:**

```bash
pgrep -sl
```

<img width="268" height="126" alt="imagen" src="https://github.com/user-attachments/assets/acc2b3c5-6bd7-4472-8a9d-ede1c98a371a" />

<br>

**Stop (pause) the process using SIGSTOP:**

```bash
kill -19 <PID>
```

> Signal 19 is `SIGSTOP` — it pauses the process without terminating it. The animation will freeze.

<br>

<img width="288" height="26" alt="imagen" src="https://github.com/user-attachments/assets/612c9297-1d6d-4246-817a-aa366baa9cef" />

<br>

<img width="657" height="291" alt="imagen" src="https://github.com/user-attachments/assets/1fb29639-cc40-4409-aaa2-a540d5a5485c" />

<br>

**Resume the process using SIGCONT:**

```bash
kill -18 <PID>
```

> Signal 18 is `SIGCONT` — it resumes a paused process from where it left off.

<br>

<img width="1563" height="473" alt="imagen" src="https://github.com/user-attachments/assets/80e19292-b10e-41c4-8ebf-1dba21e7f671" />

<br>

---

<br>

# 🤝 SPRINT 4 — Domain Trust

> **Goal:** Establish a **forest trust** between two separate Samba Active Directory domains (`lab12.lan` and `lab120.lan`). This allows users from one domain to authenticate and access resources in the other, enabling cross-domain collaboration while keeping each domain administratively independent.

<br>

---

## Step 1 — Install and Configure the Second Domain Controller (LS120)

A second Ubuntu Server is provisioned as a completely separate AD DC for the `lab120.lan` domain. This machine goes through the same initial setup process as Sprint 1.

<br>

<img width="801" height="512" alt="image" src="https://github.com/user-attachments/assets/e485bfc4-b333-436a-a402-f5399eea63a8" />

<br>

<img width="801" height="512" alt="image" src="https://github.com/user-attachments/assets/52e791f3-6942-4936-b1fb-8b16f111e492" />

<br>

<img width="798" height="320" alt="image" src="https://github.com/user-attachments/assets/440d3c14-49c0-4242-8bf5-fe312c9f06e1" />

<br>

<img width="799" height="595" alt="imagen" src="https://github.com/user-attachments/assets/d5b3436b-e980-4fff-9db1-c5cb8feb5bc4" />

<br>

---

## Step 2 — Network Configuration for LS120

**Edit the Netplan configuration file:**

```bash
nano /etc/netplan/00-installer-config.yaml
```

<img width="616" height="168" alt="imagen" src="https://github.com/user-attachments/assets/f3debf6c-2daf-4552-848e-e246bb7df90d" />

<br>

**Apply the network changes:**

```bash
sudo netplan apply
```

<img width="799" height="231" alt="imagen" src="https://github.com/user-attachments/assets/d2d54bb5-0b57-452e-ae68-22265328c44d" />

<br>

<img width="800" height="339" alt="imagen" src="https://github.com/user-attachments/assets/8cb90cd3-b4d4-4c00-97ab-d1c1bb2ca4a7" />

<br>

**Remove the symbolic link to /etc/resolv.conf and reconfigure it:**

```bash
sudo unlink /etc/resolv.conf
```

<img width="415" height="18" alt="imagen" src="https://github.com/user-attachments/assets/6e52f4d5-7961-4d9a-847f-49aca9786fe1" />

<br>

<img width="796" height="598" alt="imagen" src="https://github.com/user-attachments/assets/c0077bf7-e2dc-4c54-9d8f-e212ce34dd27" />

<br>

<img width="437" height="21" alt="imagen" src="https://github.com/user-attachments/assets/1622c6e4-9825-4d69-a832-b88d187f2251" />

<br>

---

## Step 3 — Install Samba on LS120

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 \
  krb5-config krb5-user dnsutils chrony net-tools
```

<br>

Kerberos realm configuration for `LAB120.LAN`:

<img width="907" height="285" alt="image" src="https://github.com/user-attachments/assets/36ce743a-b587-41fc-acf4-e92bc4e56a56" />

<br>

`ls120.lab120.lan` hostname configuration:

<img width="907" height="218" alt="image" src="https://github.com/user-attachments/assets/91e70db1-92a4-4ed2-9f7c-d3cd18b7b24f" />

<br>

<img width="913" height="222" alt="image" src="https://github.com/user-attachments/assets/51399d25-3dd9-420e-9884-83beac40af2c" />

<br>

---

## Step 4 — Disable Unneeded Services and Enable the AD DC

**Stop and disable the member-server daemons** (they are not needed on a Domain Controller):

```bash
sudo systemctl disable --now smbd nmbd winbind
```

<img width="923" height="138" alt="image" src="https://github.com/user-attachments/assets/2d76573f-b3ca-4d3d-804a-e1c93f3f336e" />

<br>

**Enable only samba-ad-dc, which handles all AD and DC functionality:**

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
```

<img width="516" height="45" alt="image" src="https://github.com/user-attachments/assets/e8d4dfe9-3051-4d91-b1b5-12adfb666e77" />

<br>

<img width="937" height="125" alt="image" src="https://github.com/user-attachments/assets/6fa38068-74eb-4a50-878a-c4d01a6b407b" />

<br>

---

## Step 5 — Provision Samba AD on LS120

**Back up the default smb.conf:**

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
```

<img width="678" height="21" alt="image" src="https://github.com/user-attachments/assets/490745f2-8e2f-4c00-8ff3-ba3fb8676700" />

<br>

**Run the Samba domain provisioning wizard:**

```bash
sudo samba-tool domain provision
```

Enter the following values when prompted:

```
Realm:       LAB120.LAN
Domain:      LAB120
Server Role: dc
DNS backend: SAMBA_INTERNAL
DNS forwarder IP address: (leave blank or enter an external DNS)
```

<img width="941" height="257" alt="image" src="https://github.com/user-attachments/assets/ab455bb6-4c07-404a-be91-d7d74d35a1f3" />

<br>

**Replace the default Kerberos config with Samba's generated one:**

```bash
sudo mv /etc/krb5.conf /etc/krb5.conf.orig
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

<img width="597" height="20" alt="image" src="https://github.com/user-attachments/assets/7b9042f4-a44a-4e0b-94d9-67e4d5a1774e" />

<br>

<img width="702" height="23" alt="image" src="https://github.com/user-attachments/assets/105b68aa-5f7b-4c10-a53d-fce162c01af3" />

<br>

**Start and verify the service:**

```bash
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```

<img width="488" height="28" alt="image" src="https://github.com/user-attachments/assets/a70b9a6f-3402-4af8-9ec3-8d38c84b5c31" />

<br>

<img width="942" height="577" alt="image" src="https://github.com/user-attachments/assets/de81b232-1c96-4c49-938c-f9e7807fec44" />

<br>

---

## Step 6 — Time Synchronization for LS120

**Configure Chrony NTP permissions for Samba:**

```bash
sudo chown root:_chrony /var/lib/samba/ntp_signd/
sudo chmod 750 /var/lib/samba/ntp_signd/
```

<img width="657" height="30" alt="image" src="https://github.com/user-attachments/assets/6f13422d-1863-4b56-a9d8-8229e61592f5" />

<br>

<img width="570" height="25" alt="image" src="https://github.com/user-attachments/assets/8c92ed38-2fec-4fbf-b56f-fc0617e40785" />

<br>

**Edit the Chrony configuration file:**

```bash
sudo nano /etc/chrony/chrony.conf
```

<img width="513" height="22" alt="image" src="https://github.com/user-attachments/assets/40ab6841-c6cd-4f9a-a5f4-91ef064a2a63" />

<br>

Add the following lines to allow NTP clients on the local subnet and point to the Samba NTP socket:

```
bindcmdaddress 192.168.30.42
allow 192.168.30.0/24
ntpsigndsocket /var/lib/samba/ntp_signd
```

<img width="941" height="933" alt="image" src="https://github.com/user-attachments/assets/e996678b-853e-47f6-9871-6bcdd8d6384f" />

<br>

**Restart and verify:**

```bash
sudo systemctl restart chronyd
sudo systemctl status chronyd
```

<img width="472" height="23" alt="image" src="https://github.com/user-attachments/assets/fa7f0092-fb8a-428a-9453-ff4a6a8ce5e2" />

<br>

<img width="937" height="512" alt="image" src="https://github.com/user-attachments/assets/079a9d29-1568-448d-943b-fc6481f47d0b" />

<br>

---

## Step 7 — Verify Samba AD on LS120

Run the same verification checks performed in Sprint 1, but for the new `lab120.lan` domain.

```bash
host -t A lab120.lan
host -t A ls120.lab120.lan
```

<img width="393" height="71" alt="image" src="https://github.com/user-attachments/assets/0dfaf3ae-23aa-4d4c-9c98-1d069682c802" />

<br>

<img width="452" height="67" alt="image" src="https://github.com/user-attachments/assets/6ca673af-87bf-4442-8f39-328e771a457b" />

<br>

```bash
host -t SRV _kerberos._udp.lab120.lan
host -t SRV _ldap._tcp.lab120.lan
```

<img width="613" height="42" alt="image" src="https://github.com/user-attachments/assets/78cb578b-0773-4205-b170-34928b50ba79" />

<br>

<img width="582" height="51" alt="image" src="https://github.com/user-attachments/assets/fce9dde7-6dff-4d0b-ad72-8cef956b7fea" />

<br>

```bash
smbclient -L clockwork.local -N
```

<img width="617" height="191" alt="image" src="https://github.com/user-attachments/assets/5cc241c6-10da-4984-a95a-362c73358c51" />

<br>

```bash
kinit administrator@CLOCKWORK.LOCAL
klist
```

<img width="617" height="191" alt="image" src="https://github.com/user-attachments/assets/3219284b-18bc-41f7-9c07-b8f588f77723" />

<br>

<img width="606" height="142" alt="image" src="https://github.com/user-attachments/assets/dbeaec9c-ad91-46cb-8deb-8934279c3c5a" />

<br>

```bash
sudo smbclient //localhost/netlogon -U 'administrator'
```

<img width="737" height="71" alt="image" src="https://github.com/user-attachments/assets/7ab6af6d-e4e2-4970-8942-4531356fc927" />

<br>

```bash
sudo samba-tool user setpassword administrator
testparm
sudo samba-tool domain level show
```

<img width="640" height="85" alt="image" src="https://github.com/user-attachments/assets/1f37899d-e578-47e9-881c-0a5a9c577d2b" />

<br>

<img width="507" height="787" alt="image" src="https://github.com/user-attachments/assets/f9cf7707-31fb-4bf7-8c74-6ea10b76ca7d" />

<br>

<img width="556" height="128" alt="image" src="https://github.com/user-attachments/assets/e6e0b557-bfd9-4bda-a92b-14b4fa6df8b5" />

<br>

---

## Step 8 — Establish the Domain Trust

With both DCs operational, we can now create the bidirectional forest trust. This trust allows users from `lab120.lan` to access resources in `lab12.lan` and vice versa.

<br>

**Run the trust creation command from the LS12 domain controller:**

```bash
sudo samba-tool domain trust create lab120.lan \
  --type=forest \
  --direction=both \
  -U administrator@lab120.lan
```

> - `--type=forest` — Creates a forest-level trust (the highest level, for full cross-domain access)
> - `--direction=both` — Makes the trust bidirectional (both domains trust each other)
> - `-U administrator@lab120.lan` — Authenticates as the remote domain's administrator

<br>

<img width="1242" height="490" alt="image" src="https://github.com/user-attachments/assets/0f5c35d3-2fee-49f3-8247-526ef7358161" />

<br>

**List all configured trusts:**

```bash
sudo samba-tool domain trust list
```

<img width="606" height="48" alt="image" src="https://github.com/user-attachments/assets/1bda3171-d848-4119-9ab9-48b153324984" />

<br>

**Display detailed trust information:**

```bash
sudo samba-tool domain trust show lab120.lan
```

<img width="935" height="323" alt="image" src="https://github.com/user-attachments/assets/8843d7d7-185d-42f3-b6db-477c3c94f0b3" />

<br>

**Validate the trust is working correctly:**

```bash
sudo samba-tool domain trust validate lab06.lan
```

> A successful validation confirms the trust is active and both DCs can communicate with each other.

<br>

<img width="942" height="142" alt="image" src="https://github.com/user-attachments/assets/98bab33f-f4fc-4316-970b-09e3d21b7737" />

<br>

---

## Step 9 — Create Test Users for LAB120

To test cross-domain authentication, create two test user accounts on the `lab120.lan` domain.

```bash
sudo samba-tool user create testuser admin_21 --given-name=Test --surname=User
sudo samba-tool user create testuser2 admin_21 --given-name=Test --surname=User
```

<img width="901" height="27" alt="image" src="https://github.com/user-attachments/assets/9cded784-0c95-415f-b00e-1e1954ae4d52" />

<br>

<img width="925" height="32" alt="image" src="https://github.com/user-attachments/assets/fb3bb7dc-f757-454b-8710-b628281453b5" />

<br>

**Verify the users were created:**

```bash
sudo samba-tool user list
```

<img width="427" height="123" alt="image" src="https://github.com/user-attachments/assets/dc929015-e441-4b7e-98d0-21296d75ced8" />

<br>

---

## Step 10 — Cross-Domain Authentication Verification

The final step confirms that a user from `lab120.lan` can authenticate from a machine joined to `lab12.lan`. This is the ultimate proof that the forest trust is working end-to-end.

<br>

**On the LS12 Domain Controller, request a Kerberos ticket for a LAB120 user:**

```bash
kinit testuser@LAB120.LAN
```

> If the trust is working, Kerberos will successfully grant a TGT for the foreign domain user.

<br>

<img width="608" height="57" alt="image" src="https://github.com/user-attachments/assets/4a316fce-5e16-4827-b079-e2ee65ca71d1" />

<br>

**Verify the Kerberos ticket was issued:**

```bash
klist
```

> The output should show a ticket for `testuser@LAB120.LAN` — proof that cross-domain Kerberos authentication is functioning.

<br>

<img width="555" height="110" alt="image" src="https://github.com/user-attachments/assets/9e8ffd67-0d5c-4a92-8b48-959fac61efbe" />

<br>

**Clean up — destroy all cached Kerberos tickets:**

```bash
kdestroy
```

<img width="223" height="31" alt="image" src="https://github.com/user-attachments/assets/97fecc7c-f200-47d0-a8b3-1c815c27da76" />

<br>

---

<br>

*📄 Linux-Sprint Technical Documentation — Samba Active Directory on Ubuntu Server*
