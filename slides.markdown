# Salt

---

# Au programme du workshop

* présentation de salt
* explication de comment lire et faire des states
* présentation des states les plus utile
* installation pour ceux qui l'ont pas déjà fait (je parie que c'est tout le monde)
* exercices

---

# Salt

C'est quoi ?

* un outil d'orchestration
* un config manager
* une architecture client serveur

Mais aussi:

* permet de gérer des vms (salt-cloud)
* et probablement d'autre trucs (prototyping pour faire du monitoring)

---

# Type de deployement

* en mode client-serveur (/!\ crypto maison)
* en local   ← on va probablement utiliser ça aujourd'hui
* Nouveau: ansible-like, via ssh sans minion

Remarque: peut avoir plusieurs master en mode client-serveur

---

# Questions sur c'est quoi salt ?

---

# rapide intro à l'orchestration

---

# L'orchestration en 5min

Simple:

    salt <pattern ou nom de machine> <commande>

Exemples:

    salt '*' test.ping

    salt mon_precieux.com cmd.run "ls"

    salt-call --local state.highstate

Commande à retenir: **state.highstate** synchronise les states.

Liste des modules disponible:

Vocabulaire (salt a plein de termes à la con): module == "commande d'orchestration"

Remarque: en vrai, un module c'est &lt;module python&gt;.&lt;function dans le dis fichier&gt;

