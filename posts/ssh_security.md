# Configurer et sécuriser un serveur SSH
<small>Date de publication: 30/12/2021</small>

Dans cet article je vais présenter comment configurer et sécuriser un serveur SSH. <br>

Il s'agit ici d'un serveur **Debian** (bullseye) en version **11.2**. <br>

Avoir un serveur SSH bien configuré et sécurisé est en général une base nécessaire afin de pouvoir par la suite administrer sereinement différents services. <br>

Bonne lecture.

## Mettre à jour le système

La mise à jour de la liste des paquets :

```sh
apt-get update
```

La mise à jour des paquets eux-mêmes :

```sh
apt-get upgrade
```

Cette opération est à effectuer régulièrement afin de conserver un système à jour.

### Automatiser les mises à jours de sécurité

Il est primordial d'effectuer les mises à jour de sécurité lorsque celles-ci sortent.  
Pour cela, nous allons utiliser le paquet `cron-apt`

```sh
sudo apt install cron-apt
```

Dans notre cas, on ne veut que les mises à jour de sécurité

```sh
grep security /etc/apt/sources.list > /etc/apt/security.sources.list
```

On configure `cron-apt` pour spécifier ce qu'il doit mettre à jour.  
Editer le fichier `/etc/cron-apt/config`

```txt
APTCOMMAND=/usr/bin/apt-get
OPTIONS="-o quiet=1 -o Dir::Etc::SourceList=/etc/apt/security.sources.list"
```

Indiquer à cron-apt d'installer les mises à jour lorsqu'il en trouve.

Editer le fichier `/etc/cron-apt/action.d/3-download`

```txt
dist-upgrade -y -o APT::Get::Show-Upgraded=true
```

`cron-apt` se lance par défaut toute les nuits à 4h du matin.

## SSH

### Informations

Si possible, privilégier le chiffrement `ed25519` plutôt que `RSA`.  

Il est préférable d'ajouter une passphrase lors de la génération de clé, en effet si le système hôte est compromis et qu'un attaquant via à disposer de notre clé il ne pourra l'utiliser sans le mot de passe correspondant.

```sh
ssh-keygen -t ed25519
```

Dans le cas ou uniquement `RSA` est disponible dans ce cas choisir une clé de `4096` bits.

```sh
ssh-keygen -t rsa -b 4096
```

Pour copier ensuite sa clé publique sur le serveur utiliser `ssh-copy-id`

```sh
ssh-copy-id username@remote_host
```

Si `ssh-copy-id` n'est pas disponible, alors utiliser la commande suivante

```sh
cat ~/.ssh/id_rsa.pub | ssh username@remote_host "cat - >> ~/.ssh/authorized_keys"
```

Lorsqu'un utilisateur non-root est ajouté au système il est préférable de lui attribuer une clé SSH différente de l'utilisateur root.

1. Se connecter en tant qu'utilisateur non-root

2. Créer un répertoire `.ssh` et veillez à lui attribuer les bons droits

```sh
mkdir .ssh
chmod 0700 .ssh
```

3. Se déplacer dans le répertoire `.ssh` et créer un fichier `authorized_keys` et lui attribuer les bons droits

```sh
touch authorized_keys
chmod 0600 authorized_keys
```

4. Pour finir, ajouter une clé publique au fichier `authorized_keys`. 

**Note**: Si les opérations précédentes ont été effectuées en tant qu'utilisateur `root` il faudra simplement changer le propriétaire du dossier `.ssh` et du fichier `authorized_keys` comme ceci

```sh
chown newUser:newUser .ssh
chown new:User:newUser .ssh/authorized_keys
```

Pour se connecter avec une clé spécifique

```sh
ssh -i ~/.ssh/id_ed25519 user@remote_host
```

**Note**: Parfois OpenSSH est un peu capricieux et si sur notre machine locale nous disposons d'une multitudes de clés SSH, même lorsqu'on spécifie avec l'option `-i` le chemin et la clé que nous souhaitons utiliser, il peut arriver d'obtenir cette erreur lors d'une connexion

```txt
Received disconnect from x.x.x.x : Too many authentication failures
```

Dans ce cas on pourra utiliser l'option `-o IdentitiesOnly=yes` lors de notre connexion

```sh
ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes user@remote_host -p 49506
```

### Configuration

Toutes les paramètres ci-dessous se font lors de l'édition du fichier `etc/ssh/sshd_config`.

1. Désactiver l'authentification par mot de passe

    ```txt
    PasswordAuthentication no
    ```

2. Activer l'authentification par clé publique 
 
    ```txt
    PubkeyAuthentication yes
    ```

3. Utiliser la version 2 du protocole SSH
    
    ```txt
    Protocol 2
    ```

