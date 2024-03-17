# SECURE YOUR NEAR VALIDATOR

__Our tutorials are based on Ubuntu!__

Before you start installing the NEAR software on your fresh node, consider hardening your server's security first. 

---

## USE ROOT OR A DEDICATED USER?

Our tutorials are based on Docker which delivers containerization, improving the overall security of your node. Normally, you would first create a dedicated user for your project. We won't do that for our NEAR validator as to get you up to speed quickly while still maintaining a decent level of security. Therefore, all the commands found in our tutorials start with `root@node ~ #` instead of `user@node ~ $ sudo`.

---

## CHANGE SSH PORT

* Open SSH config: `root@node ~ # nano /etc/ssh/sshd_config`
* If line 14 says `#Port 22`, change it to your desired port by uncommenting the line and changing the port:
    - We use port `58899` in this tutorial

```bash
...truncated
Include /etc/ssh/sshd_config.d/*.conf

Port 58899
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
...truncated
```

* Restart SSH: `root@node ~ # systemctl restart ssh sshd`

__Try to open a second terminal session using the new SSH port to see if you can connect before closing this one!__

---

## DISABLE SSH PASSWORD AUTHENTICATION

*Local commands (i.e. on your laptop) are prefixed with `$` and server commands with `#`.*

#### USE SSH KEYS INSTEAD OF A PASSWORD

* If you don't have an SSH keypair yet, create one on your LOCAL machine: `$ ssh-keygen -t ed25519`
* Add your LOCAL public key (`~/.ssh/id_ed25519.pub`) to the server: `root@node ~ # nano /root/.ssh/authorized_keys`
* Open a second terminal session to test if you can login without a password using the SSH key: 
`$ ssh -i ~/.ssh/id_ed25519 root@YOUR.NODE.IP.ADDRESS`

__Only continue with the next steps if you were able to connect without a password!__

## CONFIGURE SSH TO ONLY USE SSH KEYS

* Open SSH config: `root@node ~ # nano /etc/ssh/sshd_config`
* Change/confirm these settings: 

```
...
PermitRootLogin without-password
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
...
```

* Restart SSH: `root@node ~ # systemctl restart ssh sshd`
* Our final version (FYI only):

```
Include /etc/ssh/sshd_config.d/*.conf
Port 58899
PermitRootLogin without-password
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
```

---

## CONFIGURE UFW (uncomplicated firewall)

The UFW (uncomplicated firewall) is exactly what it advertises: uncomplicated. We are going to enter some commands to allow SSH traffic, NEAR P2P traffic and optionally something to only allow your (static) IP to access the server and/or NEAR RPC.


#### ALLOW SSH TRAFFIC

If you don't have a static IP (with or without a VPN), you probably want to allow traffic to your SSH port from any external IP address, which is why we set a custom SSH port earlier in this tutorial. 

__If you DO have a static IP, skip this step and add the optional UFW rule for allowing your static IP instead.__

* Allow traffic (for everyone!) to your custom SSH port and reload the firewall: 
    - We use port `58899` in this tutorial

```bash
root@node ~ # ufw allow 58899/tcp comment 'SSH'
```

* Allow traffic (for everyone!) to the NEAR P2P port:

```bash
root@node ~ # ufw allow 24567/tcp comment 'NEAR'
```

* (optional) Allow your static IP to connect to ANY port:

```bash
root@node ~ # ufw allow from 11.22.33.44 comment 'MYSTATICIP'
```

* Enable the firewall:

```bash
root@node ~ # ufw enable
```

* Check the configured rules:

```bash
root@node ~ # ufw status numbered
```

Example output:

```
root@node ~ # ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 58899/tcp                  ALLOW IN    Anywhere                   # SSH
[ 2] 24567/tcp                  ALLOW IN    Anywhere                   # NEAR
[ 3] Anywhere                   ALLOW IN    11.22.33.44                # MYSTATICIP
[ 4] 58899/tcp (v6)             ALLOW IN    Anywhere (v6)              # SSH
[ 5] 24567/tcp (v6)             ALLOW IN    Anywhere (v6)              # NEAR
```


#### CUSTOM RPC 

Docker adds its own custom firewall rules, effectively bypassing yours. This is why we bind the RPC port (3030) to `localhost` (127.0.0.1) instead of the default in our tutorial LINK_HERE.

To allow traffic to your RPC port to check the status of your node externally, we need to configure UFW to allow external connections to `localhost` (127.0.0.1) which is called a 'martian'.

For this, we need to add 3 lines of code to the UFW 'before rules'.

* Edit file: `root@node ~ # nano /etc/ufw/before.rules`
* Add this at the __BOTTOM__ after the final 'COMMIT':

```bash
*nat :PREROUTING ACCEPT [0:0]
-A PREROUTING -p tcp -i enp35s0 --dport 3030 -j DNAT --to-destination 127.0.0.1:3030
COMMIT
```

__IMPORTANT!__

In the example above, we used `enp35s0` as our network adapter. Find yours by running `root@node ~ # ip a` and copying the network adapter name that holds your external IP address. Example (with `eth0` instead of `enp35s0`):

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether a6:a2:50:0f:1b:3e brd ff:ff:ff:ff:ff:ff
    inet 11.22.33.44/32 scope global eth0
       valid_lft forever preferred_lft forever
```

* Restart UFW: `root@node ~ # ufw disable && ufw enable`
* And finally, specifically let your system configuration know that Martian's are allowed:

```bash
root@node ~ # echo "net.ipv4.conf.all.route_localnet=1" > /etc/sysctl.d/90-enable-route-localnet.conf
root@node ~ # sysctl -p /etc/sysctl.d/90-enable-route-localnet.conf
```