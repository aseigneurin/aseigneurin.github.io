---
layout: post
title:  "Introduction à Apache Spark"
date:   2014-10-29 11:00:00
tags: spark
language: FR
---
[Spark](http://spark.apache.org/) est un outil permettant de faire du traitement de larges volumes de données, et ce, de manière distribuée (cluster computing). Le framework offre un modèle de programmation plus simple que celui d'Hadoop et permet des temps d'exécution jusqu'à 100 fois plus courts.

<img src="/images/spark-logo.png" style="float:right; padding-left: 20px; padding-bottom: 20px; width: 258px"/>

Le framework a le vent en poupe (presque autant que Docker) et il est en train de remplacer Hadoop à vitesse grand V. Car, il faut l'admettre, Hadoop, dans son orientation stricte MapReduce, est en train de mourir.

Cet article est donc le premier d'une série visant à faire découvrir Spark, son modèle de programmation, ainsi que son écosystème. Le code présenté sera écrit en Java.

# Contexte

Spark est né en 2009 dans le laboratoire [AMPLab](https://amplab.cs.berkeley.edu/) de l'université de Berkeley en partant du principe que :

- d'une part, la RAM coûte de moins en moins cher et les serveurs en ont donc de plus en plus à disposition
- d'autre part, beaucoup de jeux de données dits "Big Data" ont une taille de l'ordre de 10 Go et ils tiennent donc en RAM.

Le projet a intégré l'incubateur Apache en juin 2013 et est devenu un "Top-Level Project" [en février 2014](https://blogs.apache.org/foundation/entry/the_apache_software_foundation_announces50).

La version 1.0.0 de Spark a été releasée [en mai 2014](http://spark.apache.org/news/spark-1-0-0-released.html) et le projet poursuit une évolution rapide. L'écosystème Spark comporte ainsi aujourd'hui plusieurs outils :

- Spark pour les traitements "en batch"
- Spark Streaming pour le traitement en continu de flux de données
- MLlib pour le "machine learning"
- GraphX pour les calculs de graphes
- Spark SQL, une implémentation SQL-like d'interrogation de données.

Le framework est écrit en Scala et propose un binding Java qui permet de l'utiliser sans problème en Java. Java 8 est toutefois recommandé pour exploiter les expressions lambdas qui permettront d'écrire un code lisible.

# Notions de base

L'élément de base que l'on manipulera est le RDD : _Resilient Distributed Dataset_.

Un RDD est une abstraction de collection sur laquelle les opérations sont effectuées de manière distribuée tout en étant tolérante aux pannes matérielles. Le traitement que l'on écrit semble ainsi s'exécuter au sein de notre JVM mais il sera découpé pour s'exécuter sur plusieurs noeuds. En cas de perte d'un noeud, le sous-traitement sera automatiquement relancé sur un autre noeud par le framework, sans que cela impacte le résultat.

Les éléments manipulés par le RDD (classes `JavaRDD`, `JavaPairRDD`...) peuvent être des objets simples (String, Integer...), nos propres classes, ou, plus couramment, des tuples (classe `Tuple2`). Dans ce dernier cas, les opérations offertes par l'API permettront de manipuler la collection comme une map clé-valeur.

L'API exposée par le RDD permet d'effectuer des transformations sur les données :

- `map()` permet de transformer un élément en un autre élément
- `mapToPair()` permet de transformer un élément en un tuple clé-valeur
- `filter()` permet de filtrer les éléments en ne conservant que ceux qui correspondent à une expression
- `flatMap()` permet de découper un élément en plusieurs autres éléments
- `reduce()` et `reduceByKey()` permet d'aggréger des éléments entre eux
- etc.

Ces transformations sont "lazy" : elles ne s'exécuteront que si une opération finale est réalisée en bout de chaîne. Les opérations finales disponibles sont :

- `count()` pour compter les éléments
- `collect()` pour récupérer les éléments dans une collection Java dans la JVM de l'exécuteur (dangereux en cluster)
- `saveAsTextFile()` pour sauver le résultat dans *des* fichiers texte (voir plus loin)
- etc.

Enfin, l'API permet de conserver temporairement un résultat intermédiaire grâce aux méthodes `cache()` (stockage en mémoire) ou `persist()` (stockage en mémoire ou sur disque, en fonction d'un paramètre).

# Premiers pas

Pour un premier exemple de code, nous allons exploiter des données [Open Data de la mairie de Paris](http://opendata.paris.fr/), en l'occurence [la liste des arbres d'alignement présents sur la commune de Paris](http://opendata.paris.fr/explore/dataset/arbresalignementparis2010/?tab=metas).

Le fichier CSV peut-être téléchargé [ici](http://opendata.paris.fr/explore/dataset/arbresalignementparis2010/download/?format=csv). Il comporte 103 589 enregistrements. En voici un extrait :

    geom_x_y;geom;genre_lati;genre_fran;variete;arbre_rema;circonfere;hauteur_m;date_mesur;lib_type_e;lib_etat_c;x;y
    48.8560632291, 2.34626065083;"{""type"": ""Point"", ""coordinates"": [2.346260650825456, 48.856063229114774]}";Platanus;Platane;;0;90.0;0.0;0001-01-01T00:00:00+00:00;Trottoir;grille fonte/acier ajourée;600715.8125;128400.648438
    48.8574478618, 2.3458179812;"{""type"": ""Point"", ""coordinates"": [2.345817981197062, 48.85744786180517]}";Platanus;Platane;;0;0.0;0.0;0001-01-01T00:00:00+00:00;Trottoir;Terre sable;600683.3125;128554.617188
    48.8329651219, 2.31476577319;"{""type"": ""Point"", ""coordinates"": [2.314765773191272, 48.8329651219174]}";Prunus;Cerisier à fleur;hisakura-new-red;0;0.0;0.0;0001-01-01T00:00:00+00:00;Trottoir;Stabilisé;598404.0;125832.320313

Ce fichier présente les caractéristiques suivantes :

- une ligne de header
- un enregistrement par ligne
- les champs d'un enregistrement sont séparés par un point-virgule.

Nous allons simplement compter les enregistrements pour lesquels la hauteur de l'arbre est renseignée et est supérieure à zéro.

Il faut d'abord créer un "contexte Spark". Puisque nous écrivons du Java, la classe que nous utilisons est `JavaSparkContext` et nous lui passons un objet de configuration contenant :

- un nom d'application (utile lorsque l'application est déployée en cluster)
- la référence vers un cluster Spark à utiliser, en l'occurence "local" pour exécuter les traitements au sein de la JVM courante.

{% highlight java %}
SparkConf conf = new SparkConf()
        .setAppName("arbres-alignement")
        .setMaster("local");
JavaSparkContext sc = new JavaSparkContext(conf);
{% endhighlight %}

Nous pouvons ensuite écrire la suite de traitements et récupérer le résultat :

{% highlight java %}
long count = sc.textFile("arbresalignementparis2010.csv")
        .filter(line -> !line.startsWith("geom"))
        .map(line -> line.split(";"))
        .map(fields -> Float.parseFloat(fields[7]))
        .filter(height -> height > 0)
        .count();
System.out.println(count);
{% endhighlight %}

Détaillons ce code :

- Nous commençons par demander à Spark de lire le fichier CSV. Spark sait nativement lire un fichier texte et le découper en lignes.

    La méthode utilisée est `textFile()` et le type retourné est `JavaRDD<String>` (un RDD de Strings).

        sc.textFile("arbresalignementparis2010.csv")

- Nous filtrons directement la première ligne (la ligne de header). Ce filtrage est effectué par le contenu plutôt que par le numéro de ligne. En effet, les éléments du RDD ne sont pas ordonnés puisqu'un fichier peut être lu par fragments, notamment lorsqu'il s'agit d'un gros fichier lu sur un cluster.

    La méthode utilisée est `filter()` et elle ne modifie par le type retourné qui reste donc `JavaRDD<String>`.

        .filter(line -> !line.startsWith("geom"))

- Les lignes peuvent ensuite être découpées en champs. Nous utilisons une expression lambda qui peut être lue de la façon suivante : pour chaque élément que nous appelerons `line`, retourne le résultat de l'expression `line.split(";")`.

    L'opération `map()` est utilisée  et le type retourné devient `JavaRDD<String[]>`.

        .map(line -> line.split(";"))

- Le champ contenant la hauteur de l'arbre peut ensuite être parsé. Nous ne conservons que cette valeur : les autres champs ne sont pas conservés.

    L'opération `map()` est à nouveau utilisée  et le type retourné devient `JavaRDD<Float>`.

        .map(fields -> Float.parseFloat(fields[7]))

- Nous filtrons ensuite les éléments pour ne conserver que les hauteurs supérieures à zéro.

    L'opération `filter()` est à nouveau utilisée. Le type reste donc `JavaRDD<Float>`.

        .filter(height -> height > 0)

- Enfin, nous comptons les éléments du RDD.

    L'opération finale `count()` est utilisée et un `long` est retourné.

        .count()

Voici un extrait de ce qui est produit sur la console :

    ...
    14/10/29 17:09:54 INFO FileInputFormat: Total input paths to process : 1
    14/10/29 17:09:54 INFO SparkContext: Starting job: count at FirstSteps.java:26
    14/10/29 17:09:54 INFO DAGScheduler: Got job 0 (count at FirstSteps.java:26) with 1 output partitions (allowLocal=false)
    ...
    14/10/29 17:09:54 INFO TaskSetManager: Starting task 0.0 in stage 0.0 (TID 0, localhost, PROCESS_LOCAL, 1242 bytes)
    14/10/29 17:09:54 INFO Executor: Running task 0.0 in stage 0.0 (TID 0)
    14/10/29 17:09:54 INFO HadoopRDD: Input split: file:/Users/aseigneurin/dev/spark-samples/arbresalignementparis2010.csv:0+25008725
    ...
    14/10/29 17:09:55 INFO SparkContext: Job finished: count at FirstSteps.java:26, took 0.475835815 s
    6131

Spark a exécuté les traitements en local, au sein de la JVM.

Le fichier a été lu en un seul bloc. En effet, celui-ci fait 25 Mo et, par défaut, Spark découpe les fichiers en blocs de 32 Mo.

Le résultat (6131) est obtenu en moins d'une demi-seconde. Ce temps d'exécution n'est pas, en soi, impressionant, mais nous verrons la puissance du framework lorsque nous manipulerons des fichiers plus volumineux.

# Conclusion

Le code écrit avec Spark présente l'intérêt d'être à la fois compact et lisible. Nous verrons dans les prochains posts qu'il est possible de manipuler des volumes très importants de données, même pour des opérations manipulant l'ensemble du dataset, et ce, sans devoir modifier le code.
