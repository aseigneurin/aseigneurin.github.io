---
layout: post
title:  "Introduction à Spark Streaming"
date:   2014-12-07 11:00:00
tags: spark streaming
language: FR
---
Spark permet de traiter des données qui sont figées à un instant _T_. Grâce au module Spark Streaming, il est possible de traiter des flux de données qui arrivent en continu, et donc de traiter ces données au fur et à mesure de leur arrivée.

# Modèle de micro-batches

Avec Spark Streaming, un _contexte_ est initialisé avec une durée. Le framework va accumuler des données pendant cette durée puis produire un petit RDD (_Resilient Distributed Dataset_, Cf. [Introduction à Apache Spark](/2014/10/29/introduction-apache-spark.html)). Ce cycle accumulation / production de RDD va se reproduire jusqu'à ce le programme soit arrêté. On parle ici de micro-batches par opposition à un traitement des évènements un par un.

<p style="text-align:center;"><img src="/images/spark-streaming-micro-batches.png" style="width:75%"></p>

Spark Streaming s'oppose donc ici à [Apache Storm](https://storm.apache.org/) : Storm offre un traitement en temps-réel des évènements tandis que Spark Streaming ajoutera un délai entre l'arrivée d'un message et son traitement.

Cette différence de traitement permet toutefois à Spark Streaming d'offrir une garantie de traitement des messages en _exactly once_ en conditions normales (chaque message est délivré une et une seule fois au programme, sans perte de messages), et _at least once_ en conditions dégradées (un message peut être délivré plusieurs fois, mais toujours sans pertes). Storm permet, lui, de régler le niveau de garantie mais, pour optimiser les performances, le mode _at most once_ (chaque message est délivré au maximum une fois mais des pertes sont possibles) doit être utilisé.

Enfin, et c'est l'avantage principal de Spark Streaming par rapport à Storm, **l'API de Spark Streaming est identique à l'API classique de Spark**. Il est ainsi possible de manipuler des flux de données de la même manière que l'on manipule des données figées.

# Sources de données

Spark Streaming étant destiné à traiter des données qui arrivent en continu, il est nécessaire de choisir une source de données adaptée. On aura donc tendance à préférer des sources ouvrant une socket réseau et restant en écoute. De base, on pourra ainsi utiliser :

