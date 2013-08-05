Date: 28/03/2013
Title: Une piratebox avec un raspberry pi
Slug: piratebox-raspberry-pi

Ce texte décrit succinctement la mise en place d'une piratebox. Comprendre par là :

- un dispositif auquel peuvent se connecter des gens en Wifi (de préférence sans mot de passe, ou alors faut le leur donner)
- idéalement, ils savent pas où il est physiquement, comme ça ils le cassent pas :)
- bref, ils s'y connectent, et maintenant, quelle que soit la page web qu'ils essaient de charger, ils sont redirigés vers **la page spéciale**.
- la page spéciale présente une tête de mort.
- la page spéciale leur permet principalement de discuter et échanger des fichiers.
- mais pourquoi pas, d'autres trucs (jeux, proxy web..).

Matériel
========

Ça serait cool d'avoir un Raspberry Pi pour ça, tiens. Moi je l'ai fait sur mon pc. D'ailleurs tout sera décrit avec ubuntu 12.04.

Niveau dongle wifi, ma vieille D-Link DWA-110 convient à merveille.
Sinon c'est pas très cher, [environ 15€](http://www.materiel.net/cle-usb-wifi/).

Encore faut-il qu'elle soit compatible GNU/Linux (notamment en mode AP. Et en mode monitor au cas-où..)

Bref, on insère le dongle wifi dans le port usb, et c'est tout pour la partie matériel.

Logiciel
========

