# Wazuh SIEM Home Lab

## 🎯 Project Overview
This project documents the end-to-end installation of **Wazuh v4.14.1**, an open-source Security Information and Event Management (SIEM) platform. The primary goal of this lab was to establish a functional monitoring environment by deploying a centralized Wazuh Manager on an Ubuntu server and integrating a Windows 11 endpoint.

This guide specifically captures **real-world installation hurdles**, including resource-driven system crashes, disk management issues, and the emergency recovery procedures required to successfully deploy the stack.

I will continue this installation with various lab demonstrations within Wazuh. Currently this is focused around setting up the system and Wazuh.

## 🏗️ Lab Architecture

* **Wazuh Manager**: Ubuntu Server (VirtualBox VM).
    * **Manager IP**: `192.168.1.11` (Use `ifconfig` in terminal and capture the inet)
* **Wazuh Agent**: Windows 11 Home (x64).
    * **Agent IP**: `192.168.1.9` (Use `ipconfig` in the terminal and capture the IPv4)
* **Networking**: Configured via a **Bridged Adapter** to ensure both the Manager and the Agent reside on the same network subnet.

---

## 🛠️ Infrastructure Configuration & Prerequisites
Based on the challenges faced during this deployment, the following hardware configurations are recommended to ensure a stable installation:

* **RAM**: **6144 MB (6GB) (for my vm)**. While 2GB is the absolute minimum for the Indexer to start, 4GB is the recommended value, 6GB provides the necessary overhead for the Java-based components to run smoothly without crashing the kernel.
* **CPU**: **6 Cores (for my vm)**. A minimum of 2 cores is required to handle concurrent processing of the Indexer and Manager and the recommended is 8 cores.
* **Storage**: **50GB VDI**. The Wazuh installation requires at least 10GB of free space for initial setup; 50GB ensures overhead for logs and database shards.
* **Network**: **Bridged Adapter**. This is essential to allow the Ubuntu VM to receive an IP from the physical network, enabling seamless agent-to-manager communication.

<img width="978" height="1290" alt="vm-ubuntu-config" src="https://github.com/user-attachments/assets/60c27462-dd7e-41ae-85f9-f3087764855e" />

---

## 🚀 Phase 1: Manager Installation & Emergency Recovery (Ubuntu)

### 1. Initial Server Preparation
Minimal Ubuntu installs lack essential utilities. You must install these before starting the Wazuh script:
* **System Updates**: `sudo apt update && sudo apt upgrade -y`
* **Networking Tools**: `sudo apt install net-tools` (to identify IP via `ifconfig`)
* **Data Transfer**: `sudo apt install curl` (to fetch scripts and keys)

### 2. Successful Manager Deployment
This installation script was executed:
1. **Add GPG Key**: `curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh-archive-keyring.gpg`

<img width="2868" height="1464" alt="gpgkey-add-wazuh" src="https://github.com/user-attachments/assets/e4331491-5b18-4f80-a127-606c5db832fa" />

---

2. **Install All-in-One**: `curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a -i`

> **Note:** In the tutorial 4.12 verison was used, 4.14 is the latest as of 1st Jan 2026. Checkout the latest version before installing Wazuh

<img width="2880" height="1512" alt="wazuh-install" src="https://github.com/user-attachments/assets/5999288e-b315-41bf-819b-cf8b4c99b620" />
<img width="1248" height="920" alt="wuzuh-ubuntu-installation-complete" src="https://github.com/user-attachments/assets/6c66c14d-18b9-4a47-92e6-d8808ed556fb" />
<img width="2880" height="1518" alt="wazuh-client-browser" src="https://github.com/user-attachments/assets/ce66fdb9-a756-41cd-9c32-fbbdc2a88d92" />

### 3. Overcoming the Installation Crash

During the initial run, low disk space caused a system-wide crash, resulting in the faliure from **booting to login**.

