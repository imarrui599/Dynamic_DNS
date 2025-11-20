# Dynamic DNS (DDNS) Lab with Ansible and Vagrant

This project deploys a fully automated network environment configuring **Dynamic DNS (DDNS)**. The goal is to demonstrate how a DHCP server (ISC) and a DNS server (BIND9) can communicate securely to automatically update name records when an IP is assigned to a client.

## 📋 Scenario Description

The environment spins up three virtual machines using **Vagrant** and configures them via **Ansible**:

| Host | Hostname | IP | Role |
|------|----------|----|---------|
| **dns** | `dns.example.test` | `192.168.58.10` | Master DNS Server (BIND9) |
| **dhcp** | `dhcp.example.test` | `192.168.58.20` | DHCP Server (ISC-DHCP) |
| **cl** | `cliente1` | *Dynamic* | Test Client |

### ⚙️ How it Works

The core of this project is the automation of trust between servers using a shared secret (**TSIG Key**). The Ansible playbook execution flow is as follows:

1.  **DNS Configuration (`dns`)**:
    * Installs BIND9.
    * Generates a secure TSIG key (`ddns.key`).
    * **Important:** Ansible fetches this generated key from the VM to your local machine (`ansible/fetched_keys/`).
    * Configures the `example.test` zone and the reverse zone `58.168.192` to allow updates only if this specific key is presented.

2.  **DHCP Configuration (`dhcp`)**:
    * Installs the ISC DHCP server.
    * Copies the `ddns.key` (previously fetched) from your local machine to the DHCP server.
    * Configures the service so that every time it assigns an IP, it sends a secure update to the DNS server using the key.

## 🚀 Requirements

* [Vagrant](https://www.vagrantup.com/)
* [VirtualBox](https://www.virtualbox.org/)
* [Ansible](https://www.ansible.com/) (Installed on the host machine)

## 🛠️ Deployment

1.  Clone this repository.
2.  Start the environment with a single command:

```bash
vagrant up