4. Changer le port par défaut (22) du service SSH.
    
    Remplacez le numéro 22 par le numéro de port de notre choix.  
    N’entrez pas de numéro de port déjà utilisé sur votre système.   
    Pour des raisons de sécurité, utilisez un nombre compris entre `49152` et `65535`.

    ```txt
    Port 49506
    ```

    Ne pas oublier d'indiquer le nouveau port chaque fois que l'on établi une connexion SSH à notre serveur, par exemple:

    ```sh
    username@remote_host -p <PortNumber>
    ```

5. Désactivation de l’accès au serveur via l’utilisateur root

    ```txt
    PermitRootLogin no
    ```

6. Autoriser uniquement certains utilisateurs à se connecter

    ```txt
    AllowUsers login1 login2
    ```

7. Interdire certains utilisateurs à se connecter

    ```txt
    DenyUsers login1 login2
    ```

8. Modifier la durée d'ouverture de connexion sans être logué

    Ce paramètre défini la durée pendant laquelle une connexion sans être logué sera ouverte.  
    Si on avait gardé la bonne vieille technique du mot de passe, laisser 2 ou 3 minutes pour le taper, ce n'est pas de trop.  
    Mais vu que là, on utilise la clé, on sera logué immédiatement.

    ```txt
    LoginGraceTime 20s
    ```

9. Modifier le nombre de tentatives d'authentification

    Ce paramètre défini le nombre maximum d'essais avant de se faire jeter par le serveur.  
    Avec une clé, pas d'erreur possible, on peut le mettre à 1 essai possible sauf si on a plusieurs clés (donc un risque que ça échoue).

    Attention, si fail2ban est actif il peut surcharger ce paramètre. 

    ```txt
    MaxAuthTries 3
    ```

10. Activer la tracabilité via `Motd`

    - `PrintMotd` défini sur `yes` permettra d’afficher la bannière après connexion au serveur SSH.
    - `PrintLastLog` définie sur `yes` permet d’afficher dans la bannière la dernière connexion réussie. 

    ```txt
    PrintMotd yes
    PrintLastLog yes
    ```

11. Désactiver toutes les autres méthodes d'authentification

    ```txt
    UsePAM no
    KerberosAuthentication no
    GSSAPIAuthentication no
    PasswordAuthentication no
    ```

12. Modifier le nombre de connexions SSH simultanées en étant non authentifiées 

    Ce paramètre indique le nombre de connexions SSH non authentifiées que l'on peut lancer en même temps.  
    2 est suffisant sachant qu'avec les clés, c'est instantané.

    ```txt
    MaxStartups 2
    ```

13. Désactiver `X11Forwarding`

    Le X11Forwarding est un système permettant de renvoyer un retour graphique si notre serveur a un environnement graphique.  
    Notre serveur est probablement en interface CLI, alors il est recommandé de le désactiver.

    Il est également conseiller de désactiver l'affichage d'une interface graphique via SSH même en localhost via l'option X11UseLocalhost.

    ```txt
    X11Forwarding no
    X11UseLocalhost no
    ```

14. Désactiver la connexion via des comptes sans mots de passe

    ```txt
    PermitEmptyPasswords no 
    ```

15. Désactiver l'utilisation de PAM (optionnel)

    PAM = Pluggable Authentication Modules

    Ce système d'authentification revient à laisser une authentification par identifiant/mot de passe, la gestion de ceux-ci étant délégué à PAM qui ira lire dans /etc/passwd.

    L'authentification SSH par couple identifiant/mot de passe étant proscrite, UsePAM l'est tout autant.

    ```txt
    ChallengeResponseAuthentication no
    UsePAM no
    ```

16. Désactiver l'utilisation du TCP Forwarding et Agent Forwarding

    Si notre serveur n’a pas vocation à servir de rebond, aucun intéret d’utiliser un agent SSH, on désactive donc pour éviter la propagation de vos clefs SSH pour rien.

    Si notre serveur n’a pas vocation à servir de point d’entrée pour accéder à d’autres service, on bloque le TCP Forwarding pour éviter d'éventuelles attaques par rebonds.

    ```txt
    AllowAgentForwarding no
    AllowTcpForwarding no
    ```

17. Redémarrer le service `sshd`

    Pour redémarrer le service

    ```sh
    systemctl restart sshd
    ```

## System

1. Changer l'éditeur par défaut
    
    Par exemple, pour changer l'éditeur par défaut `nano` en `vim`

    ```sh
    sudo update-alternatives --config editor
    ```

2. Modification du mot de passe associé à l’utilisateur `root`

    Se connecter au compte `root`

    ```sh
    sudo su -
    ```

    Modifiez le mot de passe du compte root

    ```sh
    passwd
    New password:
    Retype new password:
    passwd: password updated successfully
    ```

3. Création d’un utilisateur avec des droits restreints

    En général, les tâches qui ne nécessitent pas de privilèges root doivent être effectuées via un utilisateur standard.

    - Créer un nouveau utilisateur avec un répertoire `/home`

        ```sh
        sudo useradd -m -d /home/login/ -s /bin/bash login
        ```

        `-m` : Pour la création du répertoire `home`  
        `-d` : Pour indiquer le `path` du répertoire `home`  
        `-s` : Pour indiquer le `shell` que l'on souhaite utiliser  

        Définir un password pour un utilisateur
        ```sh
        sudo passwd <new_user>
        ```

        Commande pour forcer l'utilisateur à changer de password à la 1ère connexion
        ```sh
        sudo passwd -e login
        ```

