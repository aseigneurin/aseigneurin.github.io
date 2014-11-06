---
layout: post
title:  "MapReduce et manipulation de RDD par clés avec Apache Spark"
date:   2014-11-06 11:00:00
tags: spark mapreduce
language: FR
---
Nous avons vu dans [le post précédent](/2014/11/01/initiation-mapreduce-avec-apache-spark.html) qu'Apache Spark permettait de réaliser des opérations d'agrégation sur l'ensemble des valeurs d'un RDD. Nous allons maintenant réaliser des agrégations ainsi que d'autres manipulations *par clés*.

# La théorie

Une opération de réduction par clés effectue une agrégation des valeurs pour chaque clé du RDD. Ce type d'opération ne peut être effectué que sur un RDD de type `JavaPairRDD`, c'est-à-dire un RDD dans lequel les éléments sont des tuples clé-valeur. Attention, contrairement à une Map en Java, il n'existe aucune contrainte d'unicité sur les clés. Plusieurs tuples portant la même clé peuvent donc exister dans le RDD.

L'opération de réduction va être effectuée sur des valeurs de la même clé jusqu'à ce qu'il n'existe plus qu'une seule valeur par clé. Le RDD résultant sera donc une vraie Map clé-valeur, avec unicité des clés.

Supposons qu'on ait le RDD suivant (tuples clé-valeur) :

- (A, 3)
- (A, 5)
- (B, 2)
- (A, 4)
- (B, 7)

Si on applique une réduction par clés calculant la somme des valeurs, on obtiendra le RDD suivant :

- (A, 12)
- (B, 9)

Le RDD contient autant de tuples qu'il existait de clés différentes dans le RDD d'origine.

# Les données d'exemple

