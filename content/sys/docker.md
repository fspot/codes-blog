Date: 21/01/2014
Title: Docker
Slug: docker
Status: draft

Présentation
------------

Docker est un projet permettant de packager ses applications dans des *containers*, ceux-cis pouvant ensuite être facilement stockés, déployés et exécutés sur tout type de machine. Un container est auto-suffisant, il regroupe l'application et toutes ses dépendances. Lors de son exécution, il est isolé du système d'exploitation hôte : il possède son propre système de fichier, son propre nom de machine, sa propre adresse IP, ses propres processus, etc. En cela un container se rapproche d'une machine virtuelle, mais en réalité il s'agit seulement d'un processus bien isolé, l'overhead associé est négligeable. Un simple ordinateur portable peut potentiellement exécuter des centaines de containers.

Ces containers ne sont pas une nouveauté apportée par Docker : celui-ci tire parti des Linux Containers (LXC). Docker rend leur utilisation simple, et y intègre d'autres outils, tel que le système de fichier AUFS (Another Union File System). Sans rentrer dans les détails, l'utilisation d'AUFS permet de partager, entre différents containers, des portions de système de fichiers qui soient identiques, et ainsi d'éviter le gaspillage d'espace disque et de cache. On peut également faire un *diff* entre 2 versions d'un filesystem AUFS. Un intérêt réside dans l'économie de ressources réseaux lorsqu'on télécharge un nouveau container : s'il partage une base commune avec un container déjà récupéré, alors on ne télécharge que le diff ! (quelques dizaines de Mo en général).

Je ne vais pas m'étendre sur le fonctionnement de Docker [*sous le capot*](http://www.infoq.com/fr/articles/docker-sous-le-capot), d'ailleurs, la version 1.0 de Docker ne sera plus spécifique aux LXC ni à AUFS, ce qui permettra à cet outil de fonctionner sur davantage de plateforme. En effet, et c'est là un problème non négligeable, Docker est pour l'instant cantonné à quelques distributions GNU/Linux, et requiert un noyau récent (>= 3.8). D'ici quelques temps, on peut cependant espérer que ces exigences seront facilement satisfaites.

Dockerfiles
-----------

Dans la terminologie utilisée par Docker, un *container* est une instanciation d'une *image*.
La génération d'une image peut être scriptée dans un fichier `Dockerfile`. Ce fichier, pour l'essentiel, décrit les étapes nécessaires à la préparation de notre image, et doit s'appuyer sur une image préexistante. Par exemple, on peut baser notre image sur une image vierge d'ubuntu 12.04, disponible dans l'[index public d'images Docker](https://index.docker.io).

Voici un exemple de Dockerfile trivial permettant de générer une image contenant l'application nginx (un serveur http). Les commandes parlent d'elles même.


    # on précise l'image sur laquelle on s'appuie
    FROM dockerfile/ubuntu
    
    # on installe nginx
    RUN apt-get install -y software-properties-common
    RUN add-apt-repository -y ppa:nginx/stable
    RUN apt-get update
    RUN apt-get install -y nginx
    
    # on change de répertoire de travail
    WORKDIR /etc/nginx
    
    # on rend le port accessible à la machine hôte
    EXPOSE 80
    
    # on définit la commande par défaut
    ENTRYPOINT ["nginx"]


Cette image a été publiée sur l'index public Docker par l'utilisateur `dockerfile`, qui l'a nommée `nginx`. Pour récupérer cette image, il suffit d'entrer la commande suivante :


    docker pull dockerfile/nginx


Et pour exécuter un container basé sur cette image :


    docker run -d -p 8000:80 dockerfile/nginx


Cette dernière commande a pour effet de lancer en tâche de fond (`-d`) le container précédemment récupéré (`dockerfile/nginx`), en reliant le port 8000 de l'hôte au port 80 exposé par le container (`-p 8000:80`).
On peut constater que nginx tourne bien :


    curl localhost:8000


On peut également lancer ce container une seconde fois, en parallèle, et utilisant par exemple le port 8001 de l'hôte : 


    docker run -d -p 8001:80 dockerfile/nginx


Les deux containers sont indépendants l'un de l'autre, isolés, et pourtant reposent sur une même image.
Il est facile de créer son propre Dockerfile, de générer l'image correspondante, et de la pusher sur l'index docker. Ainsi, n'importe qui peut la récupérer et exécuter l'application qui s'y trouve : plus de problèmes de dépendances !

Interactions entre containers
-----------------------------

Soit une stack applicative traditionnelle pour un projet web:
- nginx en serveur http : reverse proxy vers le serveur applicatif, et sert les fichiers statiques
- gunicorn en serveur wsgi
- django comme framework web et différents modules python (v2.7) nécessaires à l'application
- postgresql en SGBD
- redis en cache

On pourrait créer une image regroupant toute cette stack, mais ce n'est pas une utilisation correcte de Docker car cela nous bloquerait pour un déploiement sur plusieurs serveurs distincts. Il vaut mieux créer une image pour chaque partie de la stack, et faire interagir les containers ensemble par la suite.

Cependant, on a vu précédemment qu'un container en exécution était isolé. Heureusement, docker nous fournit de quoi faire interagir nos containers entre eux.

### Volumes ?

### Links

### Pattern Ambassador

### Exemple concret

On préfère souvent avoir plusieurs petits serveurs applicatifs qu'un seul gros, afin de minimiser les dégâts quand l'un d'eux tombe.


           +------------+
           |  Nginx     |
           +------------+
             +       +
             |       |
             v       v
    +-----------+  +-----------+
    | Django on |  | Django on |
    | Gunicorn  |  | Gunicorn  |
    +-----------+  +-----------+
             +       +
             |       |
             v       v
           +------------+
           | PostGreSQL |
           +------------+



Cas d'utilisation
-----------------

Docker a à peine un an et de nombreux cas d'utilisation restent à découvrir.
Le plus évoqué est la création de solutions PaaS ou SaaS basées sur Docker (cf. [Deis](http://deis.io) ou [Flynn](https://flynn.io)). Pour ce qui est du paiement, ces solutions tirent parti de la capacité des containers à limiter et tracer leur consommation mémoire et entrées/sorties.
Également, Docker peut être ajouté au processus d'intégration et de déploiement continu : construction d'une image, `push` sur l'index (privé), `pull` sur les serveurs de qualification, exécution des tests d'intégration, `pull` sur les serveurs de production.
On voit également apparaître des usages visant à fournir aux membres d'une équipe un environnement de développement uniforme, résolvant les problèmes de dépendances, et homogène avec l'état de la production.
Docker peut aussi remplacer de fastidieuses installations d'applications par un simple `docker pull`.
