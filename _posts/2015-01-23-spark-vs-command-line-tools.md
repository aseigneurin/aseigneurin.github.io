---
layout: post
title:  "Spark vs Command line tools"
date:   2015-01-23 11:00:00
tags: spark
language: FR
---
Il y a quelques jours, un post d'Adam Drake a refait surface sur Twitter : [Command-line tools can be 235x faster than your Hadoop cluster](http://aadrake.com/command-line-tools-can-be-235x-faster-than-your-hadoop-cluster.html). Adam explique qu'il a reproduit un traitement Hadoop avec des outils de ligne de commande (find, awk...) multipliant ainsi le débit de traitement par 235. J'ai cherché à reproduire cette comparaison avec Spark.

# Résumé du test conduit par Adam Drake

Adam explique avoir été surpris par l'expérimentation réalisée par Tom Hayden, lequel a utilisé Hadoop pour extraire des statistiques de gain/perte de parties d'échecs dans un dataset de 1,75 Go. Le traitement Hadoop requiert 26 minutes soit un débit de 1,14 Mo/seconde.

Malheureusement, le post original de Tom Hayden n'est plus accessible. Tout ce que l'on sait, c'est que 7 machines de type *c1.medium* ont été utilisées !

Adam a reproduit un traitement équivalent avec des outils du shell (find, awk, xargs...) en expliquant que les *pipes* permettent de paralléliser les traitements. Le dataset utilisé par Adam est constitué de 3,46 Go de données et son traitement ne prend que 12 secondes sur sa machine soit un débit de 270 Mo/seconde. Belle performance !

# L'expérimentation que j'ai conduite

J'ai commencé par exécuter le traitement shell d'Adam. Attention, [le jeu de données](https://github.com/rozim/ChessData) qu'il référence a évolué et fait désormais 4,6 Go.

Sur ma machine (MacBook Pro fin 2013, i7 à 2 GHz, 16 Go de Ram, disque SSD), le traitement prend 10 secondes soit un débit de 460 Mo/seconde :

```bash
$ time find . -type f -name '*.pgn' -print0 | xargs -0 -n4 -P4 mawk '/Result/ { split($0, a, "-"); res = substr(a[1], length(a[1]), 1); if (res == 1) white++; if (res == 0) black++; if (res == 2) draw++ } END { print white+black+draw, white, black, draw }' | mawk '{games += $1; white += $2; black += $3; draw += $4; } END { print games, white, black, draw }'
6829064 2602614 1974505 2251945

real    0m10.218s
user    0m17.589s
sys     0m4.215s
```

J'ai reproduit le traitement avec Spark 1.2.0 **sans chercher à optimiser l'exécution** :

```java
package com.seigneurin.spark;

import org.apache.spark.api.java.JavaSparkContext;

public class Chess {

    public static void main(String[] args) {

        JavaSparkContext sc = new JavaSparkContext("local[16]", "chess");

        long start = System.currentTimeMillis();

        sc.textFile("ChessData-master/*")
                .filter(line -> line.startsWith("[Result ") && line.contains("-"))
                .map(res -> res.substring(res.indexOf("\"") + 1, res.indexOf("-")))
                .filter(res -> res.equals("0") || res.equals("1") || res.equals("1/2"))
                .countByValue()
                .entrySet()
                .stream()
                .forEach(s -> System.out.println(s.getKey() + " -> " + s.getValue()));

        long duration = System.currentTimeMillis() - start;
        System.out.println("Duration: " + duration + " ms");

        sc.close();
    }
}
```

(Code complet disponible sur [mon GitHub](https://github.com/aseigneurin/spark-chess).)

Le traitement est lancé en local, via un *main*, et donc sans utiliser de cluster (la volumétrie ne s'y prête pas). Par ailleurs, Spark est configuré pour utiliser 16 threads.

Le traitement se découpe comme suit :

- lecture des fichiers via `textFile` en utilisant un wildcard
- filtrage des lignes de résultat avec un `filter`
- parsing des résultats avec un `map`
- filtrage des résultats avec un nouveau `filter` : on ne garde que les échecs ("0"), les gains ("1") et les matches nuls ("1/2")
- comptage via `countByValue` qui retourne une `Map`
- affichage sur la console via l'API de streaming des collections de Java 8 (`entrySet`, `stream` et `forEach`)

Lors de l'exécution, c'est important, les résultats sont les mêmes que ceux obtenus avec les commandes shell. Le temps de traitement, lui, passe à 30 secondes, soit un débit de 153 Mo/seconde :

```bash
0 -> 1974505
1 -> 2602614
1/2 -> 2251945
Duration: 29964 ms
```

# Peut-on faire mieux ?

Dans des conditions équivalentes (pas de cluster, fichiers en local, même machine), Spark est trois fois moins rapide que les outils du shell. Naturellement, la question peut se poser de savoir si ce programme Spark peut être optimisé.

J'ai fait varier plusieurs paramètres :

- réglage de la mémoire allouée à la JVM (`-Xmx`) et à l'exécteur Spark (`spark.executor.memory`)
- utilisation de pointeurs sur 4 octets au lieu de 8 (`-XX:+UseCompressedOops`) comme conseillé sur le [guide de tuning](http://spark.apache.org/docs/latest/tuning.html)
- comptage par des opérations élémentaires plutôt que par le `countByValue`
- différentes méthodes de parsing des lignes (`String.split`...)

Ces changements n'ont presque pas fait varier le temps de traitement. Rien qui ne soit significatif, en tout cas.

Il me semble - et c'est à vérifier - que la lecture des fichiers est effectuée sur un thread unique. Les partitions créées à partir de ces fichiers sont ensuite dispatchées. Difficile, donc, d'optimiser ce traitement...

## Update du 24/01/2015 : améliorer le pattern de fichiers

Plutôt que de lire l'ensemble des fichiers du répertoire, on peut utiliser un pattern ne récupérant que les fichiers `.pgn` (suggestion fournie par mon collègue [Stéphane Trou](https://www.linkedin.com/pub/st%C3%A9phane-trou/1b/527/335)). On écrit donc :

```java
        sc.textFile("ChessData-master/*/*.pgn")
```

Le temps de traitement est alors presque divisé par deux : 16 secondes soit un débit de 287 Mo/seconde.

```bash
0 -> 1974505
1 -> 2602614
1/2 -> 2251945
Duration: 15869 ms
```

Pourquoi une telle différence ? Pour l'instant, je ne l'explique pas. Les fichiers qui ne sont pas de type `.pgn` ne représentent que 8 ko ce qui ne peut pas en soit justifier la différence.

# Que faut-il en déduire ?

Spark n'est pas aussi rapide que les outils du shell quand il s'agit de traiter des données présentes sur une machine, bien que le résultat soit tout de même proche. Quoi qu'il en soit, l'utilité de Spark ne réside pas là : il s'agit avant tout d'**utiliser l'outil adapté au besoin**. En l'occurence, Spark est adapté au traitement distribué et le framework serait donc approprié pour de larges volumes de données répartis sur plusieurs machines. Ici, le volume de données étant très limité et le traitement étant simple, l'utilisation d'outils du shell fait sens.

Alors, certes, les mesures ne permettent pas de comparer directement Spark et Hadoop. Néanmoins, il est évident que Spark est plus rapide dans un rapport d'un ou deux [ordres de magnitude](https://en.wikipedia.org/wiki/Order_of_magnitude). Spark confirme ainsi qu'il est un challenger plus que sérieux pour Hadoop.