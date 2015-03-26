---
layout: post
title:  "Orchestration de containers Docker – Docker Compose et Crane"
date:   2015-03-26 11:00:00
tags: docker compose crane
language: FR
---
J'ai récemment mis en place un bot pour _tweeter_ plusieurs fois les posts publiés sur [le blog d'Ippon](http://blog.ippon.fr) (Cf. [fil Twitter d'Ippon](https://twitter.com/ippontech)). Cet outil ([rss2twitter](https://github.com/ippontech/rss2twitter)) repose sur deux containers Docker. Pour gérer ces containers, un outil d’orchestration est pratique. Rapide présentation de deux challengers : Docker Compose (anciennement Fig) et Crane.

# La problématique

Le _bot_ utilise deux containers Docker :

- un container Redis pour la persistance (dates de publication des posts...)
- un container pour notre application.

Ces containers sont dépendants l’un de l’autre : l’application doit être connectée (_link_, dans la terminologie Docker) au container Redis. Pour démarrer notre application, il faut donc lancer le container Redis **avant** le container de l’application.

Par ailleurs, la ligne de commande de Docker est assez complexe. Si on veut se souvenir des paramètres adaptés à notre application, on a donc tendance à conserver la ligne de commande dans un ReadMe ou dans des scripts Shell.

En l’occurence, nous avions créé un script Shell:

{% highlight bash %}
#!/bin/bash
docker build -t ippontech/rss2twitter:latest app
docker run -p 0.0.0.0:6379:6379 -v /root/rss2twitter/data:/data -d --name redis redis redis-server --appendonly yes
docker run --link redis:redis -d --name rss2twitter ippontech/rss2twitter
{% endhighlight %}

<img src="/images/orchestration-docker/docker-ps.png">

Ça fonctionne mais c’est très manuel et, au final, pas très pratique. Par exemple, pour arrêter nos containers, il nous faut un autre script. Et si un container vient à tomber, il faut relancer le bon container.

# Docker Compose

[Docker Compose](https://docs.docker.com/compose/) est l’outil “officiel” proposé par _Docker, Inc_ en ayant racheté [Fig](http://www.fig.sh/).

Pour Docker Compose, un fichier `docker-compose.yml` doit être créé. Chaque container doit être décrit : image, commande de lancement, volumes, ports…

Voici le fichier équivalent à nos commandes Shell :

{% highlight yaml %}
redis:
  image: redis
  command: redis-server --appendonly yes
  volumes:
  - /root/rss2twitter/data:/data
  ports:
  - "6379:6379"

rss2twitter:
  build: app
  links:
  - redis
{% endhighlight %}

Au premier lancement, le container _rss2twitter_ (l’application) sera buildé à partir du `Dockerfile` pour être réutilisé lors des prochaines exécutions.

Pour lancer nos containers, il faut utiliser la commande `docker-compose up` :

<img src="/images/orchestration-docker/docker-compose-up.png">

La commande ne rend pas la main. Pour lancer les containers en mode détaché, il faut utiliser l’option `-d` : `docker-compose up -d`

Il est possible d’interagir avec notre ensemble de containers via la commande `docker-compose` et ses différentes options, notamment `ps` :

<img src="/images/orchestration-docker/docker-compose-ps.png">

On notera que les noms des containers sont générés dynamiquement.

# Crane

[Crane](https://github.com/michaelsauter/crane), de son côté, est indépendant de l’entreprise _Docker, Inc_.

Le principe de Crane est identique à celui de Docker Compose : un fichier de description (qui peut être en JSON ou en YAML) et une commande unique pour interagir avec les containers.

Ici, nous avons créé un fichier `crane.yml` :

{% highlight yaml %}
containers:

  redis:
    image: redis
    run:
      volume: ["/root/rss2twitter/data:/data"]
      publish: ["6379:6379"]
      cmd: "redis-server --appendonly yes"
      detach: true

  rss2twitter:
    dockerfile: app
    image: ippontech/rss2twitter
    run:
      link: ["redis:redis"]
      detach: true
{% endhighlight %}

Deux différences :

- on peut directement indiquer que les containers seront lancés en mode détachés via `detach: true`
- pour le container de l’application, il faut indiquer le nom de l’image qui sera générée.

On peut lancer nos containers via la commande crane lift :

<img src="/images/orchestration-docker/crane-lift.png">

Les autres options de lignes de commande sont relativement similaires à celles de Docker Compose :

<img src="/images/orchestration-docker/crane-help.png">

On peut notamment utiliser `crane status` pour connaître l’état de nos containers :

<img src="/images/orchestration-docker/crane-status.png">

Dans ce cas, on voit que Crane utilise comme noms de containers les noms que nous avons indiqués dans le fichier de définition.

# Bilan

Les deux outils remplissent bien leur rôle :

- sauvegarde des paramètres de la ligne de commande de Docker
- démarrage / arrêt / … de plusieurs containers en exécutant une seule commande
- orchestration de la création des containers en respectant l’ordre des dépendances.

Les outils ont toutefois leurs limites. Par exemple, si le container Redis s’arrête, il faudrait arrêter le container de l’application puis relancer les deux containers. Ni Docker Compose, ni Crane, ne sait faire cela.

Pour finir, j’ai une légère préférence pour Crane pour sa documentation plus lisible ainsi que pour sa commande plus facile à taper (on finit par créer un alias sur `docker-compose`).

La question qui se pose est probablement celle de savoir s’il y a, à terme, de la place pour deux outils remplissant les mêmes fonctions. Docker Compose étant maintenant l’outil “officiel”, il prendra probablement le pas sur Crane. Consolation, la migration de l’un vers l’autre sera très facile étant données les similitudes de formats de fichiers de configuration.