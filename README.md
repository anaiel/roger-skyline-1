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
    [](https://fr-wiki.ikoula.com/fr/Se_prot%C3%A9ger_contre_le_scan_de_ports_avec_portsentry)

    [→ Tuto nmap - scan nmap des ports ouverts](https://www.memoinfo.fr/tutoriels-linux/tuto-nmap-scaner-les-ports-ouverts/)

    - pour tester : `nmap -Pn 10.11.200.247`
    - 

    Pas bien sur que ça ait fonctionné... comment tester ?

- **Arrêter les services inutiles**
    - `service <service> stop` : apparmor (secu), console-setup, hwclock.sh, keyboard-setup.sh
    - pas touche a dbus (permet communication entre processus), cron, fail2ban, networking, portsentry
    - Pour voir la liste des services + description : `systemctl list-units | grep service`

    Je suis toujours vraiment pas sur de moi

- **Scripts**
    - `apt install mailutils`
    - `crontab -e` pour creer la crontab
    - ajouter les règles dedans

Today 

- Apparemment le probleme de fonctionnement de portsentry vient du fait que si la policy par defaut est DROP pour incoming il ne surveille pas les ports
    - test en mettant la policy par defaut allow et en effet nmap semble coincé (yep l'IP est bannie) (pour retirer l'adresse bannie `iptables -D INPUT 1` (ou 1 correspond a la position de la règle à retirer)
    - soluce : avec un seul port ouvert, portsentry ne fonctionne pas : il faut donc ouvrir les port http et https (qui ne sont ps nécessaires suivant la question du firewall pour la partie obligatoire mais le sont pour la partie optionnelle

        ufw allow http
        ufw allow https
        service portsentry restart
