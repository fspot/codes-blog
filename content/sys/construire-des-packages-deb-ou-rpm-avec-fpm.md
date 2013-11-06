Date: 06/11/2013
Title: Construire des packages .deb ou .rpm avec FPM
Slug: construire-des-packages-deb-ou-rpm-avec-fpm

Quand on cherche à distribuer son programme, on peut tout à fait se contenter de dire aux utilisateurs de faire un git clone, configure, make, make install, éditer un fichier de conf, lancer un service...  
Mais dans un contexte professionnel, les packages natifs présentent beaucoup d'intérêts pour les mises en production : format bien connu, panoplie d'outils associés, fichier unique, gestion des dépendances, facilité d'installation, désinstallation et rollbacks. Les outils de gestion de configuration (Chef, Puppet, Salt, Ansible..) savent gérer ces packages et se chargent de leur déploiement et de leur configuration.

FPM est un outil très simple qui permet de créer des packages .deb ou .rpm pour ses programmes.

En réalité il va plus loin que cela, il peut générer des packages :

 - deb
 - rpm
 - solaris
 - pkg (mac os x)
 - tar

Pour cela il peut se baser sur différents formats d'entrée. Vous pouvez lui passer des projets contenus dans un simple dossier (le plus simple), mais également des :

 - packages deb
 - packages rpm
 - gem (packages ruby)
 - packages python
 - pear (packages php)
 - npm (packages node)

Cet article va présenter l'utilisation basique de FPM dans le cadre d'un micro-projet.

Installation
------------

Sous Debian :

    $ sudo apt-get install rubygems
    $ sudo gem install fpm
    $ sudo apt-get install rpm  # pour pouvoir générer des packages .rpm

Sous CentOS :

    $ sudo yum install rubygems
    $ sudo gem install fpm
    $ sudo yum install rpm-build  # pour pouvoir générer des packages .rpm


Utilisation
-----------

Imaginons un premier projet basique, où l'on voudrait créer un package pour un unique binaire. Prenons le hello world standard :

    :::C
    #include <stdio.h>   
    int main(void) {
        puts("Hello, world!");
        return 0;
    }

Générons le binaire `myhellow` :

    $ gcc -o myhellow myhellow.c

À partir de ce binaire, et grâce à fpm, il est trivial de générer un package debian :

    $ fpm -s dir -t deb -n myhellow -v 1.0 ./myhellow=/usr/local/bin/myhellow

Explications des arguments passés :

 - `-s <type_input>` : pour dire si on passe en entrée un package deb, rpm, gem, npm... Ici on n'a rien de tout ça : on choisit `dir`
 - `-t <type_output>` : le format du package généré. Là on a choisi `deb`, mais on aurait pu choisir `rpm`, `solaris`...
 - `-n <package_name>` : le nom du package (qu'on retrouve habituellement en faisant `apt-get install <package_name>`)
 - `-v <version>` : la version du package
 - `./myhellow=/usr/local/bin/myhellow` : de la forme `<source>=<destination>`, on peut indiquer plusieurs clés/valeurs ainsi. Cela indique où seront installé ces fichiers ou dossiers lorsqu'on tentera d'installer le package final. Là, notre unique binaire se retrouvera dans `/usr/local/bin/`

Tous ces arguments ne sont pas obligatoires, certains ont des valeurs par défaut.

Cette commande génère chez moi le fichier `myhellow_1.0_amd64.deb`. On y retrouve le nom du package, la version, ainsi que l'architecture de ma machine (en effet ce package ne fonctionnera pas sur une architecture 32 bits).

Beaucoup d'autres arguments peuvent être passés à fpm et feront peut être l'objet d'autres articles. Pour les découvrir : `fpm -h`. Notamment, on peut préciser des scripts à exécuter avant ou après l'installation ou la désinstallation du package, les dépendances du package, etc.

Utilisation du package créé
---------------------------

Pour un package deb :

    $ sudo dpkg -i package.deb  # installation
    $ sudo gdebi package.deb    # installe le paquet et ses dépendances (nécessite gdebi)
    $ sudo dpkg -r package      # désinstallation (on indique le NOM du package et non le fichier)
    $ sudo apt-get remove package  # autre façon de désinstaller

Pour un package rpm :

    $ sudo rpm -Uvh package.rpm  # installation
    $ sudo yum localinstall package.rpm  # installation + dépendances
    $ sudo rpm -e package  # désinstallation
    $ sudo yum remove package  # autre désinstallation

Dans un prochain article éventuel, je présenterai une manière de packager un projet python complet (avec son virtualenv).

