Date: 24/08/2013
Title: keychain.io pour simplifier vos accès ssh via id_rsa.pub
Slug: keychain-io-pour-simplifier-vos-acces-ssh-via-id_rsa-pub

Quand on veut se connecter à une machine via SSH, on tape généralement son mot de passe. Ceci pose des problèmes de sécurité, est répétitif, et rend impossible l'automatisation d'actions via ssh.

Cette étape peut être évitée en utilisant une solution à base de clef publique / clef privée. Si vous ne possédez pas les fichiers `~/.ssh/id_rsa` et `~/.ssh/id_rsa.pub`, ceux cis se génèrent ainsi :

	# mettez l'email que vous voulez, hein.
	$ ssh-keygen -t rsa -C "username.hostname@fspot.org"

Cette commande demande quelques renseignements, appuyez bêtement sur entrée, ça ira (mais ça ne mettra pas de passphrase sur votre clef privée). Idéalement, votre clef privée devrait être protégée par une passphrase, autrement, quiconque l'obtient peut l'utiliser pour se connecter aux machines qui font confiance à cette clef. Inconvénient, cette passphrase vous oblige à nouveau à taper un mot de passe... Mais il existe une solution à base de `ssh-agent` que je n'ai pas creusée, on verra ça plus tard.

Dans un premier temps, on fera sans passphrase donc. De toute façon, pour rajouter une passphrase sur une clef existante, c'est simple :

	$ ssh-keygen -p  # suivez les instructions, ensuite.

Pour pouvoir accéder à un serveur grâce à vos clefs, il faut uploader votre clef publique sur ce serveur et la mettre dans le fichier `~/.ssh/authorized_keys`

	$ scp ~/.ssh/id_rsa.pub login@serveur:~/clef.pub
	$ ssh login@serveur "cat clef.pub >> .ssh/authorized_keys"

Ces deux étapes sont à répéter souvent si vous souhaitez accéder à de nombreux serveurs. Du coup, Jeff Lindsay (progrium) a fait une petite app qui automatise ça.

Admettons que je sois sur ma machine tux avec mon utilisateur fred. Je fais ça une bonne fois pour toutes :

	$ curl -s ssh.keychain.io/fred.tux@fspot.org/upload | bash

Je doit valider cette action dans un mail de confirmation (cette adresse email n'existait pas, mais comme le domaine fspot.org m'appartient, je peux quand même récupérer tous les mails en @fspot.org avec une adresse catch-all). Ensuite, sur chaque serveur :

	$ curl -s ssh.keychain.io/fred.tux@fspot.org/install | bash

Je peux aussi demander à un ami de me donner un accès sur son serveur en lui demandant d'entrer cette commande. Grâce au mail de confirmation, s'il me fait confiance (ainsi qu'à keychain.io), il peut être sûr que la clef sera bien la mienne. Bref, ça peut dépanner.

Bonus : en rédigeant cet article, je suis tombé sur ces super [slides sur SSH de rom1v](http://blog.rom1v.com/wp-content/uploads/2008/08/ssh-slides.pdf).
