# roger-skyline-1

This project was completed in August 2019 as part of the 42 cursus. roger-skyline-1 is the second project of the SysAdmin branch of the cursus, which aims at giving an introduction to systems administration problematics. For this project, the goal was to create a virtual machine and play with some basic networking concepts.

This page is intended to be a walkthrough to complete the project for someone who has no idea how a virtual machine or systems administration works. My machine is running with Debian 10.0.0

![path to roger-skyline-1](https://i.imgur.com/MB6rV4f.png "SysAdmin branch > Init > roger-skyline-1")

*Systems administration and networks, Unix*

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

Resources: [Hypervisor @Wikipedia](https://en.wikipedia.org/wiki/Hypervisor), [Installing Debian Linux in a VirtualBox Virtual Machine](http://www.brianlinkletter.com/installing-debian-linux-in-a-virtualbox-virtual-machine/)

1. Download the debian image ([for example, here](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-10.0.0-amd64-netinst.iso)
2. Launch VirtualBox, click new, and follow the steps (be careful to choose the *Fixed size* option)
3. Insert the debian image: click on *Settings* then *Storage* and under *Controller: IDE* choose the image you have previously downloaded.
4. To prepare for the **[Static IP](#static-ip)**, in the settings of the machine, choose *Bridged Adapter* in the *Network* menu.
5. Launch the machine and follow the installation process: choose the root password, the initial user, etc. A couple of steps to watch for:
    - Partitioning: choose the automatic partitioning, then delete the existing partition and create a 4.2GB partition in the FREE SPACE. (you can choose primary partitioning, for more info about this see [read this](https://www.quora.com/What-is-the-difference-between-Primary-and-logical-partition)
    - Only install SSH and standard tools, nothing else.
    - domain and proxy can be left blank

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

## IP Fixe

💡 *This part could use a little more pedagogy, I'll get to it when I have a better grasp on why this works!

Resources: [What is a netmask?](https://www.computerhope.com/jargon/n/netmask.htm), [IP Calculator](http://jodies.de/ipcalc), and also kudos to @gde-pass for [his roger-skyline-1 walkthrough](https://github.com/gde-pass/roger-skyline-1#staticIP)

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

⚡️ **Testing**

- You can check the partitions with the `fdisk -l` command. ⚠️ *It is normal for the partition to be only 3.9GB when you set it up to be 4.2GB because fdisk displays the size in gibibytes. [A simple conversion](https://www.gbmb.org/gib-to-gb) shows that 3.9GiB = 4.2GB.
- You can test that your initial user has sudo rights by running any command with sudo :
    ```
    su <username>
    sudo apt update
    ```
- You can check your ip address with the `ip addr` command.
- The evaluation asks that you change the netmask and reconfigure whatever you need to reconfigure. I'm not sure I get the point of this instruction, but here's what you can do: edit the **/etc/network/interfaces.d/enp0s3** file and change the value of the netmask. A simple `service networking restart` will not work to change the netmask (you can check that `ip addr` returns the same result as before, so use `systemctl restart networking` instead.

## Setting up the SSH connexion

Resources: [How does SSH work?](https://www.hostinger.com/tutorials/ssh-tutorial-how-does-ssh-work), [How to use ssh-keygen](https://www.ssh.com/ssh/keygen/)

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

⚡️ **Testing**

In the host machine:
- `ssh <username>@10.11.200.233 -p 2222` should open an ssh connexion
- `ssh randomuser@10.11.200.233 -p 2222` should return a pubkey error
- The evaluation asks that you create a sudo user with a ssh key. The ssh key part can be a pain in the ass so don't hesitate to show that your <username>'s ssh connexion works fine. Otherwise, here's what you can do:
    - In the VM (as root (don't use sudo then) or as <username>):
        ```
        sudo adduser newuser
        sudo adduser newuser sudo
        ```
    - In the host machine: `ssh-keygen` and choose a different location for the file so as to not overwrite the <username>'s key. For example choose **/Users/<username/.ssh/newuser**
    - Here you can manually type the generated key to the VM's **/home/newuser/.ssh/authorized_keys** or edit **/etc/ssh/sshd_config** and enable PasswordAuthentification and restart the ssh service and `ssh-copy-id -i ~/.ssh/newuser <username>@10.11.200.233 -p 2222` and disable PasswordAuthentification and restart the ssh service. Phew.
    - Now you can check that ssh authentification works for newuser with `ssh -i ~/.ssh/newuser newuser@10.11.200.233`

## Protections
- Firewall
    - `apt install ufw`
      ```
      ufw enable
      ufw default deny incoming #changes the default policy and drops any oncoming traffic
      ufw default allow outgoing #allows outgoing traffic by default
      ufw allow 2222 #allows port 2222 (ssh)
      ufw allow http #pas nécessaire pour la partie obligatoire mais permet quand meme a portsentry de fonctionner
      ufw allow https
      ```
    - pour verifier l'etat : ufw status verbose
    - pour consulter la totalité des regles de pare feu : `iptables --list`
- DoS
    - `apt install fail2ban`
    - modifier le ficher `/etc/fail2ban/jail.conf` : il faut que la partie sur le ssh ressemble à ça :
      ```
      [sshd]
      # Comment...
      enabled = true
      port = 2222
      logpath = #...
      ```
    - dans le meme fichier il est aussi possible de modifier le nombre de tentative erronées avant le banissement ou le temps de bannissement
- Scan de ports
    - `apt install portsentry`
    - modifier le fichier /etc/default/portsentry
      ```
      TCP_MODE="atcp"
      UDP_MODE="audp"
      ```
    - modifier le ficher /etc/portsentry/portsentry.conf
      ```
      BLOCK_UDP="1"
      BLOCK_TCP="1"
      ```
      ```
      KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP" # Décommenter cette ligne et commenter les autres KILL_ROUTE
      ```
  
### Tester

- afficher l'ip table : `iptables --list`
- tester le firewall (arreter portsentry d'abord `service portentry stop` puis utiliser `nmap 10.11.200.233` depuis le host (penser à réactiver portsentry : `service portsentry start`
- tester le DoS:
    - faire plusieurs tentatives de connexion ssh erronées
    - utiliser slowloris :
        ```
        git clone https://github.com/llaera/slowloris.pl
        cd slowloris.pl
        perl slowloris.pl -dns 10.11.200.233
        ```
    - pour retirer l'ip des ip bannies, utiliser la commande `iptables -D INPUT 1`
 - tester le scan de port : `nmap 10.11.200.233` (puis penser à retirer l'ip des ip bannies)


## Arrêter les services

- `service <service> stop`

### Tester
- pour voir les services : `service --status-all`
- Pour voir la liste des services + description : `systemctl list-units | grep service`

## Scripts

- `apt install mailutils` pour pouvoir envoyer des mails
- crontab -e pour créer la crontab et éditer le contenu pour planifier les scripts
    ```
    SHEL=/bin/bash
    PATH=/sbin:/bin:/usr/sbin:/usr/bin
    
    @reboot sh /update.sh
    0 4 * * 1 sh /update.sh
    0 0 * * * sh /cronCheck.sh
    ```
    
### Tester
- Pour le script d'update : `cat /var/log/update_script.log` doit avoir le log du boot de debut de correction
- Pour le script de crontab : modifier le fichier `/etc/crontab` et faire `sh /cronChecker.sh` puis `mail` pour vérifier que le mail a été reçu (attention, il y a un alias, donc le mail est reçu dans la boite de anleclab. Sinon, il faut regarder dans /var/mail/mail si on supprime l'alias dans /etc/aliases.
