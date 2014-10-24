---
layout: post
title:  "Applications AngularJS et le SEO"
date:   2014-10-24 16:00:00
tags: angularjs seo
language: FR
---
AngularJS, tout comme de nombreux frameworks Javascript récents, exploitent le navigateur pour effectuer le rendu côté client. Les pages fournies par le serveur sont donc des coquilles vides. Les sites exploitant ces technologies sont donc très mal indexés par les moteurs de recherche.

Les moteurs de recherche, dont Google et Bing, permettent néanmoins d’indexer un contenu généré de manière alternative, et tous proposent le même fonctionnement. Google documente le mécanisme sur cette page et nous le détaillons ci-après.

## 1. Sitemaps

Lorsque le moteur de recherche parcourt le site Web, il obtient une page HTML vide ainsi que du Javascript. Le Javascript n’étant pas interpreté (ou très peu), le moteur ne peut pas naviguer dans le site et il n’est donc pas en mesure de découvrir les différentes pages qui le composent. Il est donc nécessaire de créer une Sitemap XML listant les pages.

Cf. Référence pour les Sitemaps : <https://support.google.com/webmasters/answer/183668?hl=fr&ref_topic=6080646&rd=1>

## 2. URLs référencées

Les URLs utilisées sur le site comportent une partie utilisée uniquement côté client : tout ce qui se trouve après le caractère “#” (ex : <http://example.com/#/shop/product/42>). Ni le serveur, ni les moteurs d’indexation ne traitent habituellement cette partie de l’URL, qu’il ne faut d’ailleurs pas confondre avec [les ancres](http://www.w3.org/TR/html401/struct/links.html#h-12.1).

Pour indiquer aux moteurs d’indexation que cette partie d’URL est un fragment d’URL, il faut insérer le caractère “!” immédiatement après le “#” dans toutes les URLs (ex : <http://example.com/#!/shop/product/42>).

Avec AngularJS, cela peut être fait très simplement de la manière suivante :

    angular.module('myApp').config([  
        '$locationProvider',
        function($locationProvider) {
            $locationProvider.hashPrefix('!');
        }
    ]);

Ces URLs contenant “#!” (que l’on appelle “hashbang”) pourront être utilisées sur le site sans que cela ait d’incidence sur le comportement d’AngularJS (prise en compte native dans ngRoute).

Côté moteur d’indexation, en revanche, ces URLs seront modifiées avant d’être envoyées au serveur. Le hashbang va être remplacé par “?_escaped_fragment_” (ex : <http://example.com/?_escaped_fragment_/shop/product/42>).

## 3. Génération de contenu côté serveur

Le serveur va désormais recevoir des URLs contenant l’ensemble des indications nécessaires à la génération d’une page alternative. Une technologie de rendu côté serveur telle que [Thymeleaf](http://www.thymeleaf.org/) ou [Freemarker](http://freemarker.org/) pourra être utilisée pour générer ces pages.

Il est important de noter que les pages générées n’ont pas besoin d’être mises en forme. En effet, lorsque le moteur de recherche remontera un résultat, l’utilisateur sera renvoyée vers l’URL non-altérée (avec “#!”). L’application AngularJS exploitera donc l’URL complète pour générer la page correctement mise en forme.

# Site mobile & Duplicate content

Deux techniques principales existent pour supporter des navigateurs “desktop” et des mobiles :
utiliser un site unique exploitant le [Responsive Web Design](http://fr.openclassrooms.com/informatique/cours/qu-est-ce-que-le-responsive-web-design)
utilisant deux site séparés, l’un pour les desktops (ex : <http://www.example.com>), l’autre pour le mobile (ex : <http://m.example.com>)

Dans le cas de deux sites séparés, on peut craindre que le contenu soit détecté comme étant du contenu dupliqué (“Duplicate content”, <https://support.google.com/webmasters/answer/66359?hl=fr>).

Toutefois, Google indique que son algorithme prend en compte ce cas particulier et ne déclare pas de contenu dupliqué : <https://www.youtube.com/watch?v=mY9h3G8Lv4k>

## URLs canoniques

Dans le cas où Google détecterait tout de même du contenu dupliqué, il resterait possible d’indiquer l’URL du contenu canonique via un header spécifique :

    <link rel="canonical" href="http://www.example.com/..."/>

Cf. <https://support.google.com/webmasters/answer/139066?hl=fr#2>

## Gestion du Google Bot

Enfin, il faut noter que Google utilise deux robots d’exploration pour parcourir le contenu : “Googlebot” pour simuler un navigateur desktop, et “Googlebot-Mobile” pour simuler un navigateur mobile.

Il est possible d’exploiter cela pour privilégier l’indexation du site mobile par le robot mobile, et l’indexation du site principal par le robot standard.

Cf. <https://support.google.com/webmasters/answer/1061943?hl=fr>
