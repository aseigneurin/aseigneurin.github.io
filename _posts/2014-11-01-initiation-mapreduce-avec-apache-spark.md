---
layout: post
title:  "Initiation au MapReduce avec Apache Spark"
date:   2014-11-01 11:00:00
tags: spark mapreduce
language: FR
---
Dans [le précédent post](/2014/10/29/introduction-apache-spark.html), nous avons utilisé l'opération _Map_ qui permet de transformer des valeurs à l'aide d'une fonction de transformation. Nous allons maintenant découvrir l'opération _Reduce_ qui permet de faire des aggrégations. Nous allons ainsi pouvoir faire du _MapReduce_ de la même manière qu'avec Hadoop.

# La théorie

Avec Spark comme avec Hadoop, une opération de _Reduce_ est une opération qui permet d'agréger les valeurs **deux à deux**, en procédant par autant d'étapes que nécessaire pour traiter l'ensemble des éléments de la collection. C'est ce qui permet au framework de réaliser des agrégations en parallèle, éventuellement sur plusieurs noeuds.

Le framework va choisir deux éléments et les passer à une fonction que nous allons définir. La fonction doit retourner le nouvel élément qui remplacera les deux premiers.

Il en découle que le type en sortie de notre fonction doit être identique au type reçu en entrée : **les valeurs doivent être homogènes**. C'est nécessaire pour que l'opération soit répétée jusqu'à ce que l'ensemble des éléments aient été traités.

Avec Spark, il existe deux types d'opérations de réduction :

- `reduce()` opère sur les éléments, quel que soit leur type, et retourne une unique valeur.
- `reduceByKey()` opère sur des valeurs associées à une même clé. Ce type d'opération n'est possible que sur des RDD de type `JavaPairRDD` (une liste de tuples clé-valeur), et elle produit un résultat qui est lui aussi un `JavaPairRDD` mais dans lequel chaque clé n'apparait qu'une fois (équivalent à une Map clé-valeur).

Nous allons étudier l'opération `reduce()` dans ce post. L'opération `reduceByKey()` sera traité [dans le prochain post](/2014/11/06/mapreduce-et-manipulations-par-cles-avec-apache-spark.html).

# Agrégation de valeurs en MapReduce

Dans notre [premier exemple de code](/2014/10/29/introduction-apache-spark.html), nous avions construit un RDD contenant les hauteurs des arbres de la commune de Paris. Nous utilisions ensuite l'opération `count()` pour compter les arbres ayant une hauteur définie.

{% highlight java %}
long count = sc.textFile("arbresalignementparis2010.csv")
        .filter(line -> !line.startsWith("geom"))
        .map(line -> line.split(";"))
        .map(fields -> Float.parseFloat(fields[3]))
        .filter(height -> height > 0)
        .count();
{% endhighlight %}

Plutôt que de compter les hauteurs non nulles, nous pouvons utiliser une opération de réduction pour calculer la hauteur totale des arbres.

Il nous faut une fonction qui va recevoir deux hauteurs et retourner la somme de ces deux hauteurs. Autrement dit, une fonction qui reçoit deux paramètres de type `Float` et qui retourne une valeur de type `Float` :

{% highlight java %}
private Float sum(Float x, Float y) {
    return x + y;
}
{% endhighlight %}

Cette fonction peut s'écrire sous la forme d'une expression lambda :

{% highlight java %}
(x, y) -> x + y
{% endhighlight %}

Nous pouvons ainsi écrire :

{% highlight java %}
float totalHeight = sc.textFile("arbresalignementparis2010.csv")
        ...
        .reduce((x, y) -> x + y);
{% endhighlight %}

Le framework va appeler notre fonction de réduction jusqu'à ce que toutes les valeurs aient été traitées.

# Comptage en _MapReduce_

L'opération `count()` vue plus tôt peut également s'écrire avec une opération de réduction.

Puisque le résultat attendu est un nombre d'arbres (type `ìnt` iou `Integer`), il faut d'abord convertir les éléments en entrée (type `Float`) afin de manipuler des données homogènes.

Nous allons donc procéder en deux étapes :

- une opération `map()` pour convertir chaque élément dans la valeur _1_
- une opération de `reduce()` pour additioner les _1_ entre eux.

Nous écrivons ainsi :

{% highlight java %}
long count = sc.textFile("data/arbresalignementparis2010.csv")
        ...
        .map(item -> 1)
        .reduce((x, y) -> x + y);
{% endhighlight %}

Notez que l'expression lamba utilisée dans la fonction `reduce()` est identique à celle utilisée plus tôt, mais que la fonction équivalente manipule des `Integer` :

{% highlight java %}
private Integer sum(Integer x, Integer y) {
    return x + y;
}
{% endhighlight %}

# Calcul de la hauteur moyenne des arbres

Nous pouvons calculer la hauteur moyenne des arbres de notre fichier en réalisant :

- une opération d'agrégation (somme) des hauteurs
- un comptage des éléments
- une division de la première valeur par la deuxième.

Puisque l'agrégation et le comptage se basent sur le même fichier et que les premières opérations de traitement sont identiques, nous pouvons réutiliser le même RDD. Nous allons utiliser l'opération `cache()` qui permet de mettre le RDD en cache en mémoire. Les calculs ne seront donc exécutés qu'une seule fois et le résultat intermédiaire sera directement utilisé pour les deux opérations d'agrégation et de comptage.

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

# Conclusion

Spark permet de réaliser des agrégations sur l'ensemble des valeurs via une opréation de _Reduce_ opérant sur les éléments deux à deux, et ce, afin de réalisée l'opération de manière distribuée.

Nous verrons [dans le prochain post](/2014/11/06/mapreduce-par-cles-avec-apache-spark.html) comment réaliser des opérations d'agrégation par clés.

---

Vous pouvez retrouver l'exemple complet de code [sur GitHub](https://github.com/aseigneurin/spark-sandbox).

**Vous aimez cette série d'articles sur Spark et souhaitez faire découvrir l'outil à vos équipes ? [Invitez-moi pour un Brown Bag Lunch](http://www.brownbaglunch.fr/baggers.html#Alexis_Seigneurin_Paris) !**