<img width="1488" height="956" alt="ubuntu-wuzuh-boot-error" src="https://github.com/user-attachments/assets/13f3558b-33a0-488d-a49d-a5557ac16d2a" />

**The Recovery Procedure:**
1. **Emergency Access**: Intercepted the boot process via **GRUB Recovery Mode** to access a **root shell**.

<img width="874" height="768" alt="recoverymode-select" src="https://github.com/user-attachments/assets/5b5f1c5c-4574-4eb1-b6e9-b41169cbf8db" />
<img width="932" height="576" alt="recovery-boot-mode" src="https://github.com/user-attachments/assets/18c43f2a-4332-47b0-a54b-d3bb376f1c1f" />
<img width="870" height="526" alt="recoverymode-shell-method2" src="https://github.com/user-attachments/assets/8b45380d-03bb-4712-9268-a1c72e8bea49" />

2. **Filesystem Repair**: Remounted the drive as read-write: 
    ```
    mount -o remount,rw /
    ```
3. **Clean Slate**: Purged corrupted components to resolve service deadlocks, and reboot the system:
    ```bash
    apt-get purge wazuh-manager wazuh-indexer wazuh-dashboard -y
    apt-get autoremove -y
    reboot -f
    ```

4. **Live Partition Expansion**: Extended the volume from the VirtualBox `media` menu item.
Used `growpart` to expand the physical partition and `lvextend` to grow the logical volume:
    ```bash
    sudo apt update && sudo apt install cloud-guest-utils -y
    sudo growpart /dev/sda 3
    sudo pvresize /dev/sda3
    sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv -r
    ```
5. **To verify the changes:**
    ```
    df -h /
    ```

### 4. Post-Deployment: Credential Recovery

If you have successfully installed the stack but did not write down the admin password, or if you lost access after the dashboard initialized, you can retrieve the auto-generated credentials using:

`sudo cat /etc/wazuh-indexer/wazuh-passwords.txt`

---

## 🚀 Phase 2: Agent Installation & Registration (Windows 11)

### 1. Installation
1. Downloaded the Wazuh agent MSI installer.
2. Installed using default settings on the Windows host.

<img width="2880" height="1450" alt="wazuh-agent-windows-installation" src="https://github.com/user-attachments/assets/7428d9c3-2c87-48bc-bd1b-aadb818c6e7c" />

### 2. Registration
1. **Generate Key**: On the Ubuntu Manager, ran the management utility:
    `sudo /var/ossec/bin/manage_agents`

    1. Select `A` to add an agent. 
    2. Assign a name to the device. 
    3. Leave IP address blank unless static assignment is needed. 

<img width="1148" height="806" alt="adding-windows-agent" src="https://github.com/user-attachments/assets/efe82b0f-054c-49cf-8372-e742ecd890f4" />

2. **Extraction**: Selected **A** to add the agent and **E** to extract the generated key.

    1. select `E` to extract the key. 
    2. Copy the key output.

<img width="1144" height="820" alt="windows-agent-key" src="https://github.com/user-attachments/assets/e3b069cb-5d94-4f8c-993f-327eb0e81d91" />

3. **Apply Key**: Opened the **Wazuh Agent Manager GUI** on Windows, pasted the key, and entered the Manager's IP (`192.168.1.11`).

<img width="638" height="560" alt="wazuh-agent-confirmation" src="https://github.com/user-attachments/assets/dbdf7742-36b4-45d1-9dfc-821435b120fd" />

4. **Service Restart**: Restarted the agent service to finalize the connection.

### 3. Verify the agent in the Wazuh Dashboard

<img width="2756" height="1350" alt="windows-agent-on-dashboard" src="https://github.com/user-attachments/assets/161ca54a-edee-481e-8ca7-22728c4e9acd" />

---

## 📚 Credits
This lab was completed following the instructional guide by **Royden Rebello (The Social Dork)**.


https://www.youtube.com/watch?v=QT81wcuoRFY

