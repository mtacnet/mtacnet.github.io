# Configurer et sécuriser un serveur SSH
<small>Date de publication: 30/12/2021</small> <br>
<small>Mis à jour: 01/01/2022</small>

Dans cet article, je vais vous présenter comment configurer et sécuriser un serveur SSH.

C'est en général une base nécessaire pour pouvoir, par la suite, administrer sereinement différents services.

Dans un premier temps, j'aborde brièvement les mises à jour afin d'avoir, constamment, un système bénéficiant des derniers patchs de sécurité.

Après un petit rappel sur la gestion des clés SSH, je procède ensuite au durcicement du fichier de configuration du serveur SSH grâce aux options à disposition.

Pour finir, une **partie optionnelle**, où j'ai fais le choix de présenter de façon simple, 2 outils éprouvés et couramment utilisés dans le cadre de la sécurisation d'un serveur, à savoir: UFW et Fail2Ban.

UFW et Fail2Ban sont des outils complets et puissants, mais nous allons nous concentrer ici uniquement sur leur mise en place dans le cadre d'un serveur SSH. Sachez simplement qu'ils ne se limitent pas à ce périmètre.
Ils feront probablement l'objet d'un article plus poussé prochainement.

Les diverses opérations sont réalisées depuis un compte disposant des privilèges administrateur sur un serveur **Debian** en version **11.2** (Bullseye).

