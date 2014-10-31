---
layout: post
title:  "Initiation au MapReduce avec Apache Spark"
date:   2014-11-06 11:00:00
tags: spark
language: FR
---
Dans [le précédent post](/2014/10/29/introduction-apache-spark.html), nous avons utilisé l'opération _Map_ qui permet de transformer des valeurs à l'aide du fonction de transformation. Nous allons maintenant découvrir l'opération _Reduce_ qui permet de faire des aggrégations. Nous allons ainsi pouvoir faire du _MapReduce_ de la même manière qu'avec Hadoop.

# Comptage en _MapReduce_

Dans notre premier exemple de code, nous utilisions l'opération `count()` pour compter les éléments de notre RDD :

{% highlight java %}
long count = sc.textFile("arbresalignementparis2010.csv")
        ...
        .count();
{% endhighlight %}

Nous pouvons réécrire cette opération avec un `map()` et un `reduce()` :

- Puisque nous devons compter les éléments, il faut convertir chaque élément dans la valeur _1_.
- Nous pouvons ensuite additioner les _1_ entre eux.

{% highlight java %}
long count = sc.textFile("data/arbresalignementparis2010.csv")
        ...
        .map(item -> 1)
        .reduce((x, y) -> x + y);
{% endhighlight %}

Le point important est que l'opération `reduce()` est réalisée en cascade, jusqu'à ce qu'il n'y ait plus qu'une seule valeur. Par exemple :

- valeures initiales : (1), (1), (1), (1), (1)
- première passe : (2), (2), (1)
- deuxième passe : (4), (1)
- troisième passe : (5)

Le type en sortie de `reduce()` doit donc être identique au type reçu en entrée : **les valeurs doivent être homogènes**. Dans notre exemple, il était donc essentiel de convertir les éléments (hauteurs d'arbres, type `Float`) en nombres d'éléments (_1_, type _Integer_).

# Agrégation de valeurs en MapReduce

L'intérêt d'écrire soi-même le _MapReduce_ pour compter les éléments est bien entendu limité. Nous pouvons toutefois imaginer calculer la hauteur moyenne des arbres de notre fichier par :

- une opération d'agrégation (somme) des hauteurs
- un comptage des éléments
- une division de la première valeur par la deuxième.

Puisque les deux opérations se basent sur le même fichier et que les premières opérations de traitement sont identiques, nous pouvons réutiliser le même RDD. Nous utilisons par ailleurs l'opération `cache()` qui permet de mettre le RDD en cache en mémoire. Les calculs ne seront donc exécutés qu'une seule fois et le résultat intermédiaire sera directement utilisé pour les deux opérations d'agrégation et de comptage.

{% highlight java %}
JavaRDD<Float> rdd = sc.textFile("data/arbresalignementparis2010.csv")
        .filter(line -> !line.startsWith("geom"))
        .map(line -> line.split(";"))
        .map(fields -> Float.parseFloat(fields[3]))
        .filter(height -> height > 0)
        .cache();

float totalHeight = rdd.reduce((x, y) -> x + y);

long count = rdd.count();

System.out.println("Total height: " + totalHeight);
System.out.println("Count: " + count);
System.out.println("Average height " + totalHeight / count);
{% endhighlight %}

Le résultat obtenu est le suivant :

    Count: 32112
    Total height: 359206.0
    Average height 11.186036