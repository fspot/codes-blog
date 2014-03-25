Date: 25/03/2014
Title: Redirection de ports sur sa box en Python
Slug: redirection-de-ports-sur-sa-box-en-python

UPnP-IGD
========

Quand on réalise un logiciel de peer-to-peer, ou même un logiciel client-serveur type jeu multijoueur ou logiciel d'aide à distance type Teamviewer, et qu'on souhaite que l'utilisateur final puisse **facilement** l'utiliser, on se heurte à un problème : en général, le serveur sera innaccessible, car situé derrière une box ou un pare-feu.

La solution à laquelle on pense en premier est de mettre en place une redirection de ports.
L'utilisateur peut faire cela manuellement, mais très peu en sont capables. Heureusement, il existe l'[UPnP](https://en.wikipedia.org/wiki/Universal_Plug_and_Play) et son protocole IGD ([Internet Gateway Device Protocol](https://en.wikipedia.org/wiki/Internet_Gateway_Device_Protocol)) pour réaliser cette redirection automatiquement depuis le logiciel.
Cela peut potentiellement représenter une faille de sécurité, mais c'est cependant activé sur la plupart des box françaises, par défaut.

Parmi les différentes implémentations de clients UPnP-IGD qui peuvent exister, je n'en ai utilisé qu'une seule : [miniupnp](http://miniupnp.free.fr/). C'est cette implémentation qui est utilisée par le client bittorrent [Transmission](http://www.transmissionbt.com/).

Client en ligne de commande
===========================

Pour jouer avec les redirections de ports directement depuis la ligne de commande :

    $ sudo apt-get install miniupnpc
    $ upnpc -a 192.168.1.10 22 12345 tcp  # redirige le port 12345 de la box vers mon port 22 local
    $ upnpc -l                            # liste les redirections existantes
    $ upnpc -d 12345 tcp                  # supprime la redirection créée précédemment


Utilisation depuis Python
=========================

Miniupnp est une bibliothèque écrite en C qui peut probablement être utilisée depuis la plupart des langages, mais la chose est triviale en Python puisque le wrapper est fourni.

**Installation** :

    $ mkvirtualenv test-upnp-igd
    $ git clone https://github.com/miniupnp/miniupnp/
    $ cd miniupnp/miniupnpc/
    $ make
    $ python setup.py install

**Utilisation** : j'ai réalisé un petit script `upn.py` ([source](https://gist.github.com/fspot/9770024)) qui permet d'ajouter/supprimer/lister les redirections de ports (limitation : le port interne et externe doivent être les mêmes).

    $ python upn.py add 6667 tcp     # redirige le port 6667 de la box vers mon port 6667 local
    $ python upn.py list
    $ python upn.py delete 6667 tcp  # supprime la redirection créée précédemment

Autres techniques de *NAT Traversal*
====================================

Parfois, l'UPnP-IGD ou les techniques similaires ne fonctionnent pas. Pour autant, tout n'est pas perdu, il existe des moyens de contourner le problème en tirant partie de la redirection automatique des ports effectuée par le routeur lors de connexions sortantes. Quelques liens sur le sujet :

- <https://en.wikipedia.org/wiki/NAT_traversal>
- <https://en.wikipedia.org/wiki/STUN>
- <https://en.wikipedia.org/wiki/Hole_punching>
- <https://progdupeu.pl/forums/sujet/521/communiquer-de-derriere-des-pare-feux>
- <https://tools.ietf.org/html/rfc5128>

La technologie WebRTC (le peer-to-peer directement dans le navigateur), notamment, utilise ces techniques là (ICE).

