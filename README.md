# roger-skyline-1

## Initialisation de la VM

OS: Debian 10
Hyperviseur: VirtualBox

- Lancer la machine et suivre l'installation
    - hostname: roger
    - domain name: (laisser vide)
    - users : root (toor), anleclab (anleclab)
    - Partitionning automatique, supprimer la partition existante, dans le FREE SPACE créer une partition de taille 4.2G
    - proxy ⇒ leave blank
    - only install SSH and standard tools

- Installer sudo :
```
su root
apt install sudo
nano /etc/sudoers #Rajouter la ligne anleclab (ALL:ALL) NOPASSWD:ALL
exit
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
