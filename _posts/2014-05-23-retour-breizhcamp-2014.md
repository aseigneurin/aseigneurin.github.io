---
layout: post
title:  "Retour sur le Breizh Camp 2014"
date:   2014-05-23 11:00:00
tags: java8 vagrant breizhcamp
language: FR
origUrl: http://blog.ippon.fr/2014/05/23/retour-sur-le-breizh-camp-2014/
origSource: le blog d'Ippon Technologies
---
Le Breizh Camp 2014 est déjà terminé pour moi. Sur deux jours et demi, la conférence organisée sur le campus universitaire de Rennes a accueilli 250 personnes. Au programme, 80 talks, tous formats confondus (conférences, hands-on labs, tools in action, quickies) !

<img src="/images/breizhcamp_2014.png" style="float:right; padding: 20px 0px 20px 30px"/>

Retour sur deux conférences auxquelles j’ai assisté.

# Continuous Delivery chez Capitaine Train

[Frédéric Menou](https://twitter.com/ptit_fred) est développeur (et accessoirement « Ops ») chez [Capitaine Train](https://www.capitainetrain.com/fr) depuis 2 ans. Il nous explique leur organisation pour livrer régulièrement tout en ayant une plateforme stable. L’objectif est le zero-downtime puisque Capitaine Train est avant tout un site marchand.

La première règle pour les déploiements, c’est « la routine » : les composants se déploient selon un modèle standard (propre à Capitaine Train). Ainsi, chacun peut déployer le composant d’un autre.

Le quotidien du développeur est de développer dans des features branches (sous Git) et de faire des pull requests sur la branche « dev ». Un code review est effectué par un pair (un seul) pour vérifier la qualité du code, sa lisibilité, et la cohérence avec l’infrastructure. Un point important, lors de cette revue, est de vérifier que les déploiements pourront être effectués sans risques.

Les pull requests sont donc mergés sur la branche « dev ». Lorsqu’une feature est prête à partir en production, « dev » est mergée sur « master ». La branche « master » doit rester « iso-production » : on ne merge pas sur master si ce n’est pas pour mettre en production dans la demi heure qui suit.

La croissance de la plateforme est compliquée à gérer, notamment avec l’arrivée des appli mobiles. Pour supporter cette croissance, les composants se spécialisent (la mode des microservices…). Un bus AMQP (RabbitMQ) est utilisé pour découpler les composants de l’infrastructure et ainsi travailler avec plus de souplesse.

Frédéric rappelle que les développeurs « accompagnent » leurs features jusqu’à la production. Ils en sont responsables de bout en bout. La mise en production est toutefois réalisée par un sysadmin. Cette MEP est manuelle (une commande) mais encore une fois routinière.

Quelques règles « d’hygiène » : on ne prode pas le vendredi (je pensais que c’était un acquis pour tout le monde, mais apparement non…), on ne prode pas après 18 heures, et on n’hésite pas à proder par petits morceaux.

Les déploiements unitaires sont la règle chez Capitaine Train.  Ce n’est toutefois pas simple pour le modèle SQL. La stratégie est de déployer – sur l’ensemble des nœuds – le code supportant l’ancien et le nouveau schémas de base de données, d’upgrader la base, puis de déployer le code nettoyé. Du code jetable est donc écrit, dont la durée de vie maximum est de l’ordre de la demi-heure.

Les mises à jour de la base de données elle-même peuvent d’ailleurs être un problème. Plutôt que de renommer des colonnes, la préférence va à ajouter une colonne puis à supprimer l’ancienne. L’ajout d’index est plus problématique puisque l’opération est bloquante sous PostgreSQL. Pas de réponse type à cette question.

Pour finir, Frédéric indique que, chez Capitaine Train, il n’y a pas de « grand soir » : les évolutions se font par petits pas. La discipline de groupe est importante dans cette marche en avant, chacun devant faire simple et penser aux autres. Un bel exemple d’une culture DevOps qui fonctionne.

# How to scale?

[Quentin Adam](https://twitter.com/waxzce) est le créateur de [Clever Cloud](https://www.clever-cloud.com/fr/). Après nous avoir donné une keynote détonnante sur sa vie de « startuper » (seulement 26 ans et startuper depuis l’age de 18 ans !), Quentin nous livre ses conseils pour monter en charge.

À un moment ou à un autre, une application est vouée à monter en charge et finira par ne plus tenir sur une seule machine. Il faut prévoir le « scaling out ». Les nœuds de l’infrastructure doivent être des clones et il faut arrêter de personnifier les serveurs : pas de pitié, on ajoute des serveurs et on en tue d’autres.

Pour y parvenir, la séparation entre les données et le code doit être claire dans l’infrastructure. Les nœuds de traitement ne doivent pas stocker de données. Les évènements sont considérés comme des données et doivent être délégués à un middleware (AMQP, JMS…) qui garantira l’acheminement.

Puisque les données doivent être découplée des nœuds de traitement, Quentin indique qu’il ne faut surtout pas utiliser le file system des serveurs. Celui-ci est propre à une machine et un « stockage objet » (type S3) est plus approprié.

D’ailleurs, les logs de nos applications ne doivent pas être stockés sur le file system mais ils doivent être centralisés. Pour cela, Logstash est approprié.

Quentin déconseille également d’utiliser la RAM de nos serveurs pour stocker des données. Par exemple, pour transférer un fichier du client vers le stockage final, il faut le streamer vers sa destination plutôt que de le stocker temporairement en RAM.

Par ailleurs, les choix techniques (frameworks, services…) doivent être effectués en fonction des besoins, pas en fonction de ce que l’on aime ou de ce que l’on sait faire. Ces choix doivent incomber aux Devs et les Ops doivent s’adapter (pas toujours facile à faire accepter…).

Dans cet esprit, il ne faut pas hésiter à mixer les technologies si ces technos sont mieux adaptées au besoin. La courbe d’apprentissage est plus élevée mais le choix est vite rentable.

Modulariser est également important (les microservices, encore une fois) pour monter en charge. Cela permet de refactorer plus facilement.

Pour le déploiement, le process doit être simple au possible. Idéalement, un simple « git push » doit suffire.

Enfin, Quentin conseille de ne jamais exposer d’application en front et de toujours utiliser un reverse proxy. Les déploiements, notamment, sont simplifiés : on démarre un nœud avec la nouvelle version de l’appli, on re-route le trafic vers ce nœud, puis on éteint l’ancien nœud.

# Conclusion sur ces deux présentations

Deux présentations sur des sujets a priori différents mais, au final, des constantes :

- spécialiser les composants (des microservices) pour monter en charge
- utiliser un bus AMQP pour découpler et utiliser un transport fiable
- simplifier et standardiser le process de release

Sans que le terme ait été prononcé une seule fois, la problématique DevOps est bien présente chez Capitaine Train et chez Clever Cloud. Elle fait partie de la culture de ces deux startups et elle est indispensable à leur réussite.

# Auto-promo…

J’étais moi-même speaker pour cette édition. J’ai présenté un quickie sur les interfaces fonctionnelles de Java 8. Petit moment de panique au moment de démarrer avec le crash de mon Mac, mais l’assistance était bienveillante. Bref, les slides sont dispos ici : [pres-java8-breizhcamp](http://aseigneurin.github.io/downloads/pres-java8-breizhcamp/)

J’ai également présenté un tools in action intitulé « Vagrant pour les développeurs ». Les slides sont également en ligne : [pres-vagrant-breizhcamp](http://aseigneurin.github.io/downloads/pres-vagrant-breizhcamp/). Je vais proposer cette présentation en Brown Bag Lunch. Rendez-vous très bientôt sur [Brown Bag Lunch France](http://www.brownbaglunch.fr/) ou sur [Twitter](https://twitter.com/aseigneurin).