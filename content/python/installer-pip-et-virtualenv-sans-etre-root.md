Date: 24/07/2013
Title: Installer Pip et Virtualenv sans être root
Slug: installer-pip-et-virtualenv-sans-etre-root

Sur une install fraîche d'un GNU/Linux, il y a bien python, mais il n'y a ni pip, ni virtualenv, ni même easy_install. C'est très dommage, et c'est une des raisons qui fait que par exemple, ici à mon boulot, quand on utilise par hasard python, c'est sans modules (ou alors avec des modules qui tiennent en un fichier, genre bottle.py, path.py, peewee.py).

Pourtant, même sans être root, il est très facile de se mettre en place ces outils, pour peu qu'on soit connecté à internet (et si c'est pas le cas, une petite archive et hop).

Voici un petit script, `create_venv.sh`, qui fait le boulot :

	:::bash
	#!/bin/bash

	set -e

	echo "> Downloading virtualenv.py..."
	wget -q https://raw.github.com/pypa/virtualenv/develop/virtualenv.py # -O virtualenv.py
	# or curl -s https://raw.github.com/pypa/virtualenv/develop/virtualenv.py > virtualenv.py

	echo "> Creating and activating new virtualenv <venv>..."
	python virtualenv.py venv > out.log
	source venv/bin/activate

	echo "> Downloading setuptools/easy_install in <venv>..."
	wget -q https://bitbucket.org/pypa/setuptools/raw/bootstrap/ez_setup.py # -O ez_setup.py
	python ez_setup.py >> out.log 2>&1

	echo "> Installing pip in <venv>..."
	easy_install pip >> out.log 2>&1

	echo "> Cleaning..."
	rm *.tar.gz *.py *.pyc
	rm out.log

	echo "Done."

Utilisation :

	:::bash
	curl -sL https://raw.github.com/fspot/create-venv/master/create_venv.sh | bash

- Exemple live : [screencast](http://ascii.io/a/4310)
- [github repo](https://github.com/fspot/create-venv/)