Nous allons utiliser des fichiers de statistiques de consultation de Wikipedia. Ces fichiers sont en [libre téléchargement](https://dumps.wikimedia.org/other/pagecounts-raw/), un fichier étant produit chaque heure. Chaque fichier pèse environ 300 Mo une fois décompressé.

Voici un extrait de fichier :

    fr Louvoy%C3%A8rent 1 7766
    fr Louvoyer 2 24276
    fr Louvre-Lens 1 39497
    fr Louvres 1 36541
    fr Louxor 2 33183

Chaque ligne représente un enregistrement selon 4 champs séparés par des espaces :

1. Le nom du "projet" Wikipedia : code pays suivi, éventuellement, d'un suffixe indiquant s'il s'agit de Wikipedia, Wikibooks, Wiktionary, etc.
1. Le titre de la page, URL encodé.
1. Le nombre de requêtes reçues.
1. La taille de la page en octets.

# En pratique

A partir des statistiques Wikipedia, nous pouvons calculer le nombre de visites par projet. Contrairement à une agrégation globale, nous cherchons donc à obtenir une liste clé-valeur : code projet - nombre de visites.

Commençons par lire un fichier par lignes et en découpant chaque ligne selon les espaces :

{% highlight java %}
sc.textFile("data/wikipedia-pagecounts/pagecounts-20141101-000000")
        .map(line -> line.split(" "))
{% endhighlight %}

Le type obtenu est `JavaRDD<String[]>`.

Nous pouvons transformer ce `JavaRDD` en `JavaPairRDD` via l'opération `mapToPair()`. Il faut alors renvoyer des objets de type `Tuple2` :

{% highlight java %}
        .mapToPair(s -> new Tuple2<String, Long>(s[0], Long.parseLong(s[2])))
{% endhighlight %}

La classe `JavaPairRDD` offre des transformations permettant de travailler nativement sur cette collection clé-valeur : `reduceByKey()`, `sortByKey()`, ainsi que des fonctions de croisement entre deux `JavaPairRDD` (`join()`, `intersect()`, etc.).

En l'occurence, nous allons utiliser la fonction `reduceByKey()` en lui donnant une opération de somme. Les valeurs reçues par la fonction appartiendront à la même clé sans que l'on puisse connaître celle-ci :

{% highlight java %}
        .reduceByKey((x, y) -> x + y)
{% endhighlight %}

Enfin, nous pouvons écrire l'ensemble des tuples sur la console. La clé du tuple est représentée par le champ `_1` tandis que la valeur est représentée par le champ `_2`.

{% highlight java %}
        .foreach(t -> System.out.println(t._1 + " -> " + t._2))
{% endhighlight %}

Voici le code complet :

{% highlight java %}
SparkConf conf = new SparkConf()
        .setAppName("wikipedia-mapreduce-by-key")
        .setMaster("local");
JavaSparkContext sc = new JavaSparkContext(conf);

sc.textFile("data/wikipedia-pagecounts/pagecounts-20141101-000000")
        .map(line -> line.split(" "))
        .mapToPair(s -> new Tuple2<String, Long>(s[0], Long.parseLong(s[2])))
        .reduceByKey((x, y) -> x + y)
        .foreach(t -> System.out.println(t._1 + " -> " + t._2));
{% endhighlight %}

A l'exécution, nous obtenons un résultat similaire à ce qui suit :

    ...
    got.mw -> 14
    mo.d -> 1
    eo.q -> 38
    fr -> 602583
    ja.n -> 167
    mus -> 21
    xal -> 214
    ...

La valeur 3269849 pour Wikipedia France est donc la somme des nombre de visites des pages recensées en "fr" dans le fichier.

# Tri des résultats par clés

Nous pouvons remarquer que les résultats ne sont pas triés. En effet, pour des raisons de performance, Spark ne garantit pas l'ordre au sein du RDD : les tuples sont traités de manière indépendante les uns des autres.

Nous pouvons trier les tuples par leur clé grâce à la méthode `sortByKey()` qui prend éventuellement un booléen en paramètre pour inverser le tri :

{% highlight java %}
        .sortByKey()
{% endhighlight %}

Le résultat devient :

    AR -> 195
    De -> 115
    EN -> 4
    En -> 10
    En.d -> 8
    FR -> 1
    It -> 2
    SQ.mw -> 11
    Simple -> 1
    aa -> 27
    aa.b -> 6
    aa.d -> 1
    aa.mw -> 11
    ...

Le tri est *case-sensitive*. Si nous voulons trier de manière *case-insensitive*, nous pouvons passer un comparateur.

Malheureusement, nous ne pouvons pas utiliser un comparateur issu de `Comparator.comparing()` (nouveauté Java 8) car le comparateur retourné n'est pas sérialisable.

{% highlight java %}
// génère une exception :
//    Task not serializable: java.io.NotSerializableException
.sortByKey(Comparator.comparing(String::toLowerCase))
{% endhighlight %}

Il faut donc avoir recours à un comparateur implémentant l'interface `Serializable` :

{% highlight java %}
class LowerCaseStringComparator implements Comparator<String>, Serializable {
    @Override
    public int compare(String s1, String s2) {
        return s1.toLowerCase().compareTo(s2.toLowerCase());
    }
}
{% endhighlight %}

Ce comparateur est alors utilisé de manière plus classique :

{% highlight java %}
Comparator<String> c = new LowerCaseStringComparator();

...
        .sortByKey(c)
{% endhighlight %}

On obtient alors le résultat souhaité :

    ...
    ang.q -> 15
    ang.s -> 9
    AR -> 195
    ar -> 108324
    ar.b -> 293
    ...

# Tri des résultats par valeurs

La classe `JavaPairRDD` possède une méthode `sortByKey()` mais il n'existe pas de méthode `sortByValue()`.  Si l'on souhaite trier par valeurs, il faut inverser nos tuples pour que les valeurs soient les clés.

Pour rappel, un `JavaPairRDD` n'impose pas que les clés des tuples soient uniques au sein du RDD. On peut donc avoir des doublons de valeurs sans que cela pose problème.

Nous inversons donc les tuples, toujours avec la fonction `mapToPair()` de sorte à récupérer un `JavaPairRDD` en sortie :

{% highlight java %}
.mapToPair(t -> new Tuple2<Long, String>(t._2, t._1))
{% endhighlight %}

Nous pouvons alors trier le RDD par ordre descendant (les plus grandes valeurs en premier) et conserver les 10 premiers éléments grâce à la méthode `take()` :

{% highlight java %}
.sortByKey(false)
.take(10)
{% endhighlight %}

Notez que `take()` retourne une collection Java (`java.util.List`) et non un RDD. La méthode `forEach()` que nous utilisons est donc celle de l'API de collections, et non `foreach()` sur un RDD :

{% highlight java %}
.forEach(t -> System.out.println(t._2 + " -> " + t._1));
{% endhighlight %}

Le code de tri :

{% highlight java %}
.mapToPair(t -> new Tuple2<Long, String>(t._2, t._1))
.sortByKey(false)
.take(10)
.forEach(t -> System.out.println(t._2 + " -> " + t._1));
{% endhighlight %}

On obtient alors le top 10 des projets Wikipedia les plus consultés dans l'heure :

    meta.m -> 15393394
    meta.mw -> 12390990
    en -> 7209054
    en.mw -> 4405366
    es -> 1210216
    de -> 692501
    ja.mw -> 674700
    es.mw -> 666607
    ru -> 664970
    ja -> 637371

# Traitement de multiples fichiers

Pour finir, notez qu'il est possible de traiter plusieurs fichiers grâce à l'utilisation de wildcards en paramètre de la méthode `textFile()`. Nous pouvons ainsi agréger des données provenant de plusieurs fichiers.

Nous en profitons pour indiquer à l'exécuteur Spark d'utiliser 16 threads via `setMaster("local[16]")`.

Ainsi, le calcul du top 10 des projets Wikipedia pour la journée du 1er novembre (24 fichiers) devient :

{% highlight java %}
SparkConf conf = new SparkConf()
        .setAppName("wikipedia-mapreduce-by-key")
        .setMaster("local[16]");
JavaSparkContext sc = new JavaSparkContext(conf);

sc.textFile("data/wikipedia-pagecounts/pagecounts-20141101-*")
        .map(line -> line.split(" "))
        .mapToPair(s -> new Tuple2<String, Long>(s[0], Long.parseLong(s[2])))
        .reduceByKey((x, y) -> x + y)
        .mapToPair(t -> new Tuple2<Long, String>(t._2, t._1))
        .sortByKey(false)
        .take(10)
        .forEach(t -> System.out.println(t._2 + " -> " + t._1));
{% endhighlight %}

Le résultat obtenu est le suivant :

    meta.m -> 430508482
    meta.mw -> 313310187
    en -> 171157735
    en.mw -> 103491223
    ru -> 29955421
    ja -> 22201812
    ja.mw -> 22186094
    es -> 21424067
    de -> 21098513
    fr -> 17967662

Sur un MacBook Pro fin 2013 (SSD, Core i7 à 2 GHz), ce traitement prend environ 40 secondes pour près de 9 Go de données en entrée. C'est très raisonnable.

# Conclusion

Nous commençons à avoir un bon aperçu de l'API de Spark et des possibilités offertes par celle-ci. La manipulation de tuples clé-valeur sera notamment un des usages principaux du framework.

Dans les prochains posts, nous déploierons un cluster Spark et étudierons l'API de streaming.

---

Vous pouvez retrouver l'exemple complet de code [sur GitHub](https://github.com/aseigneurin/spark-sandbox).

**Vous aimez cette série d'articles sur Spark et souhaitez faire découvrir l'outil à vos équipes ? [Invitez-moi pour un Brown Bag Lunch](http://www.brownbaglunch.fr/baggers.html#Alexis_Seigneurin_Paris) !**