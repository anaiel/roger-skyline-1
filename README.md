# roger-skyline-1

This project was completed in August 2019 as part of the 42 cursus. roger-skyline-1 is the second project of the SysAdmin branch of the cursus, which aims at giving an introduction to systems administration problematics. For this project, the goal was to create a virtual machine and play with some basic networking concepts.

![path to roger-skyline-1](https://i.imgur.com/MB6rV4f.png "SysAdmin branch > Init > roger-skyline-1")

*Systems administration and networks, Unix*

This page is intended to be a walkthrough to complete the project for someone who has no idea how a virtual machine or systems administration works. My machine is running with Debian 10.0.0.

⚠️ *I am not an experienced SysAdmin. In fact, this project is as far as I went down that road. So please take everything written here with a grain of salt*

**Overview of the project**
- [Setting up the VM](#setting-up-the-vm)
    + [Creating the VM](#creating-the-vm)
    + [sudo](#sudo)
    + [Static IP](#static-ip)
- [Setting up the SSH connexion](#setting-up-the-ssh-connexion)
- [Protecting the machine](#protecting-the-machine)
    + [Firewall](#firewall)
    + [DoS attacks](#dos-attacks)
    + [Port scanning](#port-scanning)
- [Scripts](scripts)

## Setting up the VM

### Creating the VM

*📕 Resources: [Hypervisor @Wikipedia](https://en.wikipedia.org/wiki/Hypervisor), [Installing Debian Linux in a VirtualBox Virtual Machine](http://www.brianlinkletter.com/installing-debian-linux-in-a-virtualbox-virtual-machine/)*

1. Download the debian image ([for example, here](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-10.0.0-amd64-netinst.iso))
2. Launch VirtualBox, click new, and follow the steps (be careful to choose the *Fixed size* option)
3. Insert the debian image: click on *Settings* then *Storage* and under *Controller: IDE* choose the image you have previously downloaded.
4. To prepare for the **[Static IP](#static-ip)**, in the settings of the machine, choose *Bridged Adapter* in the *Network* menu.
5. Launch the machine and follow the installation process: choose the root password, the initial user, etc. A couple of steps to watch for:
    - Partitioning: choose the automatic partitioning, then delete the existing partition and create a 4.2GB partition in the FREE SPACE (you can choose primary partitioning, for more info about this see [read this](https://www.quora.com/What-is-the-difference-between-Primary-and-logical-partition)).
    - Only install SSH and standard tools, nothing else.
    - Domain and proxy can be left blank.

### sudo

1. Connect to your machine as root (at boot or using `su root`)
2. Install sudo: `apt install sudo`
3. Add you initial user to the sudoers group: edit the **/etc/sudoers** file to look like this :
    ```
    # Allow members of group sudo to execute any command
    %sudo   ALL=(ALL:ALL)   ALL
    <username>  ALL=(ALL:ALL)   NOPASSWD:ALL
    ```
    Alternatively, you can use the command `adduser <username> sudo`

## Static IP

💡 *This part could use a little more pedagogy, I'll get to it when I have a better grasp on why this works!*

📕 *Resources: [What is a netmask?](https://www.computerhope.com/jargon/n/netmask.htm), [IP Calculator](http://jodies.de/ipcalc), and also kudos to gde-pass for [his roger-skyline-1 walkthrough](https://github.com/gde-pass/roger-skyline-1#staticIP)*

1. Edit the **/etc/network/interfaces** file to look like this:
    ```
    # The primary network interface
    auto enp0s3
    ```
2. Create the **/etc/network/interfaces.d/enp0s3** file:
    ```
    iface enp0s3 inet static
        address 10.11.200.233
        netmask 255.255.255.252
        gateway 10.11.254.254
    ```
    NB: I chose 10.11.200.233 but you can use the IP the dhcp service gave you before you deactivated it (`ip addr` and write down the enp0s3 IP).
    NB: 255.255.255.252 corresponds to a /30 netmask.
3. Restart the networking service: `service networking restart`

-------

⚡️ **Testing**

- [x] You can check the partitions with the `fdisk -l` command.
    ⚠️ *It is normal for the partition to be only 3.9GB when you set it up to be 4.2GB because fdisk displays the size in gibibytes. [A simple conversion](https://www.gbmb.org/gib-to-gb) shows that 3.9GiB = 4.2GB.*
- [x] You can test that your initial user has sudo rights by running any command with sudo :
    ```
    su <username>
    sudo apt update
    ```
- [x] You can check your ip address with the `ip addr` command.
- [x] The evaluation asks that you change the netmask and reconfigure whatever you need to reconfigure. I'm not sure I get the point of this instruction, but here's what you can do: edit the **/etc/network/interfaces.d/enp0s3** file and change the value of the netmask. A simple `service networking restart` will not work to change the netmask (you can check that `ip addr` returns the same result as before, so use `systemctl restart networking` instead.

------------

## Setting up the SSH connexion

📕 *Resources: [How does SSH work?](https://www.hostinger.com/tutorials/ssh-tutorial-how-does-ssh-work), [How to use ssh-keygen](https://www.ssh.com/ssh/keygen/)*

1. Edit the **/etc/ssh/sshd_config** file:
    - Uncomment `PasswordAuthentification yes` (you need to be able to use a password to connect to the machine for now, in order to copy the ssh key you will generate on the host machine to the VM without using a ssh key... because you haven't copied the key yet)
    - Uncomment `Port 22` and choose the port you want. I'll use 2222 in the future
    - Uncomment `PermitRootLogin no`
3. Restart the ssh service: `service ssh restart`
2. Setup the ssh key for <username>. Type the following commands in the host machine:
    - `ssh-keygen` (you can keep the default location for your key if you haven't generated a key before)
    - `ssh-copy-id -i ~/.ssh/id_rsa <username>@10.11.200.233 -p 2222`
    🔮 *This will automatically copy the generated key to the ~/.ssh/authorized_keys file of <username>. It will ask for <username>'s password to connect to the VM.This is why you needed to leave the possibility of password identification, otherwise you would only have been able to copy the key to the machine via ssh pubkey identification which you haven't set up yet. Or you would have had to copy by hand the key to the ~/.ssh/authorized_keys file, which would have been a pain in the ass.*
3. Edit again the **/etc/ssh/sshd_config** file and change `PasswordAuthentification no`.
4. `service ssh restart`

---------

⚡️ **Testing**

In the host machine:
- [x] `ssh <username>@10.11.200.233 -p 2222` should open an ssh connexion
- [x] `ssh randomuser@10.11.200.233 -p 2222` should return a pubkey error
- [x] The evaluation asks that you create a sudo user with a ssh key. The ssh key part can be a pain in the ass so don't hesitate to show that your <username>'s ssh connexion works fine. Otherwise, here's what you can do:
    - In the VM (as root (don't use sudo then) or as <username>):
        ```
        sudo adduser newuser
        sudo adduser newuser sudo
        ```
    - In the host machine: `ssh-keygen` and choose a different location for the file so as to not overwrite the <username>'s key. For example choose **/Users/<username/.ssh/newuser**
    - Here you can manually type the generated key to the VM's **/home/newuser/.ssh/authorized_keys** or edit **/etc/ssh/sshd_config** and enable PasswordAuthentification and restart the ssh service and `ssh-copy-id -i ~/.ssh/newuser <username>@10.11.200.233 -p 2222` and disable PasswordAuthentification and restart the ssh service. Phew.
    - Now you can check that ssh authentification works for newuser with `ssh -i ~/.ssh/newuser newuser@10.11.200.233`

-----------------

## Protecting the machine

### Firewall

📕 *Resources: [Debian firewall](https://wiki.debian.org/DebianFirewall), [Uncomplicated Firewall](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29)*

For this part you can propably try and edit the iptable yourself with your bare hands. But Uncomplicated Firewall is, well... uncomplicated and makes it really easy to set up the firewall.

1. Install Uncomplicated Firewall : `apt install ufw`
2. `ufw enable`
3. The subject asks that only the usefull services should be accessible, so you need to close all the ports except for ssh (mine is 2222), http and https (for the optional webserver part) :
    ```
    ufw default deny incoming
    ufw default allow outgoing
    ufw allow 2222
    ufw allow http
    ufw allow https
    ```
    🔮 *Even if you don't do the webserver part, allow the http and https ports. It will be useful when you set up PortSentry, which won't work if only one port is open, but will work if 3 are*
4. `service ufw restart`

### DoS attacks

📕 *Resources: [Denaial of Service attacks: definition and prevention](https://javapipe.com/blog/denial-of-service-attack/), [Sécuriser votre serveur avec fail2ban](https://www.geek17.com/fr/content/debian-9-stretch-securiser-votre-serveur-avec-fail2ban-31)*

Once again, you can probably try to play with the iptable to do this, but fail2ban makes it easier (and is also mentioned in the evaluation).

1. Install fail2ban: `apt install fail2ban`
2. Enable fail2ban on opened ports: edit the  **/etc/fail2ban/jail.conf** file to look like this:
    ```
    [sshd]
    # Comment...
    enabled = true
    port = 2222
    logpath = #...
    ```
3. (optional) It is also possible to change the number of failed attempts and ban time in the same file: change `bantime = 180` and `maxretry = 5`.
4. `service fail2ban restart`

### Port scanning

📕 *Resources: [Se protéger contre le scan de ports avec PortSentry](https://fr-wiki.ikoula.com/fr/Se_prot%C3%A9ger_contre_le_scan_de_ports_avec_portsentry)*

1. Install PortSentry: `apt install portsentry`
2. Use advanced tcp and udp mode: edit the **/etc/default/portsentry** file to look like this:
    ```
    TCP_MODE="atcp"
    UDP_MODE="audp"
    ```
3. Block udp and tcp scans and add the offending IPs to the iptable: edit the **/etc/portsentry/portsentry.conf** file to look like this (uncomment this KILL_ROUTE and comment the others):
    ```
    BLOCK_UDP="1"
    BLOCK_TCP="1"
    ```
    ```
    KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
    ```
 4. `service portsentry restart`
      
-------

⚡️ **Testing**

- [x] You can list the firewall rules with `ufw status verbose` or `iptables --list`. The default policy should be DROP. `iptables --list | grep 2222` should show that the policy for 2222 is ACCEPT. Same for http and https.
- [x] To simulate a DoS attack, you can use Slowloris (you can refer to [this tutorial](https://www.yeahhub.com/perform-dos-attack-5-different-tools-2018-update/)) :
    * In the host machine:
        + Download slowloris: `git clone https://github.com/llaera/slowloris.pl`
        + Launch the attack:
            ```
            cd slowloris.pl
            perl slowloris.pl -dns 10.11.200.233
            ```
            Nothing should happen.
    * In the the VM, `iptables --list | head` should show your IP address is banned.
    * Stop the attack, and remove your IP from the list of banned IP: `iptables -D INPUT 1`.
    * `service restart portsentry` (it needs to take into account your change in the iptables)
- [x] To simulate the portscan, use nmap (you can refer to [this tutorial](https://www.memoinfo.fr/tutoriels-linux/tuto-nmap-scaner-les-ports-ouverts/) :
    * In the host machine:
        + Download nmap: `brew install nmap`
        + Launch the scan: `nmap 10.11.200.233`. Nothing should happen.
    * In the the VM, `iptables --list | head` should show your IP address is banned.
    * `iptables -D INPUT 1`
    * `service restart portsentry`

-------

## Scripts

The first script should perform `apt update` and `apt upgrade` and log the result in the **/var/log/update_script.log** file. I added a timestamp at the beginning to know when the update was performed:

```
LOGFILE=/var/log/pudate_script.log

date `+[%c]` >> $LOGFILE
apt update -y 2>/dev/null >> $LOGFILE
apt upgrade -y 2>/dev/null >> $LOGFILE
echo "" >> $LOGFILE
```

💡 *In case you didn't know:*
- *`2>/dev/null` redirects any message on the error output (file descriptor 2) to /dev/null (basically, it sends it to the void)*
- *`>>` appends to a file while `>` overwrites a file.*

The second script should monitor the **/etc/crontab** file and send an email if it was modified since the last check was performed. Therefore, you need some kind of backup of the file to compare the old one with the current one. I chose to store a hash of the file in another file (**/var/tmp/cron_hash**).
You'll need to download mailutils to be able to send a mail (`apt install mailutils`).

```
CRON=/etc/crontab
CRONHASH=/var/tmp/cron_hash

if [ -f $CRON ]; then
    if [ -f $CRONHASH ]; then
        if [ "$(cat $CRONHASH)" != "$(md5sum $CRON)" ]; then
            echo "$CRON was modified" | mail -s "cronCheck report" root@roger
        fi
    fi
    md5sum $CRON > $CRONHASH
fi
```

🔮 *By default, there is an alias of root to your initial users. Which means the mail you send to root will be in <username>'s inbox. You can change it in the /etc/aliases file. But even after deleting the alias or changing it for `root: root`, you will not be able to read the mail received by root using the `mail` command, because the mail will then be stored in /var/mail/mail and not /var/mail/root. There's probably a way to fix that though, but I couldn't be bothered as it is not super useful for this project.*

Now that the scripts are done, you need to program their execution with crontab. Use `crontab -e` to create and edit (or only edit if it already exists) a crontab (it will not be the same as the **/etc/crontab** as this is the system's crontab). Then edit it to look like this (minus the comments):

```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
  
@reboot sh /update.sh       #execute the update script when the vm reboots
0 4 * * 1 sh /update.sh     #execute the update script when it is 0 minutes past 4 hours, whatever the day of the month, every month, on Monday
0 0 * * * sh /cronCheck.sh  #execute the cronCheck script when it is 0 mintutes past 0 hours, every day, every month, every day of the week
```

-------

⚡️ **Testing**

- [x] The **/var/log/update_script.log** should have a log of the boot at the begining of the evaluation. You can also execute the script and check that a log was added at the end of the file.
- [x] For the crontab checker :
    * Execute the script
        + If you kept the alias and are logged in as <username>, `mail` should tell you `No mail for <username>
        + If you kept the alias and are logged in as someone else, **/var/mail/<username>** should be empty
        + If you deleted the alias, **/var/mail/mail** should be empty
    * Edit **/etc/crontab** and execute the script. Now the same method as before should tell you that there is mail.
