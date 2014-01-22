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
    RUN echo "\ndaemon off;" >> /etc/nginx/nginx.conf
    
    # on change de répertoire de travail
    WORKDIR /etc/nginx
    
    # on rend le port accessible à la machine hôte
    EXPOSE 80
    
    # on définit la commande par défaut
    ENTRYPOINT ["nginx"]


Cette image a été construite à partir de ce Dockerfile (via la commande `docker build`) puis publiée (`docker push`) sur l'index public Docker par l'utilisateur `dockerfile`, qui l'a nommée `nginx`. Pour récupérer cette image, il suffit donc d'entrer la commande suivante :


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

### Volumes

Docker permet de partager des volumes entre containers (des portions du système de fichier). Ces volumes sont exclus de la gestion traditionnelle d'AUFS, et ne risquent pas d'être *committés*, tout en permettant une persistance des données. De manière plus intéressante, plusieurs containers peuvent se partager des volumes communs, permettant ainsi un partage de fichier malgré l'isolation du filesystem. Pour finir, on peut également partager un volume de l'hôte avec ses containers.

À titre d'exemple, l'image `paintedfox/postgresql` permet de lancer une instance de postgresql dont les données seront dans un volume spécifié.


    # les données de postgresql seront persistées dans /tmp/pg
    docker run -d -p 5432:5432 -v /tmp/pg:/data paintedfox/postgresql


### Links

Dans le même registre, docker permet de créer des liens entre les containers, afin de s'affranchir du minimum d'isolation nécessaire à une découverte et une communication réseau avec d'autres containers.

Avant que cette possibilité soit intégrée à docker, il était toujours possible de passer directement par les adresses ip des containers et de réaliser un contrôleur réalisant l'orchestration. Mais depuis que cette fonctionnalité est présente, il est recommandé de rendre impossible la communication entre containers quand celle ci n'est pas explicitement demandée grâce aux links.

La mise en place d'un link, du point de vue du container, se concrétise par le positionnement de différentes variables d'environnements, permettant la découverte des containers liés.

Exemple avec un link :

    root@xub:~/# docker ps
    CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                    NAMES
    ae3a6d879bd2        myapp:latest        /bin/sh -c python ap   29 minutes ago      Up 29 minutes       0.0.0.0:5000->5000/tcp   sad_einstein        
    root@xub:~/# docker run -i -t -rm -link sad_einstein:foo ubuntu bash
    root@2a5aed07acc1:/# env
    FOO_NAME=/stupefied_bardeen/foo
    FOO_PORT=tcp://172.17.0.12:5000
    FOO_PORT_5000_TCP=tcp://172.17.0.12:5000
    FOO_PORT_5000_TCP_PORT=5000
    FOO_PORT_5000_TCP_PROTO=tcp
    FOO_PORT_5000_TCP_ADDR=172.17.0.12
    HOSTNAME=2a5aed07acc1
    TERM=xterm
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    PWD=/
    SHLVL=1
    HOME=/
    container=lxc
    _=/usr/bin/env
    root@2a5aed07acc1:/#

La variable `FOO_PORT` résume à elle seule le lien existant vers le container `sad_einstein` (`foo` est le nom d'alias, `sad_einstein` est un nom généré aléatoirement par docker, on peut en spécifier un avec `-name`).

### Exemple concret

Soit l'exemple suivant :


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


On peut l'obtenir facilement en lançant 2 containers Django et un 3e avec nginx en reverse proxy en load balancing sur les 2 premiers. En cherchant les adresses IP et les ports des deux containers Django (*via* les variables d'environnement vues précédemment, ou, plus salement, *via* la commande `docker inspect <container>`), on peut arriver à cette conf :


    upstream myapp {
        server 172.17.0.12:5000;
        server 172.17.0.15:5000;
    }


Cependant, ces valeurs en dur dans la configuration ne sont pas du tout satisfaisantes : un moyen de les éviter serait de générer ces fichiers de configuration à partir de templates lors du démarrage du container, en se basant sur les variables d'environnements issues des `links`. Pour les applications sur lesquelles on a la main, on peut directement traiter ces variables dans le code applicatif : le fait qu'elles soient fournis par docker ou non est transparent.

### Pattern Ambassador

Dans des cas réels, tous nos containers auront peu de chance d'être sur la même machine. Malheureusement, le `link` ne fonctionne qu'au sein d'une même machine hôte.

Pour compenser cela, l'idéal est d'utiliser un agent intermédiaire, permettant de ne pas polluer les containers métier avec la gestion des containers distants. Le *pattern ambassador* suit ce modèle :


    # Exemple avec un client et un serveur redis
    (consumer) --> (redis-ambassador) ---network---> (redis-ambassador) --> (redis)


Dans ce schéma, l'ambassadeur est un container dont l'unique rôle est de mettre en place un tunnel vers son homologue et d'exposer ses ports.

Cas d'utilisations de docker
----------------------------

Docker a à peine un an et de nombreux cas d'utilisation restent à découvrir.
Le plus évoqué est la création de solutions PaaS ou SaaS basées sur Docker (cf. [Deis](http://deis.io) ou [Flynn](https://flynn.io)). Pour ce qui est du paiement, ces solutions tirent parti de la capacité des containers à limiter et tracer leur consommation mémoire et entrées/sorties.
Également, Docker peut être ajouté au processus d'intégration et de déploiement continu : construction d'une image, `push` sur l'index (privé), `pull` sur les serveurs de qualification, exécution des tests d'intégration, `pull` sur les serveurs de production.
On voit également apparaître des usages visant à fournir aux membres d'une équipe un environnement de développement uniforme, résolvant les problèmes de dépendances, et homogène avec l'état de la production.
Docker peut aussi remplacer de fastidieuses installations d'applications par un simple `docker pull`.

Dans de nombreux cas, docker se place en concurrent des traditionnelles machines virtuelles. S'il reste confiné à certains environnements, il est cependant incroyablement plus léger. À titre d'exemple, mon laptop a supporté sans broncher 150 containers de nginx (chaque container possédant 1 master + 4 workers nginx).

Limites
-------

En terme de rapidité et d'espace disque, les containers ne sont pas si légers que cela. A titre d'exemple, l'image `postgresql` précédente m'a pris 8 minutes à récupérer (136 Mo), en possédant pourtant l'image `ubuntu` sur laquelle elle se base. Dans des cas moins favorables (i.e. à la campagne), j'ai même eu droit à des timeouts.
En termes de sécurité, une connaissance pointue du fonctionnement des linux containers est requise pour maîtriser les risques. Cependant, même en souffrant d'un niveau d'isolation moindre, il semblerait que les containers ne soit pas beaucoup plus dangereux que des machines virtuelles. Voir le [billet de blog](http://blog.docker.io/2013/08/containers-docker-how-secure-are-they/) de dotcloud sur le sujet.