Tous les modules disponible: [http://docs.saltstack.com/ref/modules/all/index.html](http://docs.saltstack.com/ref/modules/all/index.html)

---

# States

---

# Détails importants

* les states sont écrits dans des fichier en ".sls"
* on doit définir un fichier top.sls dans /srv/salt/top.sls (customisable)
* exemple de top.sls:

    !yaml
    base:
      '*':  # tout le monde
        - github
        - zabbix-agent
        - users

      'mon.server_1.com':
        - ii
        - logstash
        - clean_backups

* chaque nom dans top.sls pointe vers soit:
     - un fichier du même nom en .sls (eg: github.sls)
     - un répertoire du même nom contenant un fichier init.sls (exemple: github/init.sls

* rappel: une fois que les states sont ecrit, pour les "executer" utiliser le module state.highstate

---

# Les states

Un state est un état dans lequel on veut que la machine soit. Exemple: je veux
que ce package soit installé, je veux que cet user existe.

Généralement très proche d'une commande ou d'une série de commandes shell (mais
pas exactement).

Structure (en yaml):

    !yaml
    nom_du_state:
      module_python.fonction_python:
        - clef_1: argument_1
        - clef_2: argument_2
        - clef_3: argument_3
        ...

Vous l'avez deviné, c'est exactement ça en python:

    !python
    from module_python import fonction_python

    fonction_python(clef_1=argument_1, clef_2=argument_2, clef_3=argument_3, ...)

---

# Exemple

    !yaml
    fred_user:
      user.present:
        - name: fred
        - fullname: Fred Jones
        - shell: /bin/zsh
        - home: /home/fred
        - uid: 4000
        - gid: 4000
        - groups:
          - wheel
          - storage
          - games

    dependance:
      pkg.installed:
        - name: git

Rappel:

    !yaml
    nom_du_state:
      module_python.fonction_python:
        - clef_1: argument_1
        - clef_2: argument_2
        - clef_3: argument_3
        ...

---

# Entering salt confusing syntaxe ! #F34R

---

# useless name !

Rappel:

    !yaml
    nom_du_state:
      module_python.fonction_python:
        - name: <name>
        - clef_2: argument_2
        - clef_3: argument_3
        ...

Tous les states salt acceptent comme premier argument "name", c'est une convention. Donc ils ont décidé qu'on pouvait écrire ça comme ça:

    !yaml
    <name>:
      module_python.fonction_python:
        - clef_2: argument_2
        - clef_3: argument_3
        ...

Aussi possible (si pas d'arguments):

    !yaml
    <name>:
      module_python.fonction_python

---

# encore plus !

Rappel:

    !yaml
    nom_du_state:
      module_python.fonction_python:
        - name: <name>
        ...

Ils ont aussi décidé qu'on avait le droit d'écrire ça comme ça:

    !yaml
    nom_du_state:
      module_python:
        - fonction_python
        - name: <name>
        ...

Utilise uniquement ici:

    !yaml
    nom_du_state:
      module_python_1:
        - fonction_python_1
      module_python_2:
        - fonction_python_2

Car ceci est illégale en yaml:

    !yaml
    nom_du_state:
      module_python_1.fonction_python_1
      module_python_2.fonction_python_2

---

# jinaj2

Vous pouvez utiliser du jinja2 (un système de tempalting python similaire à celui de django) directement dans vos states.

Exemple:

    !yaml
    {% for pkg in ['git', 'svn', 'hg'] %}
    {{ pkg }}:
      pkg.installed
    {% endfor %}

Remarque: ne codez pas ça, y a un moyen bien plus simple de le faire.

---

# Ordonner les states

---

# Ordre d'execution des states

NOUVEAU: désormait les fichiers .sls sont executé séquentiellement, ce n'était pas le cas avant.

Il est possible de créé des dépendances d'excution entre state avec "require" et "require\_in".

ATTENTION: syntaxe confusante:

    !yaml
    service apache2 start:
      cmd.run:
        - require:
          - pkg: apache-fpm-prefork

Ici "require" n'est PAS un argument pour l'appel de la fonction python.

"require\_in" fait l'inverse de "require", il dit "je suis nécessaire pour ce
state" ("require" dit "j'ai besoin de ce state").

---

# Gestion des événements

---

# Retour d'un state

Chaque state peut revenir dans 3 états différents:

* erreur (rouge): je me suis torché
* ok (vert): je suis déjà dans l'état désiré
* j'ai du agir pour passer dans l'état désiré (bleu)

C'est le dernier qui est intéressant car il permet de triggerer un autre state grâce à "watch" et "watch\_in".

Exemple: si on répertoire git a été mis à jours, je veux lancer un collectstatic (django).

    !yaml
    git://pouet.com/pouet.git:
      git.latest:
        - target: /home/moi/pouet

    python manage.py collectstatic --noinput:
      cmd.wait:
        - cwd: /home/moi/pouet
        - user: moi
        - watch:
          - git: git://pouet.com/pouet.git

Remarque: ici non plus, "watch" et "watch\_in" ne sont **PAS** des arguments pour la fonction.

Remarque: y a un mod\_watch qui existe parfois dans certains states pour faire une action spécial si jamais ils sont appelé via un watch (par exemple "service.running").

---

# Et voilà, il faut juste savoir ça !

---

# Liste de states cool

---

# User

    !yaml
    fred:
      user.present:
        - fullname: Fred Jones
        - shell: /bin/zsh
        - home: /home/fred
        - uid: 4000
        - gid: 4000
        - groups:
          - wheel
          - storage
          - games


Remarque: user.absent existe aussi

Exemple de doc: [http://docs.saltstack.com/ref/states/all/salt.states.user.html#module-salt.states.user](http://docs.saltstack.com/ref/states/all/salt.states.user.html#module-salt.states.user)

---

# pkg.installed

    !yaml
    vim:
      pkg.installed

Remarque: salt va automagiquement aller choisir votre gestionnaire de pkg (marche pour apt et yum en tout cas).

Astuce, vous avez le droit d'écrire:

    !yaml
    mes_pkgs:
      pkg.installed:
        - names:
          - vim
          - emacs
          - nano
---

# file.managed

    !yaml
    /etc/http/conf/http.conf:
      file.managed:
        - source: salt://apache/http.conf
        - user: root
        - group: root
        - mode: 644
        - template: jinja
        - defaults:
            custom_var: "default value"
            other_var: 123

Existe aussi: file.directory, file.symlink, file.exists et beaucoup d'autres.

Remarque: on peut aussi mettre une url combiné avec un hash du fichier.

---

# git.latest

    !yaml
    https://github.com/saltstack/salt.git:
      git.latest:
        - rev: develop
        - target: /tmp/salt

---

# cmd.run et cmd.wait

Super pratique !

    !yaml
    service apache2 reload:
      cmd.wait:
        - watch:
          - git: <url de mon super projet>

    python manage.py collectstatic:
      cmd.run:
        - unless: ls /path/vers/les/statics

---

# cron.present

    !yaml
    date > /tmp/crontest:
      cron.present:
        - user: root
        - minute: 7
        - hour: 2

---

# Divers

* au début on a parfois du mal à savoir quoi faire car c'est une autre façon de penser. Basiquement c'est généralement coder en state exactement ce qu'on aurait fait à la main PLUS ce qu'il faut faire pour une mis à jours (à coup de watch).

* vous pouvez aussi écrire vos states en:
    * full python
    * dans un dsl python
    * en json
    * en yaml/mako
    * voir écrire votre propre renderer
    * mais yaml/jinja2 convient dans l'imense majorité des cas

* vous pouvez écrire le truc appelé par vos states, c'est vraiment pas difficile
* pareil pour les modules
* hésitez pas à aller voir le code

* y a BEACOUP plus que ce que je vous ai montré, vraiment (pillar, grains, include, gitfs, se parler entre minions etc ...)

En fait, quasiment tout est customisable dans salt, mais les defaults sont suffisant.

---

# Exercice: ce que vous voulez

Propositions:
* déployer un projet django classique (genre carnet-rose) avec toute la stack
* déployer un projet django plus avancé: hacker agenda
* utiliser salt pour gérer la config de votre machine (user, dotfiles, pkgs, config de vos apps)
* vos idées

Remarque: apprennez direct à utiliser la documentation (demo).
