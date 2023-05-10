# Configuration Linux Keychron K4

<small>Date de publication: 10/05/2023</small> <br>

Cet article présente comment configurer un **clavier Keychron K4** sous Linux.

Nous verrons comment activer et utiliser les touches de fonction mais également les accents et symboles Français à partir d'un `layout US ANSI`.

La configuration à été réalisée sur un système **Debian 11.7** (Bullseye)

## Accents et symboles FR

J'ai personnellement testé différentes `Sources de saisie` depuis les paramètres de configuration de l'OS.

Après plusieurs tentatives, la configuration retenue est `Anglais (Macintosh)` car elle me permet d'utiliser les commandes suivantes afin de réaliser "facilement" les divers accents et symboles Français.

![Anglais (Macintosh)](../assets/images/Anglais_Macintosh.png "Anglais (Macintosh)")

Voici les raccourcis clavier que j'utilise au quotidien (on s'y fait vite):

- `CMD (right) + <c>` => `ç`
- `CMD (right) + <e>` => `é`
- `CMD (right) + <backtick> puis <e>` => `è`
- `CMD (right) + <i> puis <e>` => `ê`
- `CMD (right) + <e> puis <SHIFT> + <E>` => `É`
- `CMD (right) + <backtick> puis <SHIFT> + <E>` => `È`
- `CMD (right) + <i> puis <SHIFT> + <E>` => `Ê`
- `CMD (right) + <backtick> puis <a>` => `à`
- `CMD (right) + <i> puis <a>` => `â`
- `CMD (right) + <backtick> puis <SHIFT> + <a>` => `À`
- `CMD (right) + <SHIFT> + 2` => `€`

## Touches de fonction

Malheureusement, sur Linux, le `Keychron K4` n'enregistre par défaut aucune des touches de fonction `F1-F12` en tant que touches de fonction réelles. À la place, il les traite comme des touches multimédias (augmenter/diminuer la luminosité, le son, etc.).

Le Keychron K4 dispose de `2 modes`, `Windows/Android` et `MacOS/iOS` et aucun d'eux ne semble fonctionner correctement.

Afin de résoudre ce problème, voici les étapes à suivre:

1. Régler le clavier en mode `Windows/Android` via le bouton latéral
2. Utiliser `fn + X + L`(maintenir pendant 4 secondes) pour régler la rangée de touches de fonction sur le mode "Fonction".
3. Lancer la commande suivante:

    ```sh
    echo 0 | sudo tee /sys/module/hid_apple/parameters/fnmode
    ```

Désormais, les touches `F1-F12` fonctionnent correctement. En maintenant la touche `fn` cela transforme les touches de fonction en touches multimédia.
Exemple:

- `fn + F2` => Baisser la luminosité
- `F2` => Renommer un dossier/fichier

Pour conserver ce changement après redémarrage, il faut ajouter une option de module pour `hid_apple` (par défaut, sur `Debian 11`, le fichier n'existe pas, il sera automatiquement créé par la commande suivante)

```sh
echo "options hid_apple fnmode=0" | sudo tee -a /etc/modprobe.d/hid_apple.conf
```

Il faudra également mettre à jour l'`initramfs` afin d'inclure le nouveau module:

```sh
sudo update-initramfs -u
```

Enjoy 😉
