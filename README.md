# 🐧 Linux-Sprint

> **Documentación técnica** del proceso de configuración de un entorno Linux con Samba Active Directory, integración de clientes, gestión de almacenamiento y administración de dominios.

---

## 📋 Tabla de Contenidos

- [Sprint 1 — Promote Domain Controller](#sprint-1--promote-domain-controller)
- [Sprint 2 — Linux and Windows Client Integration](#sprint-2--linux-and-windows-client-integration)
- [Sprint 3 — Storage, Shares & Permissions](#sprint-3--storage-shares--permissions)
- [Sprint 4 — Domain Trust](#sprint-4--domain-trust)

---

# 🖥️ SPRINT 1 — Promote Domain Controller

## ⚙️ Configuración inicial del hostname

We change the hostname with this command:

```bash
sudo hostnamectl set-hostname ls12
```

<img width="474" height="71" alt="imagen" src="https://github.com/user-attachments/assets/6932197e-3e60-4bac-ad15-ba54634bc41a" />

---

## 📄 Edición del archivo /etc/hosts

We proceed to edit this file and add the actual FQDN to the domain controller:

```bash
sudo nano /etc/hosts
```

<img width="801" height="179" alt="imagen" src="https://github.com/user-attachments/assets/23de9c27-387b-4207-8132-0e11b7a36283" />

---

## 🌐 Configuración de red (Netplan)

**Edit the network file:**

```bash
nano /etc/netplan/00-installer-config.yaml
```

Add the following configuration:

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

<img width="978" height="396" alt="imagen" src="https://github.com/user-attachments/assets/a107fee4-0764-4043-90f0-0b62c1d6cabd" />

<img width="795" height="102" alt="imagen" src="https://github.com/user-attachments/assets/495f7dc0-e14f-49c1-9405-8be5337216b9" />

**Apply the changes:**

```bash
sudo netplan apply
```

<img width="631" height="131" alt="imagen" src="https://github.com/user-attachments/assets/e77da347-3d6d-4ac4-b232-ef60bb549406" />

<img width="798" height="341" alt="imagen" src="https://github.com/user-attachments/assets/09f7ee55-1976-4699-abee-e882b24fada9" />

<img width="623" height="53" alt="imagen" src="https://github.com/user-attachments/assets/683f6c83-ddb9-4143-b90e-cfaa1ea63084" />

---

## 🔗 Configuración de resolv.conf

**Remove symbolic link to the /etc/resolv.conf file:**

```bash
sudo unlink /etc/resolv.conf
```

<img width="382" height="20" alt="imagen" src="https://github.com/user-attachments/assets/596162ac-cb9f-49ff-94d7-e7bc392f9728" />

**Recreate the /etc/resolv.conf file:**

```bash
sudo nano /etc/resolv.conf
```

<img width="802" height="110" alt="imagen" src="https://github.com/user-attachments/assets/736ebab6-1fa6-430b-85e5-c3229df50664" />

**Make the file immutable so that it cannot be changed:**

```bash
sudo chattr +i /etc/resolv.conf
```

<img width="407" height="28" alt="imagen" src="https://github.com/user-attachments/assets/772b9b80-3d01-4288-8f97-1aaa1ab82880" />

<img width="674" height="234" alt="imagen" src="https://github.com/user-attachments/assets/d2ea2c58-ae52-4299-82a5-d5eac2c8902e" />

---

## 📦 Instalación de paquetes Samba y dependencias

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 \
  krb5-config krb5-user dnsutils chrony net-tools
```

<img width="797" height="40" alt="imagen" src="https://github.com/user-attachments/assets/74169420-b712-4ac3-87f7-0ebb8f2ca49d" />

<img width="766" height="222" alt="imagen" src="https://github.com/user-attachments/assets/814abeca-9769-4d4e-88b5-cb9654f418b9" />

<img width="719" height="177" alt="imagen" src="https://github.com/user-attachments/assets/10b47588-1973-4418-bb87-273d9ca4d8d5" />

<img width="721" height="180" alt="imagen" src="https://github.com/user-attachments/assets/f9cc1d19-d2c8-4180-92ca-478be067531e" />

<img width="801" height="183" alt="imagen" src="https://github.com/user-attachments/assets/a93d94f4-6e17-4ccc-8af1-0ba59aab0c09" />

<img width="802" height="126" alt="imagen" src="https://github.com/user-attachments/assets/303dc108-ff3a-42bf-a8e3-03280eb874c8" />

<img width="796" height="68" alt="imagen" src="https://github.com/user-attachments/assets/62c396aa-772e-42d2-b5ec-b89512604af5" />

<img width="578" height="39" alt="imagen" src="https://github.com/user-attachments/assets/faaa0ea9-a377-43bd-8c2c-7582223b0b7f" />

<img width="635" height="116" alt="imagen" src="https://github.com/user-attachments/assets/fc49ebbd-a635-4b78-bd18-c135e22feee5" />

<img width="515" height="28" alt="imagen" src="https://github.com/user-attachments/assets/7419dc0d-2085-4af4-ad25-f76d9b3a06cd" />

<img width="590" height="22" alt="imagen" src="https://github.com/user-attachments/assets/8e86ed15-e96c-49a4-9be7-f273cfba777e" />

**Initialize the service and check its status to ensure it is functioning correctly:**

```bash
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```

<img width="425" height="43" alt="imagen" src="https://github.com/user-attachments/assets/bf81ba96-adbf-41c5-98e7-aae772bca9ca" />

<img width="805" height="578" alt="imagen" src="https://github.com/user-attachments/assets/cf3a6fef-17a1-4848-b401-f5a1131e90be" />

---

## 🕐 Configuración de sincronización horaria (Chrony)

We changed the default permission and ownership of the `/var/lib/samba/ntp_signd/ntp_signed` directory for time synchronization:

<img width="551" height="36" alt="imagen" src="https://github.com/user-attachments/assets/679df16d-dcf3-4dc0-a57d-f90c370f1872" />

```bash
nano /etc/chrony/chrony.conf
```

We add the following lines with the DNS IP and allow the subnet of our network:

<img width="802" height="547" alt="imagen" src="https://github.com/user-attachments/assets/c45bda87-c6a5-475a-b388-c3a6e9b1752a" />

**Initialize the service and check its status:**

```bash
sudo systemctl restart chronyd
sudo systemctl status chronyd
```

<img width="403" height="44" alt="imagen" src="https://github.com/user-attachments/assets/b5083434-c798-4624-af01-14ffc8df1922" />

<img width="802" height="352" alt="imagen" src="https://github.com/user-attachments/assets/9ae21ca1-d1d7-4ce7-b330-76e6b7dfc0b1" />

---

## ✅ Verificación del dominio

**Verify domain names:**

```bash
host -t A lab12.lan
```

<img width="310" height="54" alt="imagen" src="https://github.com/user-attachments/assets/247487de-c970-45fb-9eba-92fb4864e103" />

```bash
host -t A ls12.lab12.lan
```

<img width="346" height="57" alt="imagen" src="https://github.com/user-attachments/assets/ea8f1764-f027-491b-8985-17e6bb6a413d" />

**Verify that the Kerberos and LDAP service records point to the FQDN of our Samba Active Directory server:**

```bash
host -t SRV _kerberos._udp.lab12.lan
```

<img width="515" height="38" alt="imagen" src="https://github.com/user-attachments/assets/302eea64-0d36-40fb-b392-70df9eec6fde" />

```bash
host -t SRV _ldap._tcp.clockwork.local
```

<img width="491" height="41" alt="imagen" src="https://github.com/user-attachments/assets/2d2826f8-0a65-426a-8e34-a5bebe87294e" />

**Verify available resources in Active Directory Samba:**

```bash
smbclient -L clockwork.local -N
```

<img width="556" height="161" alt="imagen" src="https://github.com/user-attachments/assets/1d5e33aa-b4e7-43ef-9471-98e892ab747c" />

**Verify authentication on the Kerberos server using the user manager:**

```bash
kinit administrator@LAB12.LAN
```

<img width="603" height="58" alt="imagen" src="https://github.com/user-attachments/assets/65c2f14b-cfbd-412e-a0db-99299a01f084" />

<img width="541" height="121" alt="imagen" src="https://github.com/user-attachments/assets/a5c53eb9-445a-4918-b86b-3d110ae71864" />

**Log in to the server via SMB for checks:**

```bash
sudo smbclient //localhost/netlogon -U 'administrator'
```

<img width="591" height="78" alt="imagen" src="https://github.com/user-attachments/assets/e43de933-1169-4b0c-8d7e-92bc33ab8730" />

**Change administrator user password:**

```bash
sudo samba-tool user setpassword administrator
```

<img width="710" height="40" alt="imagen" src="https://github.com/user-attachments/assets/b3070cd2-72e4-494f-bd14-5ac07cbbdc5a" />

**Verify the integrity of the Samba configuration file:**

```bash
testparm
```

<img width="450" height="666" alt="imagen" src="https://github.com/user-attachments/assets/b18f299c-b956-4958-a56d-4489de8dbbb8" />

**Verify the functionality of WINDOWS AD DC 2008:**

```bash
sudo samba-tool domain level show
```

<img width="500" height="107" alt="imagen" src="https://github.com/user-attachments/assets/6b95ef8c-59e5-4d96-870c-7acd9c31af06" />

---

# 💻 SPRINT 2 — Linux and Windows Client Integration

## 🔄 Actualización del sistema

```bash
sudo apt update
```

<img width="718" height="150" alt="imagen" src="https://github.com/user-attachments/assets/6f4f6456-baf5-4cd4-997c-9f9a413a53fa" />

```bash
sudo apt upgrade
```

<img width="1190" height="362" alt="imagen" src="https://github.com/user-attachments/assets/8a12c418-ac51-4d48-aa86-faed7b35be2c" />

---

## 🔐 Instalación y verificación de SSH

```bash
sudo apt-get install ssh
```

<img width="708" height="185" alt="imagen" src="https://github.com/user-attachments/assets/8745d1ea-bfbc-4469-8254-554181492278" />

**Check SSH state:**

```bash
sudo systemctl status ssh
```

<img width="717" height="303" alt="imagen" src="https://github.com/user-attachments/assets/c11c775c-0c94-4d51-851d-04b056c48ee0" />

---

## ⚙️ Configuración del cliente Linux

**Change hostname:**

```bash
sudo hostnamectl set-hostname lc12
hostname -f
```

<img width="460" height="72" alt="imagen" src="https://github.com/user-attachments/assets/25069b5f-c4d4-47de-9ac9-8b4045a32060" />

**Configure file /etc/hosts:**

```bash
nano /etc/hosts
```

<img width="347" height="23" alt="imagen" src="https://github.com/user-attachments/assets/3ee4ed8b-172b-4ece-8d48-652026784e09" />

<img width="716" height="243" alt="imagen" src="https://github.com/user-attachments/assets/c05ac32f-439f-407f-9eb9-3d8ac8382dce" />

**Check connectivity to the domain using the ping command:**

```bash
ping lab12.lan
```

<img width="663" height="152" alt="image" src="https://github.com/user-attachments/assets/845a1d93-cd53-4fb8-860e-e07a93489425" />

---

## 🕐 Sincronización horaria con ntpdate

```bash
sudo apt-get install ntpdate
```

<img width="713" height="427" alt="imagen" src="https://github.com/user-attachments/assets/7dc08504-a255-4d04-8e5c-2c6236345ab2" />

```bash
sudo ntpdate -q lab12.lan
sudo ntpdate lab12.lan
```

<img width="767" height="94" alt="imagen" src="https://github.com/user-attachments/assets/7d76e4a1-b8c6-4ea3-b908-e80dd6eaa96f" />

---

## 📦 Instalación de paquetes necesarios

```bash
sudo apt-get install samba krb5-config krb5-user winbind libpam-winbind libnss-winbind
```

<img width="882" height="23" alt="imagen" src="https://github.com/user-attachments/assets/3c583e93-afa7-4d85-b9e2-b92c7ff5609b" />

<img width="768" height="199" alt="imagen" src="https://github.com/user-attachments/assets/b702a6f7-29b0-49a7-8575-866cc959bcf5" />

<img width="617" height="196" alt="imagen" src="https://github.com/user-attachments/assets/c52ba31b-8a79-4cc6-a450-1f96b6dba191" />

<img width="616" height="184" alt="imagen" src="https://github.com/user-attachments/assets/7c6d9a3c-66ec-4238-b29b-788c83a1c6fe" />

---

## 🔑 Autenticación Kerberos

**Verify authentication on the Kerberos server using the user manager:**

```bash
kinit administrator@LAB12.LAN
```

<img width="594" height="57" alt="imagen" src="https://github.com/user-attachments/assets/bbe0e8d6-a145-4305-9334-7fcaffa4f749" />

```bash
klist
```

<img width="520" height="119" alt="imagen" src="https://github.com/user-attachments/assets/0405eebe-9aeb-4fe1-9341-9ea3aec70157" />

---

## 📝 Configuración del archivo smb.conf

**Move the smb.conf file and create a backup:**

```bash
mv /etc/samba/smb.conf /etc/samba/smb.conf.initial
```

<img width="491" height="20" alt="imagen" src="https://github.com/user-attachments/assets/1e1fde2e-991e-472c-b80e-cea68e656273" />

**Create an empty smb.conf file:**

```bash
nano /etc/samba/smb.conf
```

<img width="422" height="17" alt="imagen" src="https://github.com/user-attachments/assets/6f795203-931c-4f2b-ac78-7b1a0f005ff3" />

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

<img width="732" height="386" alt="imagen" src="https://github.com/user-attachments/assets/a1ad0024-b608-4669-b422-be1df542a250" />

---

## 🔄 Gestión de servicios Samba

**Restart all Samba daemons:**

```bash
sudo systemctl restart smbd nmbd
```

<img width="448" height="22" alt="imagen" src="https://github.com/user-attachments/assets/f31b2edb-98b2-4852-8404-7b7d35c33d5e" />

**Stop unnecessary services:**

```bash
sudo systemctl stop samba-ad-dc
```

<img width="443" height="23" alt="imagen" src="https://github.com/user-attachments/assets/f3d6f5b9-de2e-4c7e-a46e-1dccca5e6ce2" />

**Enable samba services:**

```bash
sudo systemctl enable smbd nmbd
```

<img width="789" height="115" alt="imagen" src="https://github.com/user-attachments/assets/40e8baea-f2ca-4ccf-8d62-90a8cb3c05d8" />

---

## 🔗 Unión al dominio Samba AD DC

**Join Ubuntu Desktop to SAMBA AD DC:**

```bash
sudo net ads join -U administrator
```

<img width="508" height="26" alt="imagen" src="https://github.com/user-attachments/assets/3b2bb6cc-2744-430e-a2d7-ccd375f9a1bc" />

**List SAMBA AD teams:**

```bash
sudo samba-tool computer list
```

<img width="378" height="69" alt="imagen" src="https://github.com/user-attachments/assets/a01d1e3c-f5d4-4ff6-86ee-fa2bb2c84eaa" />

**Verify from the client's end whether the connection is valid:**

```bash
net ads testjoin
```

<img width="366" height="44" alt="imagen" src="https://github.com/user-attachments/assets/a041644d-4ec4-4a4c-8994-9337983a8d8c" />

---

## 🛠️ Configuración de autenticación AD

**Edit the Name Service Switch (NSS) configuration file:**

```bash
sudo nano /etc/nsswitch.conf
```

<img width="425" height="28" alt="imagen" src="https://github.com/user-attachments/assets/b54fbe04-95e4-4777-ae2f-ee24d24b1a54" />

<img width="793" height="433" alt="imagen" src="https://github.com/user-attachments/assets/66a97ad6-0543-4690-a904-fe784cc8aa82" />

**Restart winbind service:**

```bash
sudo systemctl restart winbind
```

<img width="433" height="19" alt="imagen" src="https://github.com/user-attachments/assets/6bd46a2c-e4e6-4c6f-9b01-5192b10c9838" />

**Check if Ubuntu Desktop has been integrated into the domain:**

```bash
wbinfo -u
wbinfo -g
```

<img width="322" height="364" alt="imagen" src="https://github.com/user-attachments/assets/c7bcc3ed-fb1e-4cea-932d-955fc72ce5eb" />

**Verify the Winbind nsswitch module with the getent command:**

```bash
sudo getent passwd | grep administrator
sudo getent group | grep 'domain admins'
id administrator
```

<img width="517" height="33" alt="imagen" src="https://github.com/user-attachments/assets/95b19adc-a219-4149-b6e1-7b84199324b3" />

<img width="493" height="36" alt="imagen" src="https://github.com/user-attachments/assets/29377847-b312-491b-b851-243d74657580" />

<img width="795" height="75" alt="imagen" src="https://github.com/user-attachments/assets/e54074c0-8cd9-4c56-baf7-712fb0b5cf42" />

---

## 👥 Configuración de PAM

**Configure pam-auth-update to authenticate with domain accounts and to automatically create directories:**

```bash
sudo pam-auth-update
```

<img width="350" height="21" alt="imagen" src="https://github.com/user-attachments/assets/af2a2bf9-b546-4041-9e0f-0dd874cf5000" />

<img width="757" height="445" alt="imagen" src="https://github.com/user-attachments/assets/9550ef33-07bc-4535-a19a-4452f580b0a6" />

**Edit the /etc/pam.d/common-account file to automatically create directories:**

```bash
sudo nano /etc/pam.d/common-account
```

<img width="487" height="24" alt="imagen" src="https://github.com/user-attachments/assets/d62dd0bb-be12-4df5-a74a-24218b4482de" />

**Add to the end of the file:**

```
session required pam_mkhomedir.so skel=/etc/skel/ umask=0022
```

<img width="786" height="532" alt="imagen" src="https://github.com/user-attachments/assets/cc293e06-9297-4a46-b06e-c7eb03e7379c" />

**Log in with Samba4 AD account:**

```bash
su administrator
```

<img width="314" height="56" alt="imagen" src="https://github.com/user-attachments/assets/09dc6311-0fe8-4b0a-9486-51756e6f3be9" />

**Authenticate with GUI:**

<img width="376" height="304" alt="imagen" src="https://github.com/user-attachments/assets/f93ecbd0-f72b-45df-8b78-7bd7f78fc32b" />

<img width="368" height="228" alt="imagen" src="https://github.com/user-attachments/assets/2201485a-1c2e-4724-939d-57872451b735" />

<img width="489" height="74" alt="imagen" src="https://github.com/user-attachments/assets/1405ea13-b647-4546-9346-570b5dd731b6" />

<img width="613" height="483" alt="imagen" src="https://github.com/user-attachments/assets/8be61fb0-a733-41c2-a4b2-7a765df495f0" />

<img width="609" height="481" alt="imagen" src="https://github.com/user-attachments/assets/5ea4733b-6137-4e58-b857-f14c1d1ab896" />

<img width="612" height="481" alt="imagen" src="https://github.com/user-attachments/assets/70fd1e12-8126-4fb4-834f-8b3f26bb1c26" />

<img width="611" height="477" alt="imagen" src="https://github.com/user-attachments/assets/a82d7d9a-8a55-4cd6-b5c3-d98ce2220a9b" />

<img width="407" height="482" alt="imagen" src="https://github.com/user-attachments/assets/d8c410d4-c0b4-4dfb-9b68-5cd976326a1d" />

---

## 👤 Creación de usuarios y grupos

**Create Users (Alice, Bob, Charlie):**

```bash
sudo samba-tool user create Alice --userou="OU=IT_Department"
```

<img width="663" height="77" alt="imagen" src="https://github.com/user-attachments/assets/23515350-6a24-4771-aafb-fa9e324e0f74" />

```bash
sudo samba-tool user create Bob --userou="OU=Students"
```

<img width="601" height="70" alt="imagen" src="https://github.com/user-attachments/assets/44ad168f-9be1-4157-9505-365f9fc04114" />

```bash
sudo samba-tool user create Charlie --userou="OU=Students"
```

<img width="626" height="68" alt="imagen" src="https://github.com/user-attachments/assets/5946eb66-0bab-4c26-b946-3d219d400e91" />

**Create Groups (IT_Admins, Students):**

```bash
sudo samba-tool group add IT_Admins
```

<img width="435" height="25" alt="image" src="https://github.com/user-attachments/assets/551a9e87-754e-4e29-839c-983f36c265ad" />

```bash
sudo samba-tool group add Students
```

<img width="427" height="23" alt="image" src="https://github.com/user-attachments/assets/8cdadbf0-1d3b-4066-b87d-35a172025cc7" />

**Add Users to Groups:**

```bash
sudo samba-tool group addmembers IT_Admins Alice
sudo samba-tool group addmembers Students Bob,Charlie
```

<img width="542" height="27" alt="image" src="https://github.com/user-attachments/assets/982c6b7c-ad7d-4d5d-a216-e356fe25215e" />

<img width="585" height="22" alt="image" src="https://github.com/user-attachments/assets/70b59cc6-e834-4119-9cc0-ca415971a4c1" />

**Create the OU hierarchy:**

```bash
sudo samba-tool ou add "OU=IT_Department,DC=lab12,DC=lan"
sudo samba-tool ou add "OU=Students,DC=lab12,DC=lan"
sudo samba-tool ou add "OU=HR_Department,DC=lab12,DC=lan"
```

<img width="628" height="36" alt="image" src="https://github.com/user-attachments/assets/6fe6093e-7216-42ad-8c34-1d458ac5f202" />

<img width="593" height="32" alt="image" src="https://github.com/user-attachments/assets/c8d3c92f-49c6-4597-9eaf-11d89a28683c" />

<img width="632" height="35" alt="image" src="https://github.com/user-attachments/assets/66847513-2859-458c-96dd-81113129f2d1" />

---

## 🔒 Introducción a GPOs (Group Policy Objects)

### 1. Política de contraseñas

According to the security requirements, all users must have a minimum password length of 8 characters:

```bash
sudo samba-tool domain passwordsettings set --min-pwd-length=8 --complexity=on
```

<img width="784" height="130" alt="imagen" src="https://github.com/user-attachments/assets/4d484726-1ff2-42cd-a799-8e92e9b7e8e8" />

### 2. Política de bloqueo de cuentas

To mitigate brute-force attacks, we implement a lockout policy: **3 failed attempts lead to a 5-minute lockout**.

**1. Set the threshold (3 attempts):**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-threshold=3
```

<img width="742" height="55" alt="imagen" src="https://github.com/user-attachments/assets/1585bde3-d978-49ba-ace5-3532cb3874c3" />

**2. Set the lockout duration (5 minutes):**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-duration=5
```

<img width="731" height="53" alt="imagen" src="https://github.com/user-attachments/assets/f3d5a0e6-1a31-4a10-b6f2-5c880cb3b099" />

**3. Set the reset counter timer (5 minutes):**

```bash
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=5
```

<img width="731" height="51" alt="imagen" src="https://github.com/user-attachments/assets/7880d848-5a6c-4534-a8c2-bc1a031c7efb" />

### 3. Verificación y pruebas

**Account Lockout Checking:**

To verify the Account Lockout Policy, a brute-force simulation was performed:

1. Attempted to log in with user **Bob** using a wrong password.
2. Repeated this **3 times** to trigger the threshold.
3. On the 4th attempt, the system displayed the lockout message:

<img width="796" height="719" alt="imagen" src="https://github.com/user-attachments/assets/82c44e16-1e08-4aea-97e2-f7cdc99302b3" />

4. Waited 5 minutes (as configured in the lockout duration) and verified that the account was automatically unlocked.

**Verifying the Organizational Structure:**

```bash
sudo samba-tool ou list
```

<img width="358" height="104" alt="imagen" src="https://github.com/user-attachments/assets/6ffa1252-2d2b-4f6d-9c2f-05d5d44005c0" />

**Auditing User Placement and Distinguished Names:**

```bash
sudo samba-tool user show Alice | grep dn
```

<img width="507" height="106" alt="imagen" src="https://github.com/user-attachments/assets/e89984d2-9745-4b93-9303-36af4480e04e" />

**Group Membership Validation:**

```bash
sudo samba-tool group listmembers IT_Admins
sudo samba-tool group listmembers Students
```

<img width="591" height="86" alt="imagen" src="https://github.com/user-attachments/assets/ae88b763-ca0b-4359-b2f5-72bc19de5bca" />

**Security Policy Audit:**

```bash
sudo samba-tool domain passwordsettings show
```

<img width="513" height="205" alt="imagen" src="https://github.com/user-attachments/assets/49879fd8-6c84-42f0-87e1-858cfdb4bffc" />

---

# 💾 SPRINT 3 — Storage, Shares & Permissions

## 🔍 Verificación de conectividad

**Check the domain status:**

```bash
sudo samba-tool domain info 127.0.0.1
```

<img width="458" height="148" alt="image" src="https://github.com/user-attachments/assets/ae1e256c-0dde-47e5-83f4-61b10f8e999d" />

**Ping the domain to verify DNS resolution:**

```bash
ping -c 3 lab12.lan
```

<img width="597" height="152" alt="image" src="https://github.com/user-attachments/assets/6c2d4c83-c52f-43b6-b225-90d6383f42aa" />

---

## 1. 🗄️ Gestión de almacenamiento

### Paso 1: Crear el nuevo disco

We need to create the new storage device (usually 20GB or 40GB):

<img width="1025" height="595" alt="image" src="https://github.com/user-attachments/assets/361a78ba-7147-4173-ab4d-44d88ed749b7" />

### Paso 2: Identificar el nuevo disco

```bash
lsblk
```

<img width="582" height="230" alt="image" src="https://github.com/user-attachments/assets/c4e68074-9f56-497b-b1a0-f4975e2638fb" />

### Paso 3: Crear la tabla de particiones

We will use `fdisk` to initialize the disk and create a primary partition:

```bash
sudo fdisk /dev/sdb
```

**fdisk command sequence:**

| Tecla | Acción |
|-------|--------|
| `n` | Create a new partition |
| `p` | Define it as primary |
| `1` | Assign the partition number |
| `Enter` (x2) | Accept default start and end sectors |
| `w` | Write the changes to the disk |

<img width="677" height="397" alt="image" src="https://github.com/user-attachments/assets/78e3d80b-87b1-4671-bbd2-7cd360d7f362" />

### Paso 4: Formatear el sistema de archivos

For Samba to correctly manage Active Directory permissions, the disk must be formatted in **EXT4**:

```bash
sudo mkfs.ext4 /dev/sdb1
```

<img width="520" height="182" alt="image" src="https://github.com/user-attachments/assets/aaa52e39-9562-45f1-9d2f-fd68049fb510" />

### Paso 5: Punto de montaje permanente

**1. Create mountpoint:**

```bash
sudo mkdir -p /srv/samba/Data
```

<img width="391" height="27" alt="image" src="https://github.com/user-attachments/assets/631bf1b6-dcb7-47c5-8f64-019291f3122c" />

**2. Configure automatic mounting (/etc/fstab):**

Get the disk's UUID so that the mount persists after rebooting:

```bash
sudo blkid /dev/sdb1
```

<img width="802" height="55" alt="image" src="https://github.com/user-attachments/assets/43f930b2-e609-4df1-904d-330d894943a0" />

Copy the UUID and add it to the configuration file:

```bash
sudo nano /etc/fstab
```

Add this line to the end:

```
UUID=your-uuid-here /srv/samba/Data ext4 user_xattr,acl,barrier=1 1 1
```

<img width="931" height="283" alt="image" src="https://github.com/user-attachments/assets/621a52c0-f4cd-41e6-b168-72ab2261e54c" />

**3. Mount the disc:**

```bash
sudo mount -a
df -h /srv/samba/Data
```

<img width="490" height="62" alt="image" src="https://github.com/user-attachments/assets/4b3b45c1-ff0c-4920-88fd-f18b69079795" />

### Verificaciones de almacenamiento

**1. Verificación de dispositivos y particiones:**

```bash
lsblk
```

<img width="620" height="298" alt="image" src="https://github.com/user-attachments/assets/17e3fa2e-3e02-481e-8e41-f37aa518d58e" />

**2. Mounting Point Status and Space:**

```bash
df -h /srv/samba/Data
```

<img width="510" height="66" alt="image" src="https://github.com/user-attachments/assets/d3e1f9a9-211c-4e5a-a253-958bdb5def92" />

**3. Persistence Check (fstab):**

```bash
cat /etc/fstab | grep Data
```

<img width="851" height="40" alt="image" src="https://github.com/user-attachments/assets/3c3c326b-c035-432a-8f5b-ea7394c57ef4" />

**4. Validation of Extended Attributes and ACL:**

```bash
findmnt -n -o OPTIONS /srv/samba/Data
```

<img width="517" height="47" alt="image" src="https://github.com/user-attachments/assets/ef509aaa-9e02-4ea4-93d9-5f1e422a4a53" />

---

## 2. 📁 Estructura de directorios y recursos compartidos

### Paso 1: Creación de la jerarquía de directorios

**1. Create the root folders for each department:**

```bash
sudo mkdir -p /srv/samba/Data/FinanceDocs
sudo mkdir -p /srv/samba/Data/HRDocs
sudo mkdir -p /srv/samba/Data/Public
```

<img width="555" height="86" alt="image" src="https://github.com/user-attachments/assets/aadd8f37-c0b1-4d56-96fa-7dfe4c1051aa" />

**2. Verify the structure:**

```bash
ls -l /srv/samba/Data
```

<img width="482" height="125" alt="image" src="https://github.com/user-attachments/assets/9379cb03-c380-4bd4-988b-35ac13e3f3d1" />

### Paso 2: Configuración de recursos compartidos Samba

**1. Open the Samba configuration file:**

```bash
sudo nano /etc/samba/smb.conf
```

**2. Add the Share Definitions:**

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

<img width="562" height="862" alt="image" src="https://github.com/user-attachments/assets/ba062069-d25b-43fa-963d-30b9f811896f" />

**3. Validate and Restart:**

```bash
testparm
```

<img width="467" height="132" alt="image" src="https://github.com/user-attachments/assets/13e1c51c-e60d-4585-bc3d-dcd2fb24af43" />

```bash
sudo systemctl restart samba-ad-dc
```

<img width="481" height="22" alt="image" src="https://github.com/user-attachments/assets/9356308d-3935-4b3f-8143-f7e59e4e8d83" />

### Paso 3: Verificaciones

**Local Share Listing:**

```bash
smbclient -L localhost -N
```

<img width="627" height="247" alt="image" src="https://github.com/user-attachments/assets/3e607ae5-571a-4f67-ac73-04cec65797ed" />

**Directory Ownership Check:**

```bash
ls -ld /srv/samba/Data/*
```

<img width="632" height="103" alt="image" src="https://github.com/user-attachments/assets/1d984dde-e425-4f85-9b01-3aad1b874891" />

---

## 3. 🔐 Permisos avanzados y privilegios de usuario

### Paso 1: Instalación y habilitación de soporte ACL

```bash
sudo apt update && sudo apt install acl -y
```

---

## 4. ⚙️ Gestión de tareas y procesos en Linux

### 🔍 Monitoreo de procesos

**The `top` Utility**

The standard tool for monitoring processes. It provides a dynamic, real-time view of a running system.

- **PID (Process ID):** The unique numerical identifier for the process.
- **%CPU & %MEM:** Shows the percentage of resources the process is consuming.
- **Command:** The name of the executable or script.

```bash
top
```

<img width="185" height="20" alt="imagen" src="https://github.com/user-attachments/assets/14bed00a-c0bd-45e3-80cc-4b15ac4fb3b3" />

<img width="801" height="371" alt="imagen" src="https://github.com/user-attachments/assets/197661f5-aee2-4a27-9fe2-88422b196d36" />

---

**The `htop` Utility**

For a more intuitive, colorful, and Task Manager-like experience. It allows you to scroll vertically and horizontally to see all processes and their full command lines.

```bash
htop
```

<img width="634" height="134" alt="imagen" src="https://github.com/user-attachments/assets/0b79537a-2c61-432f-8539-d9dd3d074ab0" />

<img width="798" height="364" alt="imagen" src="https://github.com/user-attachments/assets/8ffee18a-3ef0-4983-9f06-b322e4dc6b58" />

**Key interactions in htop:**

| Tecla | Acción |
|-------|--------|
| `F2` | Customize columns and display colors |
| `F6` | Sort processes by CPU, Memory, or Priority |
| `F9` | Send signals (SIGTERM or SIGKILL) to a process |

---

**The `ps` (Process Status) Command**

Used to view currently running processes:

```bash
# View all processes
ps aux

# Filter for Samba processes
ps aux | grep samba
```

<img width="795" height="245" alt="imagen" src="https://github.com/user-attachments/assets/d9dad8a9-70d3-4342-ad55-df73a9328e02" />

---

### 🛠️ Comandos de gestión de servicios

```bash
# Check status (Running/Stopped)
sudo systemctl status samba-ad-dc
```

<img width="802" height="399" alt="imagen" src="https://github.com/user-attachments/assets/adb7f9ee-c7d6-47f2-9dff-fb81baa2f535" />

```bash
# Restart a hung service
sudo systemctl restart samba-ad-dc
```

<img width="426" height="20" alt="imagen" src="https://github.com/user-attachments/assets/d2921b5e-1fba-40a5-b692-1cf94f4d64a5" />

```bash
# Enable service to start on boot
sudo systemctl enable samba-ad-dc
```

<img width="800" height="69" alt="imagen" src="https://github.com/user-attachments/assets/fbbb5ded-cde1-4a74-8aad-7525276ac856" />

```bash
# List all active services
systemctl list-units --type=service --state=running
```

<img width="798" height="469" alt="imagen" src="https://github.com/user-attachments/assets/7302992d-1513-424d-8528-9a03744609bb" />

---

### 📡 Interacción con procesos mediante señales

```bash
# Graceful termination (SIGTERM)
kill <PID>

# Forced termination (SIGKILL)
kill -9 <PID>

# Kill all processes by name
sudo killall smbd
```

---

### 🚂 Ejercicio práctico

**Execute sl:**

<img width="797" height="433" alt="imagen" src="https://github.com/user-attachments/assets/81ee956f-97c3-4203-880c-674755a4aa2b" />

**Find the PID:**

```bash
pgrep -sl
```

<img width="268" height="126" alt="imagen" src="https://github.com/user-attachments/assets/acc2b3c5-6bd7-4472-8a9d-ede1c98a371a" />

**Stop the process:**

```bash
kill -19 <PID>
```

<img width="288" height="26" alt="imagen" src="https://github.com/user-attachments/assets/612c9297-1d6d-4246-817a-aa366baa9cef" />

<img width="657" height="291" alt="imagen" src="https://github.com/user-attachments/assets/1fb29639-cc40-4409-aaa2-a540d5a5485c" />

**Resume process:**

```bash
kill -18 <PID>
```

<img width="1563" height="473" alt="imagen" src="https://github.com/user-attachments/assets/80e19292-b10e-41c4-8ebf-1dba21e7f671" />

---

# 🤝 SPRINT 4 — Domain Trust

We install another Linux Server to make the domain trust:

<img width="801" height="512" alt="image" src="https://github.com/user-attachments/assets/e485bfc4-b333-436a-a402-f5399eea63a8" />

<img width="801" height="512" alt="image" src="https://github.com/user-attachments/assets/52e791f3-6942-4936-b1fb-8b16f111e492" />

<img width="798" height="320" alt="image" src="https://github.com/user-attachments/assets/440d3c14-49c0-4242-8bf5-fe312c9f06e1" />

<img width="799" height="595" alt="imagen" src="https://github.com/user-attachments/assets/d5b3436b-e980-4fff-9db1-c5cb8feb5bc4" />

---

## ⚙️ Configuración del nuevo servidor (LS120)

**Edit the network file:**

```bash
nano /etc/netplan/00-installer-config.yaml
```

<img width="616" height="168" alt="imagen" src="https://github.com/user-attachments/assets/f3debf6c-2daf-4552-848e-e246bb7df90d" />

**Apply changes:**

```bash
sudo netplan apply
```

<img width="799" height="231" alt="imagen" src="https://github.com/user-attachments/assets/d2d54bb5-0b57-452e-ae68-22265328c44d" />

<img width="800" height="339" alt="imagen" src="https://github.com/user-attachments/assets/8cb90cd3-b4d4-4c00-97ab-d1c1bb2ca4a7" />

**Remove symbolic link to the /etc/resolv.conf file:**

```bash
sudo unlink /etc/resolv.conf
```

<img width="415" height="18" alt="imagen" src="https://github.com/user-attachments/assets/6e52f4d5-7961-4d9a-847f-49aca9786fe1" />

<img width="796" height="598" alt="imagen" src="https://github.com/user-attachments/assets/c0077bf7-e2dc-4c54-9d8f-e212ce34dd27" />

<img width="437" height="21" alt="imagen" src="https://github.com/user-attachments/assets/1622c6e4-9825-4d69-a832-b88d187f2251" />

**Install Samba with its packages and dependencies:**

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 \
  krb5-config krb5-user dnsutils chrony net-tools
```

LAB120.LAN

<img width="907" height="285" alt="image" src="https://github.com/user-attachments/assets/36ce743a-b587-41fc-acf4-e92bc4e56a56" />

ls120.lab120.lan

<img width="907" height="218" alt="image" src="https://github.com/user-attachments/assets/91e70db1-92a4-4ed2-9f7c-d3cd18b7b24f" />

<img width="913" height="222" alt="image" src="https://github.com/user-attachments/assets/51399d25-3dd9-420e-9884-83beac40af2c" />

**Stop and disable the services that the Samba Active Directory server does not require:**

```bash
sudo systemctl disable --now smbd nmbd winbind
```

<img width="923" height="138" alt="image" src="https://github.com/user-attachments/assets/2d76573f-b3ca-4d3d-804a-e1c93f3f336e" />

**We only need samba-ad-dc to function as an Active Directory and domain controller:**

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
```

<img width="516" height="45" alt="image" src="https://github.com/user-attachments/assets/e8d4dfe9-3051-4d91-b1b5-12adfb666e77" />

<img width="937" height="125" alt="image" src="https://github.com/user-attachments/assets/6fa38068-74eb-4a50-878a-c4d01a6b407b" />

---

## 🔧 Configuración de Samba Active Directory (LS120)

**Create a backup of the /etc/samba/smb.conf file:**

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
```

<img width="678" height="21" alt="image" src="https://github.com/user-attachments/assets/490745f2-8e2f-4c00-8ff3-ba3fb8676700" />

**Run the samba-tool command to begin provisioning Samba Active Directory:**

```bash
sudo samba-tool domain provision
```

```
Realm: LAB120.LAN
Domain: LAB120
Server Role: dc
DNS backend: SAMBA_INTERNAL
DNS forwarder IP address: 
```

<img width="941" height="257" alt="image" src="https://github.com/user-attachments/assets/ab455bb6-4c07-404a-be91-d7d74d35a1f3" />

**Create a backup of the default Kerberos configuration:**

```bash
sudo mv /etc/krb5.conf /etc/krb5.conf.orig
```

<img width="597" height="20" alt="image" src="https://github.com/user-attachments/assets/7b9042f4-a44a-4e0b-94d9-67e4d5a1774e" />

**Replace with the file /var/lib/samba/private/krb5.conf:**

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

<img width="702" height="23" alt="image" src="https://github.com/user-attachments/assets/105b68aa-5f7b-4c10-a53d-fce162c01af3" />

**Start the Samba Active Directory service:**

```bash
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```

<img width="488" height="28" alt="image" src="https://github.com/user-attachments/assets/a70b9a6f-3402-4af8-9ec3-8d38c84b5c31" />

<img width="942" height="577" alt="image" src="https://github.com/user-attachments/assets/de81b232-1c96-4c49-938c-f9e7807fec44" />

---

## 🕐 Sincronización horaria (LS120)

Change the default permissions and ownership of the `/var/lib/samba/ntp_signd/` directory:

```bash
sudo chown root:_chrony /var/lib/samba/ntp_signd/
sudo chmod 750 /var/lib/samba/ntp_signd/
```

<img width="657" height="30" alt="image" src="https://github.com/user-attachments/assets/6f13422d-1863-4b56-a9d8-8229e61592f5" />

<img width="570" height="25" alt="image" src="https://github.com/user-attachments/assets/8c92ed38-2fec-4fbf-b56f-fc0617e40785" />

**Modify the /etc/chrony/chrony.conf file:**

```bash
sudo nano /etc/chrony/chrony.conf
```

<img width="513" height="22" alt="image" src="https://github.com/user-attachments/assets/40ab6841-c6cd-4f9a-a5f4-91ef064a2a63" />

Add the following lines:

```
bindcmdaddress 192.168.30.42
allow 192.168.30.0/24
ntpsigndsocket /var/lib/samba/ntp_signd
```

<img width="941" height="933" alt="image" src="https://github.com/user-attachments/assets/e996678b-853e-47f6-9871-6bcdd8d6384f" />

**Restart and verify the chronyd service:**

```bash
sudo systemctl restart chronyd
sudo systemctl status chronyd
```

<img width="472" height="23" alt="image" src="https://github.com/user-attachments/assets/fa7f0092-fb8a-428a-9453-ff4a6a8ce5e2" />

<img width="937" height="512" alt="image" src="https://github.com/user-attachments/assets/079a9d29-1568-448d-943b-fc6481f47d0b" />

---

## ✅ Verificación de Samba Active Directory (LS120)

**Verify domain names:**

```bash
host -t A lab120.lan
host -t A ls120.lab120.lan
```

<img width="393" height="71" alt="image" src="https://github.com/user-attachments/assets/0dfaf3ae-23aa-4d4c-9c98-1d069682c802" />

<img width="452" height="67" alt="image" src="https://github.com/user-attachments/assets/6ca673af-87bf-4442-8f39-328e771a457b" />

**Verify Kerberos and LDAP service records:**

```bash
host -t SRV _kerberos._udp.lab120.lan
host -t SRV _ldap._tcp.lab120.lan
```

<img width="613" height="42" alt="image" src="https://github.com/user-attachments/assets/78cb578b-0773-4205-b170-34928b50ba79" />

<img width="582" height="51" alt="image" src="https://github.com/user-attachments/assets/fce9dde7-6dff-4d0b-ad72-8cef956b7fea" />

**Verify the default resources available in Samba Active Directory:**

```bash
smbclient -L clockwork.local -N
```

<img width="617" height="191" alt="image" src="https://github.com/user-attachments/assets/5cc241c6-10da-4984-a95a-362c73358c51" />

**Check authentication on the Kerberos server:**

```bash
kinit administrator@CLOCKWORK.LOCAL
klist
```

<img width="617" height="191" alt="image" src="https://github.com/user-attachments/assets/3219284b-18bc-41f7-9c07-b8f588f77723" />

<img width="606" height="142" alt="image" src="https://github.com/user-attachments/assets/dbeaec9c-ad91-46cb-8deb-8934279c3c5a" />

**Log in to the server via SMB:**

```bash
sudo smbclient //localhost/netlogon -U 'administrator'
```

<img width="737" height="71" alt="image" src="https://github.com/user-attachments/assets/7ab6af6d-e4e2-4970-8942-4531356fc927" />

**Change the administrator user password:**

```bash
sudo samba-tool user setpassword administrator
```

<img width="640" height="85" alt="image" src="https://github.com/user-attachments/assets/1f37899d-e578-47e9-881c-0a5a9c577d2b" />

**Verify the integrity of the Samba configuration file:**

```bash
testparm
```

<img width="507" height="787" alt="image" src="https://github.com/user-attachments/assets/f9cf7707-31fb-4bf7-8c74-6ea10b76ca7d" />

**Verify Windows Active Directory Domain Controller (DADC) functionality:**

```bash
sudo samba-tool domain level show
```

<img width="556" height="128" alt="image" src="https://github.com/user-attachments/assets/e6e0b557-bfd9-4bda-a92b-14b4fa6df8b5" />

---

## 🔗 Proceso de creación del Trust Domain

**Build trust using this command:**

```bash
sudo samba-tool domain trust create lab120.lan \
  --type=forest \
  --direction=both \
  -U administrator@lab120.lan
```

<img width="1242" height="490" alt="image" src="https://github.com/user-attachments/assets/0f5c35d3-2fee-49f3-8247-526ef7358161" />

**Verify trust:**

```bash
sudo samba-tool domain trust list
```

<img width="606" height="48" alt="image" src="https://github.com/user-attachments/assets/1bda3171-d848-4119-9ab9-48b153324984" />

**Display the trust details:**

```bash
sudo samba-tool domain trust show lab120.lan
```

<img width="935" height="323" alt="image" src="https://github.com/user-attachments/assets/8843d7d7-185d-42f3-b6db-477c3c94f0b3" />

**Validate trust:**

```bash
sudo samba-tool domain trust validate lab06.lan
```

<img width="942" height="142" alt="image" src="https://github.com/user-attachments/assets/98bab33f-f4fc-4316-970b-09e3d21b7737" />

---

## 👤 Creación de usuarios de prueba para LAB120

### LS120

**Create test users:**

```bash
sudo samba-tool user create testuser admin_21 --given-name=Test --surname=User
sudo samba-tool user create testuser2 admin_21 --given-name=Test --surname=User
```

<img width="901" height="27" alt="image" src="https://github.com/user-attachments/assets/9cded784-0c95-415f-b00e-1e1954ae4d52" />

<img width="925" height="32" alt="image" src="https://github.com/user-attachments/assets/fb3bb7dc-f757-454b-8710-b628281453b5" />

**Verify:**

```bash
sudo samba-tool user list
```

<img width="427" height="123" alt="image" src="https://github.com/user-attachments/assets/dc929015-e441-4b7e-98d0-21296d75ced8" />

---

## 🔄 Verificación de autenticación cruzada entre dominios

### LS12

**Authenticate with a LAB120 user:**

```bash
kinit testuser@LAB120.LAN
```

<img width="608" height="57" alt="image" src="https://github.com/user-attachments/assets/4a316fce-5e16-4827-b079-e2ee65ca71d1" />

**Verify ticket:**

```bash
klist
```

<img width="555" height="110" alt="image" src="https://github.com/user-attachments/assets/9e8ffd67-0d5c-4a92-8b48-959fac61efbe" />

**Clean up:**

```bash
kdestroy
```

<img width="223" height="31" alt="image" src="https://github.com/user-attachments/assets/97fecc7c-f200-47d0-a8b3-1c815c27da76" />

---

*Documentación generada para el proyecto Linux-Sprint* 🐧
