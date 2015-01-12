---
layout: post
title:  "Introduction à Spark SQL"
date:   2015-01-12 11:00:00
tags: spark sql
language: FR
---
Spark permet de manipuler d'importants volumes de données en utilisant une API de bas niveau. Pour simplifier l'exploration des données, Spark SQL offre une API de plus haut niveau avec une syntaxe SQL. Spark SQL permet ainsi de réaliser, très rapidement, de nombreuses opérations sans écrire de code.

Tous types de données peuvent être exploités avec Spark SQL dès lors que ces données sont tabulaires. Dans ce post, nous allons ainsi exploiter un fichier CSV listant les arbres présents sur la commune de Paris ([source](http://opendata.paris.fr/explore/dataset/arbresalignementparis2010/)) :

    geom_x_y;circonfere;adresse;hauteurenm;espece;varieteouc;dateplanta
    48.8648454814, 2.3094155344;140.0;COURS ALBERT 1ER;10.0;Aesculus hippocastanum;;
    48.8782668139, 2.29806967519;100.0;PLACE DES TERNES;15.0;Tilia platyphyllos;;
    48.889306184, 2.30400164126;38.0;BOULEVARD MALESHERBES;0.0;Platanus x hispanica;;
    48.8599934405, 2.29504883623;65.0;QUAI BRANLY;10.0;Paulownia tomentosa;;1996-02-29
    ...

# Préparation des données

Une étape de préparation des données est nécessaire pour permettre à l'interpréteur SQL de connaître les données. Le concept de RDD est réutilisé et nécessite simplement d'être enrichi d'un schéma. La classe manipulée devient un `JavaSchemaRDD`. Deux options existent pour construire un  `JavaSchemaRDD` :

1. En utilisant le type générique `Row` et en décrivant le schéma manuellement.
2. En utilisant des types personnalisés et en laissant Spark SQL découvrir le schéma par réflexion.

Pour étudier ces deux options, nous allons utiliser un RDD constitué en lisant le fichier des arbres, en retirant la première ligne et découpant chaque ligne en champs :

{% highlight java %}
JavaSparkContext sc = new JavaSparkContext("local", "arbres");

JavaRDD<String[]> trees = sc
        .textFile("data/arbresalignementparis2010.csv")
        .filter(line -> !line.startsWith("geom"))
        .map(line -> line.split(";", -1));
{% endhighlight %}

## Option 1 : description programmatique des données

Nous allons utiliser le type générique `Row` pour _wrapper_ nos données. Puisqu'il s'agit de transformer des données (en l'occurrence, un tableau de chaines de caractères en objets `Row`), il faut utiliser une opération _map_.

L'objet `Row` peut contenir un nombre indéterminé de champs. Notez toutefois que les conversions de types doivent être effectuées par l'application.

Nous créons ainsi des records avec la hauteur (champs "hauteurenm" parsé en type flottant) et le type d'arbre (champs "espece") :

{% highlight java %}
JavaRDD<Row> rdd = trees.map(fields -> Row.create(Float.parseFloat(fields[3]), fields[4]));
{% endhighlight %}

Nous pouvons ensuite construire un schéma en décrivant les champs. Il faut alors nommer les champs et indiquer leurs types en respectant l'ordre de leur présence dans le type `Row`.

Les types de données disponibles sont : `BinaryType`, `BooleanType`, `ByteType`, `DateType`, `DoubleType`, `FloatType`, `IntegerType`, `LongType`, `ShortType` et `TimestampType`.

{% highlight java %}
List<StructField> fields = new ArrayList<StructField>();
fields.add(DataType.createStructField("hauteurenm", DataType.FloatType, false));
fields.add(DataType.createStructField("espece", DataType.StringType, false));

StructType schema = DataType.createStructType(fields);
{% endhighlight %}

Il faut ensuite appliquer le schéma à notre RDD en ayant, au préalable, construit un contexte d'exécution (type `JavaSQLContext`) :

{% highlight java %}
JavaSQLContext sqlContext = new JavaSQLContext(sc);
JavaSchemaRDD schemaRDD = sqlContext.applySchema(rdd, schema);
{% endhighlight %}

L'ultime étape consiste à nommer le `JavaSchemaRDD` pour qu'il puisse être utilisé en SQL :

{% highlight java %}
schemaRDD.registerTempTable("tree");
{% endhighlight %}

## Option 2 : inférence de schéma par réflexion

Spark SQL permet aussi de déterminer le schéma d'un RDD en découvrant les types par réflexion. Il faut, dans ce cas, utiliser des POJO plutôt que le type générique `Row`.

Nous créons un POJO `Tree` :

{% highlight java %}
public class Tree {
    private float hauteurenm;
    private String espece;

    public Tree(float hauteurenm, String espece) {
        this.hauteurenm = hauteurenm;
        this.espece = espece;
    }

    public float getHauteurenm() { return hauteurenm; }
    public void setHauteurenm(float hauteurenm) { this.hauteurenm = hauteurenm; }

    public String getEspece() { return espece; }
    public void setEspece(String espece) { this.espece = espece; }
}
{% endhighlight %}

L'opération de conversion est similaire à celle utilisée plus haut :

{% highlight java %}
JavaRDD<Tree> rdd = trees.map(fields -> new Tree(Float.parseFloat(fields[3]), fields[4]));
{% endhighlight %}

On peut ensuite directement obtenir un `JavaSchemaRDD` en appliquant le schéma inféré à partir du type `Tree` :

{% highlight java %}
JavaSchemaRDD schemaRDD = sqlContext.applySchema(rdd, Tree.class);
{% endhighlight %}

Dans ce cas, les noms des champs sont déterminés à partir des champs de la classe.

# Exploitation en SQL

La préparation a permis d'obtenir une structure de données nommée _tree_ et possédant deux champs : un champ _hauteurenm_ de type `float` et un champs _espece_ de type `String`.

    ---------------------------------------
    | hauteurenm | espece                 |
    ---------------------------------------
    |       10.0 | Aesculus hippocastanum |
    |       15.0 | Tilia platyphyllos     |
    |        0.0 | Platanus x hispanica   |
    |       10.0 | Paulownia tomentosa    |
    ...

Nous pouvons manipuler ces données avec des requêtes SQL classiques. Le résultat obtenu est lui-même un `JavaSchemaRDD`.

Il est ainsi possible d'obtenir la liste des espèces dans l'ordre alphabétique :

{% highlight java %}
sqlContext.sql("SELECT DISTINCT espece FROM tree WHERE espece <> '' ORDER BY espece")
        .foreach(row -> System.out.println(row.getString(0)));
{% endhighlight %}

    Acacia dealbata
    Acer acerifolius
    Acer buergerianum
    Acer campestre
    ...

Ici, on a itéré sur les éléments du RDD résultant (opération `foreach`) et affiché le premier champ (`row.getString(0)`).

Les opérations d'agrégations sont également possibles, comme pour compter le nombre d'arbres par espèce :

{% highlight java %}
sqlContext.sql("SELECT espece, COUNT(*) FROM tree WHERE espece <> '' GROUP BY espece ORDER BY espece")
        .foreach(row -> System.out.println(row.getString(0) + " : " + row.getLong(1)));
{% endhighlight %}

    Acacia dealbata : 2
    Acer acerifolius : 39
    Acer buergerianum : 14
    Acer campestre : 452
    ...

Les opérations mathématiques de base sont aussi disponibles. On peut ainsi calculer la hauteur moyenne des arbres :

{% highlight java %}
double avgHeight = sqlContext.sql("SELECT AVG(hauteurenm) FROM tree WHERE hauteurenm > 0")
        .collect().get(0).getDouble(0);
System.out.println(avgHeight);
{% endhighlight %}

    11.186036372695565

Ici, pour récupérer le résultat, nous récupérons le RDD dans une liste (`collect()`) dans laquelle nous prenons le premier record (`get(0)`). Nous récupérons enfin la valeur sous forme d'un nombre (`getDouble(0)`).

# Conclusion

Spark SQL permet d'exploiter très simplement des données. Dans les trois exemples montrés ici, une requête SQL simple a permis, à chaque fois, d'effectuer des manipulations de données sans écrire de code Spark.

Un des intérêts de Spark SQL est qu'il permet d'exploiter n'importe quelle source de données avec une syntaxe SQL, offrant ainsi de nombreuses possibilités.

Autre avantage de Spark SQL : les résultats sont retournés sous forme de RDD. Il est donc possible d'enchaîner des requêtes SQL avec des opérations plus bas niveau (type MapReduce), voire de réutiliser des résultats dans d'autres requêtes SQL.