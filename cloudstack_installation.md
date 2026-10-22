# Apache CloudStack Installation Guide (KVM, Single-Host Lab)

This guide walks through installing Apache CloudStack 4.22 on a single Ubuntu VM that acts as the management server, the KVM compute host, and the NFS storage server at the same time.

## Table of Contents

1. [Lab Environment](#1-lab-environment)
2. [Management Server Credentials and SSH Access](#2-management-server-credentials-and-ssh-access)
3. [Configure CloudStack Management](#3-configure-cloudstack-management)
4. [Configure Nginx](#4-configure-nginx-to-bridge-connection-between-tester-and-compute-server)
5. [Configure the Local Compute Server](#5-configure-the-local-compute-server)
6. [Configure the Local Storage Server](#6-configure-the-local-storage-server)
7. [Download and Register an Image](#7-download-and-register-an-image)
8. [Appendix: Zone Configuration Values](#8-appendix-zone-configuration-values)

---

## 1. Lab Environment

Create a VM and set its assigned VMnet to **NAT mode**, so the VM lives behind the host device.

> **Why NAT and not bridged?** When the host device connects to a Wi-Fi network with a captive portal (login page), a VM in NAT mode does not need to log in again. Bridged mode treats the VM as a separate device with its own MAC address, which forces a new login on the captive portal. The same credential cannot be used by multiple sessions at the same time.

### How NAT mode works

VMnet in NAT mode puts two components in place:

- **NAT gateway**: acts as a virtual router that processes and translates packets from the outside network into the VM's internal network.
- **Virtual adapter**: an identification badge for the host device, so the VM can communicate with the host locally.

### Network addressing

| Item                    | Value              |
| ----------------------- | ------------------ |
| Subnet IP               | `192.168.126.0/24` |
| NAT gateway address     | `192.168.126.2`    |
| Virtual adapter address | `192.168.126.1`    |
| VM address              | `192.168.126.3`    |

---

## 2. Management Server Credentials and SSH Access

### VM credentials

| Item     | Value                  |
| -------- | ---------------------- |
| Hostname | `cloudstack-management` |
| Username | `management`           |
| Password | `g12026`               |

### 2.1 Install and enable OpenSSH

For convenience, install OpenSSH and enable it so it auto-starts on boot.

```bash
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

### 2.2 Generate an SSH key pair (on the host, in PowerShell)

To allow a secure SSH login as root, use a public SSH key instead of the root user password.

Template:

```powershell
ssh-keygen -t <type of algorithm> -f <path to store the key> -C "key label"
```

Example used in this lab:

```powershell
ssh-keygen -t ed25519 -f ~/.ssh/ed25519_adi -C "k1-cloud-adi"
```

Add a passphrase if wanted, then open the `.pub` file and copy the line:

```powershell
cat ~/.ssh/ed25519_adi.pub
```

| Item           | Value                                  |
| -------------- | -------------------------------------- |
| SSH key label  | `k1-cloud-adi`                         |
| Algorithm      | `ed25519`                              |
| Key passphrase | `g1adi`                                |
| Key file names | `ed25519_adi`, `ed25519_adi.pub`       |

### 2.3 Remove the old SSH fingerprint (only if needed)

If you previously had a VM with the same IP, remove the SSH fingerprint of the old VM first. Otherwise you cannot SSH to the new VM.

```powershell
ssh-keygen -R <ip address>
```

### 2.4 Install the public key on the VM

SSH into the VM with the regular user first, then paste the copied line into `/root/.ssh/authorized_keys`. Don't forget to restrict the permissions of the `.ssh` folder and the `authorized_keys` file, otherwise SSH will ignore them.

```bash
sudo mkdir -p /root/.ssh
sudo nano /root/.ssh/authorized_keys   # paste the public key line here
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
```

### 2.5 Register the VM in the VS Code SSH config file

```
Host cloudstack-management
  HostName 192.168.126.3
  User root
  IdentityFile ~/.ssh/ed25519_adi
```

---

## 3. Configure CloudStack Management

Two things must be installed:

- **cloudstack-management**: the main CloudStack program for the management server.
- **mysql-server**: the database that stores information regarding CloudStack.

### 3.1 Install MySQL and CloudStack Management

```bash
# add repository
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.22 /" > /etc/apt/sources.list.d/cloudstack.list

# installation
apt-get update -y
apt-get install cloudstack-management mysql-server
sudo systemctl enable --now mysql
```

### 3.2 Configure the behaviour of the MySQL database

The config file is located at `/etc/mysql/mysql.conf.d/mysqld.conf`.

```ini
[mysqld]
server_id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1  # roll back the entire transaction if a lock wait times out, instead of just the current statement
innodb_lock_wait_timeout=600  # a transaction waits up to 600 seconds for a row lock held by another transaction before giving up
max_connections=1000
log-bin=mysql-bin             # turn on binary logging
binlog-format = 'ROW'         # log the data that changed instead of the statement that changed it
```

Restart MySQL to apply the change:

```bash
sudo systemctl restart mysql
```

It is also recommended to set a password for the MySQL root user. The password `g12026` is used in this lab.

```bash
sudo mysql
```

Then, inside the MySQL prompt:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'g12026';
FLUSH PRIVILEGES;
exit
```

### 3.3 Deploy the database

Template:

```bash
sudo cloudstack-setup-databases <username>:<password>@<hostname> --deploy-as=<root_username>:<root_password> -i <management_server_ip>
```

- `<username>:<password>@<hostname>`: the credential of the MySQL user that CloudStack uses to talk to the database.
- `<root_username>:<root_password>`: the credential of the MySQL root user. The password can be left blank if MySQL and the management server are on the same device.

Final commands used in this lab:

```bash
sudo cloudstack-setup-databases cloud:g12026@localhost --deploy-as=root:g12026 -i 192.168.126.3
sudo cloudstack-setup-management
sudo systemctl enable --now cloudstack-management
```

---

## 4. Configure Nginx to Bridge Connection Between Tester and Compute Server

### 4.1 Install Nginx

```bash
sudo apt install nginx
```

### How Nginx configuration works

You create a configuration file stored inside `/etc/nginx/sites-available/`. To apply it, you add a shortcut (symlink) inside `/etc/nginx/sites-enabled/` so Nginx can load the file. There is already a config file named `default` with a symlink tied to it, which you need to delete:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

### 4.2 Create the configuration file, enable it, and restart Nginx

```bash
sudo nano /etc/nginx/sites-available/cloudstack-proxy
sudo ln -s /etc/nginx/sites-available/cloudstack-proxy /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

> **Note:** the contents of `cloudstack-proxy` were not included in the original notes. Add your `server { ... }` block (listen port and `proxy_pass` target) in the first command above before restarting Nginx.

---

## 5. Configure the Local Compute Server

Programs needed: `cloudstack-agent`, `qemu-kvm`, `libvirt-daemon-system`, `libvirt-clients`, `bridge-utils`, `cpu-checker`.

Almost all of them are installed automatically alongside `cloudstack-agent`. The only one you need to install manually is `bridge-utils`.

### 5.1 Install cloudstack-agent and other necessary programs

The CloudStack repository was already added in section 3.1, so no extra repository setup is needed on this VM.

```bash
sudo apt install cloudstack-agent bridge-utils
```

### 5.2 Create a bridge interface with Netplan

The network interface used earlier (`ens33`) is not a bridge. A guest VM needs a bridge so it can attach its address to that bridge and access the internet.

Edit the YAML file in `/etc/netplan/` to add a bridge interface called `cloudbr0` that uses `ens33` to reach the outside network:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      dhcp6: false
  bridges:
    cloudbr0:
      interfaces: [ens33]
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.126.3/24
      routes:
        - to: default
          via: 192.168.126.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
      parameters:
        stp: false
        forward-delay: 0
```

Apply the new Netplan config:

```bash
sudo netplan try
```

### 5.3 Configure libvirt

**a. Make the libvirt daemon listen on TCP port 16509** in `/etc/libvirt/libvirtd.conf`:

```conf
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
auth_tcp = "none"
mdns_adv = 0
```

**b. Add the `--listen` argument** in `/etc/default/libvirtd` so the daemon starts listening on the port assigned in the config file:

```
LIBVIRTD_ARGS="--listen"
```

**c. Bind QEMU VNC to all interfaces** instead of the `127.0.0.1` address in `/etc/libvirt/qemu.conf`:

```
vnc_listen = "0.0.0.0"
```

### 5.4 Disable AppArmor for libvirt

Disable AppArmor profiles so they won't block libvirt:

```bash
sudo ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
sudo ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
sudo apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

Then disable the systemd sockets so libvirtd can listen on the TCP port without conflicting with systemd, and restart the services:

```bash
sudo systemctl stop libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
sudo systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
sudo systemctl restart libvirtd
sudo systemctl restart cloudstack-agent
```

---

## 6. Configure the Local Storage Server

Programs needed: `nfs-kernel-server`, `nfs-common`, `chrony`. `nfs-common` is already included with `nfs-kernel-server`, so no manual installation is needed for it.

### 6.1 Install nfs-kernel-server and other necessary programs

```bash
sudo apt install nfs-kernel-server chrony
```

### 6.2 Create the primary and secondary folders in `/export/` and set the appropriate access

```bash
sudo mkdir -p /export/primary
sudo mkdir -p /export/secondary
sudo chmod -R 777 /export
```

### 6.3 Configure NFS exports

Add the following to `/etc/exports`:

```
/export/primary *(rw,async,no_root_squash,no_subtree_check)
/export/secondary *(rw,async,no_root_squash,no_subtree_check)
```

### 6.4 Apply the exports and start the NFS service alongside chrony

```bash
sudo exportfs -arv
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server

sudo systemctl enable --now chrony
sudo chronyc tracking
```

### 6.5 Download and unpack the system VM template from Apache

```bash
sudo /usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
-m /export/secondary \
-u http://download.cloudstack.org/systemvm/4.22/systemvmtemplate-4.22.0-x86_64-kvm.qcow2.bz2 \
-h kvm -F
```

---

## 7. Download and Register an Image

### 7.1 Download the image manually in the VM

```bash
wget https://github.com/cirros-dev/cirros/releases/download/0.6.2/cirros-0.6.2-x86_64-disk.img -O /tmp/cirros.img
```

### 7.2 Create a temporary web server to serve the downloaded image

```bash
cd /tmp
python3 -m http.server 8000
```

### 7.3 Create the template

Use the temporary web server URL (for example `http://192.168.126.3:8000/cirros.img`) to create the template in CloudStack.

---

## 8. Appendix: Zone Configuration Values

Values used when creating the zone in the CloudStack UI.

### Zone

| Item         | Value     |
| ------------ | --------- |
| DNS1         | `8.8.8.8` |
| Internal DNS1 | `8.8.8.8` |

### Pod

| Item       | Value            |
| ---------- | ---------------- |
| Gateway    | `172.16.100.1`   |
| Netmask    | `255.255.255.0`  |
| Start IP   | `172.16.100.10`  |
| End IP     | `172.16.100.20`  |

### Host

| Item          | Value                                      |
| ------------- | ------------------------------------------ |
| WSL IP        | `172.16.100.2` (from `ip addr show eth0`)  |
| Hostname      | WSL IP                                     |
| Host username | `root`                                     |
| Host password | `admin123`                                 |
