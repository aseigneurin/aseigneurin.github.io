---
layout: post
title:  "Spark Summit, le retour"
date:   2015-11-07 11:00:00
tags: spark datascience
language: FR
---
Le [Spark Summit](https://spark-summit.org/eu-2015/) - la conférence officielle autour du framework [Apache Spark](http://spark.apache.org/) - se tenait à Amsterdam du 27 au 29 octobre.

Votre serviteur était présent les 28 et 29 octobre. Résumé des évènements.

<!--more--><!--noteaser-->
<h2>Quelques faits</h2>
Cette conférence était le premier Spark Summit organisé en Europe, les précédents ayant eu lieu à San Francisco et New York. 930 participants étaient réunis pour 2 jours de conférence et, pour certains, un jour supplémentaire de training.

Les conférences étaient réparties entre 4 tracks : développeur, applications, Data Science et recherche. La plupart du temps, seuls deux tracks étaient menés en parallèle ce qui permettait d'assister à près de la moitié des talks.

<img src="/images/SparkSummit/IMG_7180-300x225.jpg" width="300" height="225"> <img src="/images/SparkSummit/IMG_7262-300x219.jpg" width="300" height="219" />

Les conférences étaient en format 30 minutes, format sur lequel j'ai un avis partagé. D'un côté, les sessions sont suffisamment courtes pour qu'on puisse choisir un talk sans avoir peur de se tromper. De l'autre, et c'est plus gênant, les speakers ne disposaient pas d'assez de temps pour rentrer dans les détails techniques.
<h2>Les tendances de fond</h2>
<h3>Importance croissante de la Data Science</h3>
À en juger par le public (nombreux Data Scientists) et par le programme (track Data Science), Spark est populaire notamment grâce à sa librairie de Machine Learning. Les talks portant sur ce sujet ont été nombreux, à commencer par une démo de <em>Sentiment Analysis</em> avec Databricks Cloud par Hossein Falaki, dont je recommande [la vidéo](https://vimeo.com/143883002).

<img src="/images/SparkSummit/IMG_7205-300x159.jpg" width="300" height="159" />
<h3>Full streaming, Akka, Cassandra...</h3>
La tendance est au streaming plutôt qu'au batch, et donc à un traitement des évènements au fur et à mesure de leur arrivée plutôt que de faire tourner des process à intervalles réguliers. De nombreux speakers ont mentionné [Kafka](http://kafka.apache.org/) pour stocker le flux d'évènements et le processer ensuite avec Spark Streaming.

Par ailleurs, [Cassandra](http://cassandra.apache.org/) a été mentionné de nombreuses fois (et pas uniquement par Patrick McFadin !) comme étant une base de données parfaitement adaptée pour supporter des workloads importants.

Plus généralement, c'est la stack <em>SMACK</em> (Spark, Mesos, Akka, Cassandra, Kafka) qui semble avoir les faveurs de nombreux développeurs plutôt que des stacks reposant sur Hadoop.

Dans l'idée d'utiliser le streaming, plusieurs voix se sont élevées pour abandonner la définition actuelle des ETL. La notion d'"Extract" est dépréciée, car on reçoit désormais des flux (qui plus est contenant des données typées). Il en va de même pour la notion de "Load", puisque les données transformées sont désormais consommées par plusieurs systèmes. Une définition alternative [est proposée](http://noetl.org/) : CTP (Consume, Transform, Produce).

<img src="/images/SparkSummit/IMG_7354-300x188.jpg" width="300" height="188" />

<img src="/images/SparkSummit/IMG_7272-225x300.jpg" width="225" height="300" />

Je recommande les talks d'Helena Edelson, [Lambda Architecture, Analytics and Data Pathways with Spark Streaming, Kafka, Akka and Cassandra](https://www.youtube.com/watch?v=buszgwRc8hQ&amp;index=5&amp;list=PL-x35fyliRwif48cPXQ1nFM85_7e200Jp), et de Natalino Busa, [Real-Time Anomaly Detection with Spark ML and Akka](https://www.youtube.com/watch?v=Aeg5yEBuqgM&amp;list=PL-x35fyliRwip21Nks3Pw3d0GnsUf-b4B&amp;index=9).
<h3>Popularité des langages</h3>
Côté développeurs, Scala était clairement le langage adopté par le plus grand nombre. Typesafe était d'ailleurs présent et représenté par Martin Odersky (voir [sa keynote](https://www.youtube.com/watch?v=yzfCTNukfl8&amp;list=PL-x35fyliRwjhpVYR1uYeu21v263_mLvq&amp;index=1)).

Côté Data Science, R et le binding Spark R étaient largement présents dans les talks. Voir notamment le talk d'Hossein Falaki, [Enabling exploratory data science with Spark and R](https://www.youtube.com/watch?v=wGmg6k3wGUg&amp;index=2&amp;list=PL-x35fyliRwhyhPYtiR5M0zW9bGVCVM--). Python, bien que beaucoup utilisé, était finalement moins représenté dans les talks.

<img src="/images/SparkSummit/IMG_7332-300x209.jpg" width="300" height="209" /> <img src="/images/SparkSummit/IMG_7337-300x219.jpg" width="300" height="219" />
<h3>Clusters à la demande</h3>
Chez Databricks - avec Databricks Cloud - comme chez Goldman Sachs, les clusters Spark sont lancés à la demande. Le point surprenant est que la <em>data locality</em> n'est pas possible, puisque les données sont stockées sur un datastore permanent (S3 dans le cas de Databricks Cloud). Néanmoins, chez Goldman Sachs, cela permet aux Data Scientists d'utiliser des clusters de processing sur de courtes durées pour un coût réduit.

Je recommande d'ailleurs l'excellente keynote de Vincent Saulys, [How Spark is Making an Impact at Goldman Sachs](https://www.youtube.com/watch?v=HWwAoTK2YrQ&amp;index=4&amp;list=PL-x35fyliRwj6mYuPdiGzG-jMPD8iec3E), dans laquelle il explique que Spark a connu une adoption virale chez Goldman Sachs.

<img src="/images/SparkSummit/IMG_7215-300x170.jpg" width="300" height="170" />
<h3>Performances</h3>
Spark est déjà connu pour être performant mais les développeurs de Databricks travaillent activement à améliorer encore les performances. Les optimisations sont souvent réalisées à un très bas niveau (structures mémoire, génération de bytecode, etc.) et le runtime pourrait, à terme, ne plus être la JVM (voir la keynote de Reynold Xin, [A Look Ahead at Spark's Development](https://www.youtube.com/watch?v=TynknhKavxI&amp;index=3&amp;list=PL-x35fyliRwjhpVYR1uYeu21v263_mLvq)).

<img src="/images/SparkSummit/IMG_7305-300x195.jpg" width="300" height="195" /> <img src="/images/SparkSummit/IMG_7311-300x211.jpg" width="300" height="211" />
<h3>Notebooks</h3>
Bien que les notebooks aient des défauts (pas adapté au versioning de code...), ils sont très populaires chez les Data Scientists. [Apache Zeppelin](https://zeppelin.incubator.apache.org/) a notamment été très présent dans les talks. Les créateurs de ce notebook (NFLabs) disposaient d'ailleurs d'un stand.

Le notebook de Databricks - Databricks Cloud - a lui été montré plusieurs fois mais toujours par des employés de Databricks.

<img src="/images/SparkSummit/IMG_7225-300x149.jpg" width="300" height="149" />
<h2>Nouveautés à venir</h2>
<h3>API des Datasets</h3>
Une nouveauté de Spark 1.6 sera l'API des Datasets. Les Datasets sont censés présenter une alternative entre le typage fort des RDD et la performance des Dataframes. Concrètement, on obtient un Dataset à partir d'un Dataframe et en lui appliquant une <em>case class</em>. On manipule alors le Dataset avec des opérations fortement typées et donc vérifiées par le compilateur.

<img src="/images/SparkSummit/IMG_7307-300x196.jpg" width="300" height="196" />
<h3>Spark Streaming - Back pressure</h3>
Spark Streaming étant de plus en plus utilisé, la question qui se pose le plus est celle du dimensionnement du cluster : il faut que le cluster Spark Streaming puisse traiter autant d'évènements qu'il en reçoit pour que la latence n'augmente pas en flèche (le risque est aussi que l'application crashe du fait d'une saturation mémoire).

Deux talks ([Spark Streaming: Pushing the Throughput Limits, the Reactive Way](https://www.youtube.com/watch?v=qxsOjJnwcKQ&amp;index=6&amp;list=PL-x35fyliRwif48cPXQ1nFM85_7e200Jp) et [Leverage Mesos for running Spark Streaming production jobs](https://www.youtube.com/watch?v=DonFPApGR24&amp;list=PL-x35fyliRwip21Nks3Pw3d0GnsUf-b4B&amp;index=10)) ont présenté la technique du <em>Back pressure</em> qui consiste à remonter au producteur d'évènements des informations pour qu'il ralentisse l'envoi de données. Cette technique repose sur des [correcteurs PID](https://fr.wikipedia.org/wiki/R%C3%A9gulateur_PID) et fonctionne notamment avec Kafka.

<img src="/images/SparkSummit/IMG_7381-300x187.jpg" width="300" height="187" /> <img src="/images/SparkSummit/IMG_7382-300x180.jpg" width="300" height="180" />
<h2>Conclusion</h2>
Ce Spark Summit était pour moi l'occasion de vérifier l'engouement de l'industrie pour Spark. Outre les applications Big Data, c'est bien la Data Science qui anime la communauté autour de Spark et qui rend le projet passionnant !

Comme indiqué plus haut, je reste mitigé vis-à-vis du format des talks (30 minutes) même si cela permet d'en voir beaucoup. Néanmoins, cet évènement apporte de nombreux enseignements sur les tendances actuelles et les discussions avec les professionnels du domaine étaient bien sûr très riches.

L'ensemble des talks sont disponibles en vidéo directement depuis [la page du programme](https://spark-summit.org/eu-2015/schedule/) ou via [les playlists sur Youtube](https://www.youtube.com/user/TheApacheSpark/playlists).