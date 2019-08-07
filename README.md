# roger-skyline-1

This project was completed in August 2019 as part of the 42 cursus. roger-skyline-1 is the second project of the SysAdmin branch of the cursus, which aims at giving an introduction to systems administration problematics. For this project, the goal was to create a virtual machine and play with some basic networking concepts.

This page is intended to be a walkthrough to complete the project for someone who has no idea how a virtual machine or systems administration works. My machine is running with Debian 10.0.0

![path to roger-skyline-1](https://i.imgur.com/MB6rV4f.png "SysAdmin branch > Init > roger-skyline-1"]

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

### Tester

- Vérifier les partitions `sudo fdisk -l` ⇒ donne le resultat en GibiBytes donc normal si ça ne correspond pas (3.9). Voir conversion ici :
- Tester le sudo :
```
su anleclab
sudo apt update
sudo adduser guest
su guest
sudo apt update
exit
sudo /etc/sudoers #Rajouter la ligne guest (ALL:ALL) NOPASSWD:ALL
su guest
sudo apt update
exit
sudo deluser --remove-home guest
```

## IP Fixe
- Netmask /30 = 255.255.255.252
- dans la config vb de la vm, mettre le network en bridge
- editer /etc/network/interfaces pour mettre `auto enp0s3`
- creer un fichier /etc/network/interfaces.d/enp0s3 avec les specifications appropriées
```
iface enp0s3 inet static
      address 10.11.200.233
      netmask 255.255.255.252
      gateway 10.11.254.254
```
- restart et checker avec la commande `ip addr`

### Tester


## SSH
- Editer le fichier /etc/ssh/sshd_config
    - décommenter PasswordAuthentification yes
    - décommenter Port 22 et changer en Port 2222
    - ajouter PermitRootLogin no
- Générer les clés (dans l'hote)
    - `ssh-keygen` et suivre les steps
    - `ssh-copy-id -i ~/.ssh/id_rsa anleclab@10.11.200.233 -p 2222`
- /etc/ssh/sshd_config PasswordAuthentification no

### Tester
Dans le host
```
ssh anleclab@10.11.200.233 -p 2222
exit
ssh toto@10.11.200.233 -p 2222
ssh root@10.11.200.233 -p 2222
ssh-keygen
ssh-copy-id ~/.ssh/root root@10.11.200.233 -p 2222
```

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