Voici ce que nous verrons dans cet article:
- [Mise à jour du système](#mettre-à-jour-le-système)
- [Automatiser les mises à jour de sécurité](#automatiser-les-mises-à-jour-de-sécurité)
- [Générer et utiliser des clés SSH](#générer-et-utiliser-des-clés-ssh)
- [Configuration du serveur SSH](#configuration-du-serveur-ssh)
- [Installation et configuration du firewall UFW](#installation-et-configuration-du-firewall-ufw)
- [Installation et configuration de Fail2Ban](#installation-et-configuration-de-fail2ban)

Bonne lecture.

## Mettre à jour le système

Mise à jour de la liste des paquets

```sh
apt update
```

Mise à jour des paquets eux-mêmes

```sh
apt upgrade
```

Cette opération est à effectuer régulièrement afin de conserver un système à jour.

## Automatiser les mises à jour de sécurité

Il est primordial d'effectuer les mises à jour de sécurité lorsque celles-ci sortent.  
Pour cela, nous allons utiliser le paquet `cron-apt`

```sh
apt install cron-apt
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

## Générer et utiliser des clés SSH

Si possible, privilégier le chiffrement `ed25519` plutôt que `RSA`.  

Il est préférable d'ajouter une passphrase lors de la génération de clé, en effet si le système hôte est compromis et qu'un attaquant vient à disposer de notre clé il ne pourra pas l'utiliser sans le mot de passe correspondant.

```sh
ssh-keygen -t ed25519
```

Dans le cas ou uniquement `RSA` est disponible, alors choisir une clé de `4096` bits.

```sh
ssh-keygen -t rsa -b 4096
```

Pour copier ensuite sa clé publique sur le serveur, utiliser `ssh-copy-id`

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
chown -R newUser: .ssh
```

Pour se connecter avec une clé spécifique

```sh
ssh -i ~/.ssh/id_ed25519 username@remote_host
```

**Note**: Parfois OpenSSH est un peu capricieux. En effet, il tente de s'authentifier avec toutes les clés SSH dont il dispose l'une après l'autre, jusqu'à atteindre le nombre maximum de tentatives autorisées, et ce même lorsqu'on spécifie avec l'option `-i` le chemin et la clé que nous souhaitons utiliser.

Donc, si sur notre machine locale nous disposons d'une multitudes de clés SSH, il peut arriver d'obtenir cette erreur lors d'une tentative de connexion

```txt
Received disconnect from x.x.x.x : Too many authentication failures
```

Dans ce cas on pourra utiliser l'option `-o IdentitiesOnly=yes` lors de notre connexion pour pallier ce problème

```sh
ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes username@remote_host
```

**Notes**:
1. Sachez qu'il existe des gestionnaires de clés SSH si vous ne souhaitez pas vous "embêter" avec des lignes de commandes à rallonge.

2. Depuis la version 5.3 de OpenSSH, il existe une fonctionnalité qui permet d'utiliser un CA (Certificate Authority) afin de faciliter la gestion de multiples clés SSH sur différentes machines. 

## Configuration du serveur SSH

Tous les paramètres, ci-dessous, se font lors de l'édition du fichier `etc/ssh/sshd_config`.

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
    
    Il est conseillé de remplacer le numéro 22 par le numéro de port de notre choix.  
    En effet, la majeure partie des attaques automatisées (botnets, scripts etc.) se font sur le port par défaut qui est le port 22.
    Veillez à ne pas entrer un numéro de port déjà utilisé sur votre système.   
    Pour des raisons de sécurité, il est recommandé d'utiliser un nombre compris entre `49152` et `65535`.

    ```txt
    Port 49506
    ```

    Ne pas oublier d'indiquer le nouveau port chaque fois que l'on établi une connexion SSH à notre serveur, par exemple:

    ```sh
    username@remote_host -p 49506
    ```

5. Désactivation de l’accès au serveur via l’utilisateur root

    ```txt
    PermitRootLogin no
    ```

6. Autoriser uniquement certains utilisateurs à se connecter

    ```txt
    AllowUsers username1 username2
    ```

7. Interdire certains utilisateurs à se connecter

    ```txt
    DenyUsers username3 username4
    ```

8. Modifier la durée d'ouverture de connexion sans être logué

    Ce paramètre définit la durée pendant laquelle une connexion, sans être logué, sera ouverte.  
    Si on avait gardé la bonne vieille technique du mot de passe, laisser 2 ou 3 minutes pour le taper ne serait pas de trop.  
    Mais vu que là, on utilise une clé, on sera logué immédiatement.

    ```txt
    LoginGraceTime 20s
    ```

9. Modifier le nombre de tentatives d'authentification

    Ce paramètre définit le nombre maximum d'essais avant de se faire jeter par le serveur.  
    Avec une clé, pas d'erreur possible, on peut le mettre à 1 essai possible sauf si on a plusieurs clés (donc un risque que ça échoue).

    Attention, si fail2ban est actif il peut surcharger ce paramètre. 

    ```txt
    MaxAuthTries 3
    ```

10. Activer la tracabilité via `Motd`

    - `PrintMotd` défini sur `yes` permet d’afficher la bannière après connexion au serveur SSH.
    - `PrintLastLog` défini sur `yes` permet d’afficher dans la bannière la dernière connexion réussie. 

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

12. Modifier le nombre de connexions SSH simultanées en étant non authentifié 

    Ce paramètre indique le nombre de connexions SSH non authentifiées que l'on peut lancer en même temps.  
    2 est suffisant sachant qu'avec les clés, c'est instantané.

    ```txt
    MaxStartups 2
    ```

13. Désactiver X11Forwarding

    Le `X11Forwarding` est un système permettant de renvoyer un retour graphique si notre serveur a un environnement graphique.  
    Notre serveur est probablement en interface CLI, alors il est recommandé de le désactiver.

    Il est également conseillé de désactiver l'affichage d'une interface graphique via SSH même en localhost via l'option `X11UseLocalhost`.

    ```txt
    X11Forwarding no
    X11UseLocalhost no
    ```

14. Désactiver la connexion via des comptes sans mots de passe

    ```txt
    PermitEmptyPasswords no 
    ```

15. Désactiver l'utilisation de PAM

    PAM = Pluggable Authentication Modules

    Ce système d'authentification revient à laisser une authentification par identifiant/mot de passe, la gestion de ceux-ci étant déléguée à PAM qui ira lire dans /etc/passwd.

    L'authentification SSH par couple identifiant/mot de passe étant proscrite, UsePAM l'est tout autant.

    ```txt
    ChallengeResponseAuthentication no
    UsePAM no
    ```

16. Désactiver l'utilisation du TCP Forwarding et Agent Forwarding

    Si notre serveur n’a pas vocation à servir de rebond, aucun intéret d’utiliser un agent SSH. On le désactive donc l'`AgentForwarding` pour éviter la propagation de nos clefs SSH pour rien.

    Si notre serveur n’a pas vocation à servir de point d’entrée pour accéder à d’autres service, on bloque le `TCP Forwarding` pour éviter d'éventuelles attaques par rebonds.

    ```txt
    AllowAgentForwarding no
    AllowTcpForwarding no
    ```

17. Redémarrer le service `sshd`

    Pour redémarrer le service

    ```sh
    systemctl restart sshd
    ```

## Installation et configuration du firewall UFW

UFW permet de simplifier la gestion des règles du pare-feu, offrant ainsi une bonne alternative à Iptables qui peut parfois avoir une syntaxe assez complexe pour la gestion des flux de connexion. 

On installe UFW

```sh
apt install ufw
```

UFW ne se démarre pas par défaut pour ne pas nous "enfermer" dehors. Avant de l'activer, on va commencer par gérer les politiques d'accès par défaut.

Autoriser la connexion à SSH via un port spécifique sur le protocole TCP uniquement

```sh
ufw allow 49506/tcp
```

OU

Autoriser la connexion à SSH via un port spécifique sur le protocole TCP ET UDP

```sh
ufw allow 49506
```

Obtenir le statut de UFW et lister les règles actives

```sh
ufw status
```

On peut utiliser l'option `numbered` en plus de `status` pour numéroter les différentes règles actives afin de pouvoir les manipuler plus facilement.

On peut également utiliser l'option `verbose` afin d'obtenir plus d'informations lors de l'affichage des règles actives.

Voir les règles utilisées par défaut (origine)

```sh
vim /etc/default/ufw
```

On peut passer l'option `IPV6` à `no` si on ne souhaite pas l'utiliser.  
On rechargera ensuite la configuration avec la commande `ufw reload`.

Pour recharger la configuration après modification des fichiers de configuration

```sh
ufw reload
```

Activer UFW

```sh
ufw enable
```

Pour remettre la configuration d'origine

```sh
ufw reset
```

## Installation et configuration de Fail2Ban

Fail2Ban est une application qui analyse les logs de divers services et va chercher des tentatives répétées de connexions infructueuses. Il va procéder à un bannissement en ajoutant une règle au firewall pour bannir l'adresse IP de la source.

Personnellement, je trouve qu'il est complémentaire à UFW et permet une gestion assez "fine" des règles de banissement.

On commence par installer Fail2Ban

```sh
apt install fail2ban
```

Le fichier de configuration de fail2ban est `/etc/fail2ban/jail.conf`.  
Ce fichier est écrasé lors des mises à jour de fail2ban, il faut donc en faire une copie afin de ne pas voir sa configuration supprimée.

```sh
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Pour modifier la configuration, il faut donc éditer le fichier `/etc/fail2ban/jail.local`.

```sh
vim /etc/fail2ban/jail.local
```

Configurer fail2ban pour sécuriser la connexion SSH

Editer le fichier `/etc/fail2ban/jail.local` et se rendre dans la partie `[sshd]`

```txt
[sshd]

enabled = true
port = 49506
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
```

- `enabled`: Active la protection (jail) sur le service sshd
- `port`: Spécifie le port utilisé par notre serveur
- `logpath`: Emplacement des fichiers de logs à surveiller
- `bantime`: 86400 secondes équivalent à 24h
- `maxretry`: Une IP sera bannie après 3 tentatives de connexion avortées

Par la suite, on peut évidemment adapter tous ces paramètres et en ajouter/supprimer selon nos besoins.

Une fois les règles paramétrées, on peut redémarrer fail2ban

```sh
systemctl restart fail2ban
```

Lister toutes les commandes du Client Fail2Ban.

```sh
fail2ban-client -h
```

Obtenir des informations générales

```sh
fail2ban-client status
```

Obtenir des informations sur une prison spécifique

```sh
fail2ban-client status sshd
```

Bannir une ip

```sh
fail2ban-client set sshd banip XXX.XXX.XXX.XXX
```

Débannir une ip d'une prison

```sh
fail2ban-client set ssh unbanip XXX.XXX.XXX.XXX
```

Vérification en temps réel des logs d'authentification du serveur.

```sh
tail -f /var/log/auth.log
```