4. Utiliser la commande `sudo` avec **un seul** utilisateur **avec** le mot de passe demandé  
    
    Lien: [https://debian-facile.org/doc:systeme:sudo](https://debian-facile.org/doc:systeme:sudo)

    Passer obligatoirement par la commande `visudo` pour configurer `sudo`. L'utilitaire visudo vérifie la syntaxe du fichier `/etc/sudoers` avant d'enregistrer celui-ci.

    Commencer par installer `sudo` si la commande n'est pas disponible.  

    En tant que `root`
    ```sh
    apt-get update && apt-get install sudo
    ``` 

    Puis lancer la commande `visudo`
    ```sh
    visudo
    ```

    Sous la ligne :
    ```txt
    root    ALL=(ALL:ALL) ALL
    ```

    écrire:
    ```txt
    login   ALL=(ALL:ALL) ALL
    ```

    (`login` illustre ici notre nom d'utilisateur)


5. Limitez les services exécutés sur la machine 

    Limiter les services exécutés sur la machine à ceux utilisés et dont-on a besoin.  

    Lister les services en cours d'éxécutions

    ```sh
    service --status-all
    ```

    Lister tous les packages installés sur la machine

    ```sh
    dpkg --get-selections | grep -v deinstall
    ```

    OU
    
    ```sh
    apt list --installed
    ```

    Par exemple, si on n'utilise pas les services `portmap`, `nfs` (dans le cas d'un serveur web on en a pas besoin)

    ```sh
    /etc/init.d/portmap stop
    /etc/init.d/nfs-common stop
    update-rc.d -f portmap remove
    update-rc.d -f nfs-common remove
    apt-get remove portmap
    ```

6. Installer et configurer UFW

    ```sh
    sudo apt-get install ufw
    ```

    Obtenir le statut de UFW et lister les règles actives

    ```sh
    sudo ufw status
    ```

    On peut utiliser l'option `numbered` en plus de `status` afin de numéroter les différentes règles actives afin de pouvoir les manipuler plus facilement.

    On peut également utiliser l'option `verbose` afin d'obtenir plus d'informations lors de l'affichage des règles actives.

    Voir les règles utilisées par défaut (origine)
    
    ```sh
    sudo vim /etc/default/ufw
    ```

    On peut passer l'option `IPV6` à `no` si on ne souhaite pas l'utiliser.  
    On rechargera ensuite la configuration avec la commande `ufw reload`.

    Remettre la configuration d'origine

    ```sh
    sudo ufw reset
    ```

    Recharger la configuration après modification des fichiers de configuration

    ```sh
    sudo ufw reload
    ```

    Autoriser la connexion à SSH via un port spécifique sur le protocole TCP ET UDP

    ```sh
    sudo ufw allow 49506
    ```

    Autoriser la connexion à SSH via un port spécifique sur le protocole TCP uniquement

    ```sh
    sudo ufw allow 49506/tcp
    ```

    Activer UFW

    ```sh
    sudo ufw enable
    ```

7. Installer et configurer Fail2Ban

    ```sh
    sudo apt-get install fail2ban
    ```

    Le fichier de configuration de fail2ban est `/etc/fail2ban/jail.conf`.  
    Ce fichier est écrasé lors des mises à jour de fail2ban, il faut donc en faire une copie afin de voir sa configuration supprimée.

    ```sh
    sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
    ```

    Pour modifier la configuration il faut donc éditer le fichier `/etc/fail2ban/jail.local`.

    ```sh
    sudo vim /etc/fail2ban/jail.local
    ```

    Une fois les règles paramétrées on peut redémarrer fail2ban

    ```sh
    sudo service fail2ban restart
    ```

    Configurer fail2ban pour sécuriser la connexion SSH

    Editer le fichier /etc/fail2ban/jail.local

    ```txt
    [sshd]

    enabled = true
    port = 49506
    filter = sshd
    logpath = /var/log/auth.log
    maxretry = 3
    bantime = 86400
    ```

    Lister tout les commandes du Client fail2ban.
    
    ```sh
    fail2ban-client -h
    ```

    Obtenir des informations générales

    ```sh
    fail2ban-client status
    ```
    
    Obtenir des informations sur une prison

    ```sh
    fail2ban-client status sshd
    ```
    
    Bannir une ip

    ```sh
    sudo fail2ban-client set sshd banip XXX.XXX.XXX.XXX
    ```

    Débannir une ip d'une prison

    ```sh
    fail2ban-client set ssh unbanip XXX.XXX.XXX.XXX
    ```

    Vérification en temps réel logs d'authentification du Serveur.
    
    ```sh
    tail -f /var/log/auth.log
    ```
