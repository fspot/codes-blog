Date: 10/08/2014
Title: Zenfeed
Slug: zenfeed

Présentation
============

Un dicton populaire nous dt que "si à 23 ans, t'as pas écrit ton lecteur RSS, t'as raté ta vie (de développeur)". C'est pour cela, et pour quelques autres raisons, que j'ai écrit le mien, [zenfeed](https://github.com/fspot/zenfeed), qui vient s'ajouter à la longue liste des lecteurs RSS en ligne, aux cotés de :

- feu Google Reader ou Feedly, pour citer les services propriétaires
- miniflux, Leed, tiny tiny RSS, RSSLounge ou Kriss Feed (la liste est longue)

Il est ouvertement inspiré de [zencancan](http://zencancan.com). Ce dernier affiche des objectifs très clairs :

> zenCancan affiche seulement le dernier article de chaque site que vous suivez.
> zenCancan ne vous indiquera jamais que vous n'avez pas lu un article, c'est la partie zen de l'application.
>
> Si vous utilisez souvent d'autres lecteurs de flux, vous avez remarqué que, tout comme pour le mail, il vous oblige à tout lire en indiquant les articles que vous n'avez pas lu.
> Vous êtes constamment sous le stress d'avoir tous vos flux à "0 nouvelles non lues" !
> La lecture de flux devant rester amusante et zen, vous ne verrez jamais sur zenCancan un message vous dire que vous n'avez pas fait telle ou telle chose !

Après avoir utilisé personnellement zenCancan pendant de nombreuses années, j'en étais très satisfait, à quelques petits points près :

- l'interface est un peu vieillotte, et n'est pas adaptée aux mobiles. J'aimerai avoir une interface adaptée à la lecture.
- j'aurai aimé, pour **certains** flux que je considère comme importants, une mise en évidence visuelle lorsque des nouvelles y sont apparues (cela vient un peu contredire l'esprit originel de zenCancan)
- je n'ai jamais utilisé le système d'étiquettes (tags / catégories)
- rien n'est paramétrable (pagination, intervalles entre les récupérations de flux…)
- zenCancan ne sauvegarde pas les news qui ne sont plus présentes dans le flux RSS, ce qui limite le nombre d'entrées à seulement une dizaine dans la plupart des cas (et inversement, à beaucoup trop dans d'autre cas).

Le logiciel étant libre, j'ai tenté d'en faire tourner une instance localement, sans succès. L'installation et l'exécution nécessitent de nombreuses étapes (fichiers de configuration, setup d'un serveur MySQL, tâches cron…). J'ai donc entrepris de réinventer la roue.

Zenfeed
=======

Comme zenCancan, zenfeed a pour particularité de **ne pas** afficher une traditionnelle liste de news non-lues, assortie d'un compteur de "news restantes à lire".

Interface
---------

L'interface d'utilisation "au quotidien" se veut très simple, légère et épurée, et ne nécessite pas javascript (sauf pour activer la navigation au clavier − `z,s,q,d`). Elle comporte 3 vues :


<img style="float:right; box-shadow: 0px 4px 10px #000; margin-left: 15px;" width="50%" src="https://cdn.mediacru.sh/85fg1zEf8fQR.png" />

- [vue principale](https://cdn.mediacru.sh/85fg1zEf8fQR.png) : Affiche la liste des flux dans l'ordre décroissant de mises à jour. Ainsi, les flux ayant eu des news aujourd'hui-même sont directement en haut de la page. La favicon de chaque site est téléchargée lors de la récupération initiale de celui-ci, et affichée sur cette page.
- [vue d'un flux](https://cdn.mediacru.sh/pX8d2iWXpHI6.png) : Obtenue en cliquant sur ce flux dans la vue principale, et affichant la liste des news de ce flux (seulement un petit extrait du corps de chaque news).
- [vue d'une news](https://cdn.mediacru.sh/3iiUrJfCRFql.png) : Obtenue en cliquant sur cette news dans la vue du flux correspondant, et affichant le corps complet de la news.

Administration
--------------

L'[interface d'administration](https://cdn.mediacru.sh/DRRPrVzm1On_.png) est accessible à l'url relative `/admin/` (mot de passe par défaut : `zenfeed`), et est réalisée avec AngularJS.

On peut y ajouter / supprimer des flux, modifier la configuration par défaut ou [celle individuelle de chaque flux](https://cdn.mediacru.sh/I6U0s_AkoaRz.png) :

- intervalle de mise à jour du flux
- nombre d'entrées par page
- nombre maximal d'entrées en base de données
- surligner ou non le flux quand il a été mis à jour (le flux sera considéré comme "lu" lors d'un clic sur celui-ci)

Fonctionnalités
---------------

Mis à part la description de l'interface faite ci dessus, il y a peu à dire concernant les "fonctionnalités" de zenfeed, car celui-ci se veut très simple.

- [Interface web responsive](https://cdn.mediacru.sh/2qFVC52hgB6X.png) (grâce aux grids de <http://purecss.io>)
- Léger côté front (2 petits fichiers CSS et 1 police, 50 lignes de JS)
- Peut fonctionner avec SQLite, MySQL ou PostgreSQL
- Rapide (mise en cache systématique des pages lors de leur changement)
- "Simple" (environ 1000 lignes de Python)
- Possibilité d'ajouter un flux depuis l'interface d'administration en entrant directement l'URL du blog / du site
- Tourne très bien chez moi depuis plusieurs mois, avec une trentaine de flux et environ 12 000 news maintenant.
- Facile à installer et à lancer

Ce dernier point est finalement ce qui le différencie de plus de zenCancan. Pour qui possède un PC ou un serveur sous GNU/Linux, l'installation et le lancement se font comme cela :

    $ pip install zenfeed
    $ zenfeed

Un [paquet debian/ubuntu](http://fspot.org/pkg/zenfeed.deb) est fourni pour ceux n'ayant pas la patience de récupérer zenfeed *via* `pip`, qui nécessite la compilation des dépendances C (`gevent`, `sqlalchemy`) et les paquets `python-pip` et `python-dev`.

Pour plus de détails, je vous renvoie à la [documentation](http://fspot.github.io/zenfeed/).

Todo
====

- import / export OPML
- tester zenfeed avec MySQL et PostgreSQL
- prendre en compte la limite d'entrée en base de données :)
