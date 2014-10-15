---
layout: post
title:  "[DevoxxFR 2014] Crazyflie Nano"
date:   2014-05-13 11:00:00
tags: devoxx crazyflie
language: FR
---
Parce que le Devoxx ne parle pas que de Java, je voulais revenir sur le talk d’Arnaud Taffanel et de Marcus Eliasson sur la génèse du Crazyflie Nano, un quadricoptère de seulement 19 grammes.

Le projet initial était de mettre au point un “grand” quadricoptère. Le besoin de connaissances en mécanique étant trop important, les créateurs ont décidé de construire un modèle miniature dont la structure serait le circuit imprimé. Arnaud et Marcus nous ont expliqué la difficulté de mise au point due notamment aux vibrations émises par les moteurs : là où la plupart des drônes ont des absorbeurs de vibrations entre les moteurs et le reste de la plateforme, ici, les moteurs sont placés directement sur le support sur lequel le gyroscope et l’accéléromètre sont soudés. De lourdes opérations de traitement de signal sont effectuées pour réussir à faire voler le mini-drône. Une prouesse !

Le pilotage est effectué via ondes radio, en utilisant de préférence une manette de console reliée à un PC. Sur l’ordinateur, des outils de contrôles permettent d’analyser les données en provenance des capteurs.

Le Crazyflie Nano est aujourd’hui entièrement open-source : les plans peuvent être utilisés pour fabriquer soi-même un quadricoptère. Le logiciel est également open-source et il est donc possible de reprogrammer le drône (en langage C). Et c’est bien le but des concepteurs : laisser la communauté hacker la créature volante pour créer des projets dérivés.

Le site du projet : [www.bitcraze.se](http://www.bitcraze.se/)

<img src="/images/crazyflie-nano.jpg"/>
