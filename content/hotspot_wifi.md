Date: 29/03/2013
Title: Hotspot Wifi
Slug: hotspot-wifi

Dans [mon article précédent](|filename|piratebox-raspberry-pi.md) j'aboutis à un point d'accès Wifi qui enferme les clients dans un sous réseau `192.168.37.0/24` depuis lequel ils ne peuvent pas sortir (pas même un `ping`), et où seul un site web leur est accessible, celui situé sur le Raspberry Pi (abrégé Rpi).

Il ne reste plus grand chose à faire pour que ce point d'accès devienne un véritable hotspot Wifi pour partager un accès à internet.

Sommaire :

- sécuriser le point d'accès en WPA avec **hostapd**
- virer le DNS menteur `fakedns.py`
- activer le routage sur Rpi
- activer le NAT sur Rpi avec **iptables**

Sécurité du point d'accès
=========================

Hostapd permet de filtrer les adresses MAC des clients, mais une adresse MAC ça se change en une ligne de commande, donc ne comptez pas trop sur ce genre de sécurités, tout comme la non diffusion du SSID, etc.

Rien ne vaut une bonne grosse passphrase WPA2.

Je n'ai pas réalisé cette partie, mais a priori il suffit de modifier le fichier de conf de hostapd en y rajoutant ceci pour du WPA2 :

	wpa=2
	wpa_passphrase=secret
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=CCMP
	rsn_pairwise=CCMP

Ou ça pour du WPA1 :

	wpa=1
	wpa_passphrase=secret
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=TKIP

Ou ça pour les deux à la fois :

	wpa=3
	wpa_passphrase=secret
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=TKIP
	rsn_pairwise=CCMP

Pensez ensuite à relancer hostapd.

DNS
===

Dans l'article précédent nous avions mis en place un DNS menteur qui va bien embêter nos clients si on ne l'enlève pas.

Je vois deux solutions :

- vous voulez une solution type PirateBox qui fasse point d'accès Internet et vous visez un public un minimum technophile : sur la page de la PirateBox vous indiquez la procédure de changement de DNS sur [Windows](http://www.aidoweb.com/tutoriaux/changer-serveurs-dns-windows-7-955), Mac et [GNU/Linux](http://diablotins.org/index.php/Resolv.conf)
- vous ne voulez pas de PirateBox mais seulement un point d'accès Internet sans prise de tête : vous virez le programme **fakedns.py**, et vous avez là encore deux possibilités :
    - vous installez et configurez un vrai serveur DNS sur votre Rpi
    - vous utilisez un serveur DNS existant (ceux de votre FAI, de Google, ou d'OpenDNS par exemple), en modifiant la ligne `option domain-name-servers` du fichier `/etc/dhcp/dhcpd.conf` :

Il suffit de remplacer l'adresse IP du Rpi par celle(s) de DNS providers. Exemple avec les DNS de Google :

	option domain-name-servers 8.8.8.8, 8.8.4.4; # dns de google

Routage
=======

Pour l'instant, vos clients sont confinés au sous réseau `192.168.37.0/24`. S'ils veulent atteindre une machine qui n'est pas sur ce sous réseau, ils vont transférer les paquets à Rpi, qui, pour l'instant, va les balancer.

Pour empêcher ça, il faut activer le routage sur le Rpi :

	$ sudo su
	# echo 1 > /proc/sys/net/ipv4/ip_forward

Ainsi les paquets seront bien transférés.

NAT
===

Il reste cependant un problème : le destinataire des paquets peut ne pas savoir comment joindre l'émetteur car il ne dispose pas de la route menant au sous réseau `192.168.37.0/24`.
Malheureusement, on ne peut pas modifier les routes de la freebox. On va donc mettre en place un NAT : les destinataires croiront que l'émetteur est Rpi, ils n'auront pas conscience que Rpi cache un accès à sous réseau, et ils savent comment joindre Rpi. Rpi, de son côté, va faire l'intermédiaire avec les clients du Hotspot pour leur refiler les réponses de leurs destinataires.

Voilà la commande magique :

	$ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

`eth0` est l'interface du Rpi qui est reliée à Internet et non au Hotspot.

En procédant ainsi, en revanche, les clients du Hotspot ne pourront pas être joints directement par Internet (à moins de mettre en place un tunnel). Ils seront vus seulement par Rpi et par les autres clients.

Fin
===

Une fois de plus je vous laisse mettre en place toutes les procédures d'automatisations nécessaires :).
Voilà qui devrait clore mes investigations sur les Hotspots Wifi. Ou alors, prochaine étape, le réseau mesh ;).