Parlons réseau.
Votre Raspberry Pi (que j'abrégerai Rpi) était jusque là un simple poste connecté à votre box par un câble Ethernet.
Ou en Wifi. Ou connecté à rien du tout, on s'en fiche, c'est encore plus simple. Mais disons que là, il dispose d'une interface `eth0` qu'il utilise et à laquelle on va rien toucher.
Bien.

Le simple fait d'avoir pluggé le dongle wifi devrait avoir rajouté une interface, `wlan0` dans mon cas.
Si on ne la voit pas avec un `ifconfig`, peut être qu'on la verra avec un `iwconfig`.

Sommaire :

- configurer Rpi en point d'accès wifi, grâce à **hostapd**
- configurer Rpi en serveur DHCP pour attribuer [adresse ip / dns / passerelle par défaut] aux clients, grâce à **isc-dhcp-server**
- rediriger toutes les pages web des clients vers la page spéciale, grâce à **fakedns.py**
- mettre en place la page spéciale, grâce à **nginx**

Nous allons toucher à pas mal de fichiers de conf. Pas des trucs critiques, mais faites des backups quand même.
La plupart des actions s'effectuent en tant que `root`.

HostAPD
-------

`hostapd` est le programme que l'on va installer sur Rpi, et qui va utiliser notre tout nouveau `wlan0` pour créer un point d'accès wifi que les gens verront dans leur gestionnaire de wifis, et auquel ils pourront se connecter.

**Prérequis n°1** : il faut que votre dongle wifi supporte le mode `AP`. Cela se vérifie avec la commande `iw list`. Si vous n'avez pas la commande `iw` :

    $ sudo apt-get install iw

`iw list` renvoie beaucoup de charabia, tout ce qui nous intéresse est que dans la partie "Supported interface modes", il y ai `AP`. Chez moi ça donne :

	Wiphy phy0
        Band 1:
            Frequencies:
                [...]
            Bitrates (non-HT):
                [...]
        max # scan SSIDs: 4
        max scan IEs length: 2285 bytes
        Coverage class: 0 (up to 0m)
        Supported Ciphers:
            [...]
        Available Antennas: TX 0 RX 0
        Supported interface modes:
             * IBSS
             * managed
             * AP
             * AP/VLAN
             * WDS
             * monitor
             * mesh point
        software interface modes (can always be added):
             * AP/VLAN
             * monitor
        interface combinations are not supported
        Supported commands:
             [...]
             [...]
             ...

(Évidemment j'ai pas tout mis) J'ai `AP` : c'est bon, je peux donc installer `hostapd`. D'après le tuto que j'avais sous la main, il fallait l'installer à partir des sources, mais chez moi ça marche tout aussi bien avec un bon vieux :

	$ sudo apt-get install hostapd
    
`hostapd` se lance avec un seul paramètre : un fichier de conf.
Voila le mien (dans `/etc/hostapd/piratebox.conf`) :

    interface=wlan0       # interface wlan du Wi-Fi
    driver=nl80211        # nl80211 avec tous les drivers Linux mac80211
    ssid=piratebox        # Nom du spot Wi-Fi
    hw_mode=g             # mode Wi-Fi (g = IEEE 802.11g)
    channel=6             # canal de fréquence Wi-Fi (1-14)
    auth_algs=1           # Wi-Fi ouvert, pas d'authentification !
    beacon_int=100        # Beacon interval in kus (1.024 ms)
    dtim_period=2         # DTIM (delivery trafic information message)
    max_num_sta=255       # Maximum number of stations allowed in station table
    rts_threshold=2347    # RTS/CTS threshold; 2347 = disabled (default)
    fragm_threshold=2346  # Fragmentation threshold; 2346 = disabled (default)

Ici je n'exploite pas toutes les options de configurations de `hostapd`, mais pour ce cas d'usage, ça devrait suffire. Pour le driver, j'ai rien touché : si ça marche pas chez vous, je serai ravi de savoir pourquoi et ce qu'il faut mettre à la place de `nl80211` :). Bref, voilà pour `hostapd`. Pour le démarrer :


    :::bash
	$ sudo hostapd /etc/hostapd/piratebox.conf
    [sudo] password for fred: 
    Configuration file: /etc/hostapd/piratebox.conf
    Using interface wlan0 with hwaddr 1c:af:f8:02:36:7a and ssid 'piratebox'

Vous pouvez déjà essayer de vous y connecter depuis un pc ou un smartphone : le réseau wifi "piratebox" apparaît bien dans la liste, la connexion se déroule bien, mais impossible de récupérer une adresse IP en dhcp. C'est l'objet de la partie suivante.

([Source](http://doc.ubuntu-fr.org/hostapd))

isc-dhcp-server
---------------

(autre fois appelé dhcp3-server)
L'objectif de cette section est d'avoir un serveur dhcp fonctionnel sur le Rpi, qui va faire en sorte que, à chaque fois que des clients vont se connecter au point d'accès sur `wlan0`, on leur attribuera une adresse IP avec un masque de sous réseau, on leur dira quelles sont les adresses de leur passerelle par défaut, de leurs serveurs dns, quelle est l'adresse de broadcast..

Dans mon cas, je considère que :

- le Rpi a l'adresse IP `192.168.37.254` (netmask `255.255.255.0`) sur son interface `wlan0`. 

Pour faire ça :

	$ sudo ifconfig wlan0 192.168.37.254 netmask 255.255.255.0 up

- les clients auront une adresse comprise entre `192.168.37.10` et `192.168.37.50`. Il y en aura donc 40 max.
- La passerelle par défaut et le serveur DNS seront Rpi.
- L'adresse de broadcast sera `192.168.37.255`

### Installation

	$ sudo apt-get install isc-dhcp-server

### Configuration

Là, il va y avoir deux fichiers à éditer.
Le premier, c'est super simple, il s'agit de `/etc/default/isc-dhcp-server`. Chez moi, il ne contient qu'une seule ligne qui ne soit pas commentée :

  INTERFACES="wlan0"

Cette ligne indique à notre serveur dhcp qu'il oeuvrera pour les clients de `wlan0` uniquement.
Ensuite, il y a le fichier /etc/dhcp/dhcpd.conf. Faites en un backup au cas où, et mettez y :

	## Conf trouvée sur ubuntu-fr.org ## 
    default-lease-time 3600; # bail d'une heure
    max-lease-time 86400; # bail d'un jour max
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.37.255;
    option routers 192.168.37.254; # passerelle = Rpi
    option domain-name-servers 192.168.37.254; # dns = Rpi
    
    subnet 192.168.37.0 netmask 255.255.255.0 {
      range 192.168.37.10 192.168.37.50;  # max 40 clients
    }

Sauvegardez, et (re)lancez le serveur dhcp :

	$ sudo service isc-dhcp-server restart
    
Testez à nouveau de vous connecter depuis un autre pc : ça devrait marcher ! (un `ping 192.168.37.254` aussi)

([Source](http://doc.ubuntu-fr.org/dhcp3-server))

fakedns.py
----------

Maintenant, faisons en sorte que quand les gens utilisent leur navigateur, quelle que soit l'adresse, ils soient redirigés vers la même page. C'est la même technique qui est utilisée par les hotspots freewifi ou sfr wifi pour afficher la page de login.

Ce qu'on va faire, c'est installer un serveur DNS menteur sur Rpi : quand les clients vont vouloir connaître l'IP associée à un nom de domaine, on leur répondra **systématiquement** `192.168.37.254`, l'adresse IP du Rpi.

fakedns.py est un script python qui fera parfaitement le job : [source](http://code.activestate.com/recipes/491264-mini-fake-dns-server/)

Pour l'installer et le lancer :

	$ wget http://fspot.org/files/fakedns.py
    $ sudo python fakedns.py

Et hop.

nginx
-----

Cette partie peut vous être inutile si vous utilisez apache ou un autre serveur http directement sans frontend (genre node.js, unicorn, gunicorn...).

Pour ma part, le site web affiché est propulsé par gunicorn, nginx me sert donc uniquement de reverse proxy. Une config minimale serait donc d'ajouter ceci à `/etc/nginx/sites-available/default` :

	server {
      listen 80;
      server_name 192.168.37.254;
  
      # piratebox :
      location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8337;
      }
	}

(j'ai également une section similaire pour le https)

Ensuite, on relance nginx avec :

	$ sudo service nginx restart

Fin
===

Voilà, à vous de vous amuser en proposant un site original aux visiteurs (ou de faire les fourbes en imitant facebook avec un ssid type "freewifi" pour voler leurs mots de passes). A vous aussi de faire un boitier qui déchire, auto-alimenté, à emmener partout :). Et à vous d'automatiser toutes les procédures décrites ici au boot du Rpi.

Ce boîtier pourra également vous être utile pour vous faire un petit réseau sans fil, pour une LAN par exemple.
Mais j'aborderai probablement dans un autre article les configurations nécessaires pour transformer ce boîtier en un hotspot wifi permettant d'accéder à Internet. Le DNS menteur ne fera donc plus partie du jeu ;).