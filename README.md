
## How My Ansible Playbook Works

To bring my `Vagrantfile` to life, I created this Ansible playbook. My goal wasn't just to *install* services, but to make them **work together automatically**. The core of this project is to create a secure, dynamic link between my DHCP and DNS servers. This is known as **Dynamic DNS (DDNS)**.

The entire process is split into two main plays:

1.  **Configuring the BIND9 DNS Server**
2.  **Configuring the ISC DHCP Server**

The magic is in how I make them trust each other using a shared secret: the **TSIG Key**.



---

### Play 1: Configuring the BIND9 DNS Server (`dns_servers`)

This play's job is to set up a BIND9 server that is ready to accept secure updates from the DHCP server.

1.  **Install BIND:** First, I use `ansible.builtin.apt` to install `bind9` and `bind9utils`.
2.  **Generate the TSIG Key:** I use the `tsig-keygen` command to create a secure key. I set this task to `creates: /etc/bind/ddns.key`, which means Ansible will only generate the key *once* and won't overwrite it on future runs.
3.  **Secure the Key:** I immediately fix the permissions on `ddns.key` so it's readable by the `bind` group.
4.  **Fetch the Key:** This is a **critical step**. I use `ansible.builtin.fetch` to pull the `ddns.key` from the DNS server back to my Ansible controller (my host machine) and save it in `ansible/fetched_keys/`. This allows me to share this *exact* key with the DHCP server in the next play.
5.  **Configure BIND:**
    * I include the `ddns.key` in the main BIND options (`named.conf.options`).
    * I create the initial zone files (`db.example.test` for forward lookups and `db.192.168.58` for reverse lookups).
    * I configure `named.conf.local` to define both zones. Most importantly, I add `allow-update { key "ddns-key"; };` to both. This tells BIND: "Do not accept updates to these zones unless they are signed with this specific key."
6.  **Restart:** Finally, I restart the `bind9` service to apply all my changes.

---

### Play 2: Configuring the ISC DHCP Server (`dhcp_servers`)

This play's job is to install the DHCP server and, more importantly, configure it to securely *send* updates to the DNS server.

1.  **Install DHCP:** I start by installing the `isc-dhcp-server`.
2.  **Copy the TSIG Key:** This is the other half of the key exchange. I use `ansible.builtin.copy` to **push** the `ddns.key` (which I *fetched* in Play 1) from my controller's `ansible/fetched_keys/` folder *to* the DHCP server's `/etc/dhcp/` directory. Now, both servers share the exact same secret.
3.  **Configure `dhcpd.conf`:** This is where I tie everything together.
    * `ddns-update-style interim;`: I tell the DHCP server to perform DDNS updates.
    * `include "/etc/dhcp/ddns.key";`: I tell it where to find the secret key.
    * `subnet ...`: I define the IP pool (`192.168.58.100` to `192.168.58.200`) and tell clients that their DNS server is `192.168.58.10` (my BIND server).
    * `zone ...` blocks: I explicitly define the forward (`example.test.`) and reverse (`58.168.192.in-addr.arpa.`) zones. For each, I specify the DNS server's IP (`primary 192.168.58.10;`) and the `key "ddns-key";` to use when authenticating.
4.  **Restart:** I restart the `isc-dhcp-server` service.

---

### The Final Result

When I run `vagrant up`:

1.  My `cl` (client) machine boots up with no IP.
2.  It shouts on the network for an IP (DHCP request).
3.  My `dhcp` server catches the request, gives it an IP (e.g., `192.168.58.100`), and tells it the DNS server's address.
4.  **The Magic:** The `dhcp` server then uses the shared `ddns-key` to securely tell the `dns` server: "Hey, I just gave `192.168.58.100` to `cliente1`. Please create an 'A' record for `cliente1.example.test` and a 'PTR' record for `192.168.58.100`."
5.  My `dns` server, seeing the valid key, accepts the update.

Now, from any machine, I can `ping cliente1.example.test` and it resolves correctly, all without any manual intervention.
```