- une socket TCP (via `sc.socketStream` ou `sc.socketTextStream`)
- des messages depuis [Kafka](http://kafka.apache.org/)
- des logs depuis [Flume](http://flume.apache.org/)
- des fichiers depuis HDFS (pour monitorer la création de nouveaux fichiers uniquement)
- une file MQ (type [ZeroMQ](http://zeromq.org/))
- des tweets depuis Twitter (utilise l'API [Twitter4J](http://twitter4j.org/en/index.html))

Il est également possible d'implémenter une source de données sur mesure en étendant la classe [Receiver](http://spark.apache.org/docs/latest/api/java/org/apache/spark/streaming/receiver/Receiver.html).

# Exemple

## Initialisation de Spark Streaming

Un contexte Spark Streaming est créé en instanciant la classe `JavaStreamingContext` (et plus `JavaSparkContext`). Il est alors nécessaire d'indiquer une durée de _discrétisation_ en millisecondes. Cette durée indiquera la cadence à laquelle les micro-batches seront produits.

{% highlight java %}
SparkConf sparkConf = new SparkConf()
        .setAppName("my streaming app")
        .setMaster("local[2]");
JavaStreamingContext sc = new JavaStreamingContext(sparkConf, new Duration(5000));
{% endhighlight %}

Notez qu'il est important d'initialiser l'exécuteur avec **au minimum 2 threads** (`local[2]`). En effet, un thread sera dédié à l'écoute des données entrantes et il faut au moins un thread de traitement. Sans cela, l'application bloquera après l'émission du premier batch.

## Création d'un flux

Dans cet exemple, nous allons _consommer_ des tweets. Twitter est en effet une source de données facile à exploiter puisque des tweets sont produits en permanence.

Au préalable, il faut [déclarer une application liée à un compte Twitter](https://apps.twitter.com/), récupérer des clés d'API et les placer dans [un fichier twitter4j.properties](http://twitter4j.org/en/configuration.html). La librairie Twitter4J est alors initialisé de la manière suivante :

{% highlight java %}
Configuration twitterConf = ConfigurationContext.getInstance();
Authorization twitterAuth = AuthorizationFactory.getInstance(twitterConf);
{% endhighlight %}

L'objet de base de l'API Spark Streaming est un `DStream`, c'est-à-dire un _Discretized Stream_ (flux discrétisé). Pour Twitter4J, un `DStream` est créé via la méthode `TwitterUtils.createStream()` :

{% highlight java %}
String[] filters = ...;
TwitterUtils.createStream(sc, twitterAuth, filters)
        ...
{% endhighlight %}

On obtient un objet de type `JavaDStream<Status>` (`Status` étant une classe de la librairie Twitter4J) qui offre les opérations classiques : `map`, `filter`, etc.

Nous pouvons récupérer les tweets contenant le hashtag _#Android_ et compter les autres hashtags :

{% highlight java %}
String[] filters = { "#Android" };
TwitterUtils.createStream(sc, twitterAuth, filters)
        .flatMap(s -> Arrays.asList(s.getHashtagEntities()))
        .map(h -> h.getText().toLowerCase())
        .filter(h -> !h.equals("android"))
        .countByValue()
        .print();
{% endhighlight %}

Détaillons ce code :

- On commence par créer le `DStream` afin d'obtenir les tweets mentionnant le hashtag souhaité :

        String[] filters = { "#Android" };
        TwitterUtils.createStream(sc, twitterAuth, filters)

- On extrait les hashtags via la méthode `Status.getHashtagEntities()` et on [applatit le résultat](http://martinfowler.com/articles/collection-pipeline/flat-map.html) grâce à l'opération `flatMap`. A ce stade, nous avons un `JavaDStream<HashTagEntity>`.

        .flatMap(s -> Arrays.asList(s.getHashtagEntities()))

- On extrait ensuite le texte des hashtags des entités `HashTagEntity` par une opération `map` afin d'obtenir un `JavaDStream<String>`.

        .map(h -> h.getText().toLowerCase())

- Le hashtag _#Android_ ne nous intéresse pas puisqu'il est inclus dans tous les tweets. Nous l'éliminons donc grâce à un `filter`.

        .filter(h -> !h.equals("android"))

- On effectue ensuite un map-reduce pour compter les hashtags et ainsi obtenir un `JavaPairDStream<String, Long>` qui associe des hashtags à un nombre d'occurences.

        .countByValue()

    L'opération `countByValue` est équivalente au code suivant :

        .mapToPair(h -> new Tuple2<String, Long>(h, 1L))
        .reduceByKey((x, y) -> x + y)

- Enfin, nous utilisons l'opération `print` pour écrire sur la console 10 éléments. Cette opération est à réserver au debug.

        .print();

## Lancement du traitement

Pour lancer notre traitement, contrairement à Spark où l'exécution d'une action finale lançait un traitement, Spark Streaming nécessite d'appeler la méthode `sc.start()`.

Le traitement démarre dans des threads séparés. Il faut donc empêcher le thread principal de s'arrêter. Le plus simple est d'appeler `sc.awaitTermination()` mais il est possible d'attendre un évènement particulier (la pression d'une touche du clavier, par exemple) ou d'exécuter un autre traitement.

Nous écrivons donc :

{% highlight java %}
sc.start();
sc.awaitTermination();
{% endhighlight %}

En lançant notre application, on obtient, toutes les 5 secondes, l'affichage des nombres de hashtags mentionnés dans les tweets portant la mention _#Android_ :

    2014-12-07 13:23:10,064 [pool-8-thread-1] INFO  org.apache.spark.SparkContext - Job finished: getCallSite at DStream.scala:294, took 0.005259701 s
    -------------------------------------------
    Time: 1417954990000 ms
    -------------------------------------------
    (modeon,1)
    (skydragon,6)
    (igers,1)
    (androidgames,16)
    (gameinsight,12)
    (note4,1)
    (newphone,1)
    (gameinsightbr,4)
    (igersphilippines,1)
    (htc,6)
    ...
    
    ...
    2014-12-07 13:23:15,069 [pool-8-thread-1] INFO  org.apache.spark.SparkContext - Job finished: getCallSite at DStream.scala:294, took 0.007432635 s
    -------------------------------------------
    Time: 1417954995000 ms
    -------------------------------------------
    (skydragon,9)
    (androidgames,13)
    (gameinsight,13)
    (htc,9)
    (dubai,6)

# Conclusion

Spark Streaming répond à la problématique de traitement de données produites en flux continu, par opposition à Spark qui permet de traiter des données connues à un instant donné.

En pratique, on préfèrera Spark Streaming à Storm si le traitement en temps réel des données ne s'impose pas. Spark Streaming sera adapté si la cadence de traitement est au minimum de quelques dixièmes de secondes, voire de quelques secondes.

# Ressources

- [Spark Streaming Programming Guide](http://spark.apache.org/docs/latest/streaming-programming-guide.html)
- [Apache Storm vs. Apache Spark](http://www.zdatainc.com/2014/09/apache-storm-apache-spark/)

---

Vous pouvez retrouver l'exemple complet de code [sur GitHub](https://github.com/aseigneurin/spark-sandbox).

**Vous aimez cette série d'articles sur Spark et souhaitez faire découvrir l'outil à vos équipes ? [Invitez-moi pour un Brown Bag Lunch](http://www.brownbaglunch.fr/baggers.html#Alexis_Seigneurin_Paris) !**
