---
layout: post
title:  "Java 8 – Interfaces fonctionnelles"
date:   2014-03-18 11:00:00
tags: java8
language: FR
---
Java 8 introduit le concept d’”interface fonctionnelle” qui permet de définir une interface disposant d’une unique méthode abstraite, c’est-à-dire une seule méthode ne possédant pas d’[implémentation par défaut](http://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html).

Le but d’une interface fonctionnelle est de définir la signature d’une méthode qui pourra être utilisée pour passer en paramètre :

- une référence vers une méthode statique
- une référence vers une méthode d’instance
- une référence vers un constructeur
- une expression lamba.

Même si ce n’est pas obligatoire, le JDK permet de vérifier le contrat “une seule méthode abstraite” en appliquant l’annotation [@FunctionalInterface](http://download.java.net/jdk8/docs/api/java/lang/FunctionalInterface.html) :

{% highlight java %}
@FunctionalInterface
public interface ExampleInterface {
    void doSomething();
    default int methodWithDefaultImpl() { return 0; }
}
{% endhighlight %}

Si vous définissez plusieurs méthodes abstraites, le compilateur génèrera une erreur du type :

{% highlight text %}
Unexpected @FunctionalInterface annotation
ExampleInterface is not a functional interface
multiple non-overriding abstract methods found in interface ExampleInterface
{% endhighlight %}

# Retour en arrière

En Java &lt; 8, lorsqu’il fallait passer une fonction en paramètre d’un appel de fonction, le recours à une classe anonyme était courant.

Prenons un exemple. Nous devons parser des chaînes de caractères de type “&lt;prénom&gt; &lt;nom&gt;” en les découpant sur le caractère espace.

Nous allons créer des objets de type Name :

{% highlight java %}
public class Name {

   private String firstName;
   private String lastName;

   public Name(String firstName, String lastName) {
       this.firstName = firstName;
       this.lastName = lastName;
   }

   public String getFirstName() { return firstName; }
   public String getLastName() { return lastName; }

}
{% endhighlight %}

Pour le parsing, nous créons une classe NameParser dont la responsabilité sera limitée au seul rôle de parsing. La classe NameParser ne doit donc pas construire l’objet résultant. Nous délèguons cette responsabilité à une interface Creator qui déclare une méthode “create” prenant deux arguments (le prénom et le nom extraits lors du parsing) :

{% highlight java %}
public class NameParser {
    public  T parse(String name, Creator creator) {
        String[] tokens = name.split(" ");
        String firstName = tokens[0];
        String lastName = tokens[1];
        return creator.create(firstName, lastName);
    }
}

public interface Creator {
    T create(String firstName, String lastName);
}
{% endhighlight %}

Pour utiliser notre NameParser, nous devons l’appeler en lui passant une instance d’une classe implémentant l’interface Creator. Nous avons donc recours à une [classe anonyme](http://docs.oracle.com/javase/tutorial/java/javaOO/anonymousclasses.html) :

{% highlight java %}
NameParser parser = new NameParser();

Name res = parser.parse("Eric Clapton", new Creator<name>() {
    @Override
    public Name create(String firstName, String lastName) {
        return new Name(firstName, lastName);
    }
});
{% endhighlight %}

Les responsabilités sont clairement dissociées mais la syntaxe résultante est très verbeuse et la lisibilité du code est rendue difficile…

# En Java 8…

Java 8 apporte une réponse à ce problème grâce aux “interfaces fonctionnelles”.

Bien que l’annotation @FunctionalInterface ne soit pas obligatoire, nous l’ajoutons sur notre interface Creator :

{% highlight java %}
@FunctionalInterface
public interface Creator<T> {
    T create(String firstName, String lastName);
}
{% endhighlight %}

Sans aucune modification sur la classe NameParser, nous allons maintenant pouvoir passer toute méthode dont la signature répondra aux contraintes suivantes :

- deux paramètres de type String
- type de retour générique

Java se chargera en interne de convertir l’appel de sorte que l’on aura toujours l’impression d’appeler la méthode “create” de l’interface Creator.

## Référence vers un constructeur

Le constructeur de la classe Name répond aux contraintes définies ci-dessus. Nous pouvons donc écrire :

{% highlight java %}
Name res = parser.parse("Eric Clapton", Name::new);
{% endhighlight %}

Ici, la syntaxe “&lt;cible&gt;::&lt;méthode&gt;” permet de définir une référence sur méthode, le mot-clé “new” faisant référence au constructeur de la classe Name.

## Référence vers une méthode statique

De la même manière, nous pouvons donner une référence vers une méthode statique. Prenons une factory :

{% highlight java %}
public class Factory {
    public static Name createName(String firstName, String lastName) {
        return new Name(firstName, lastName);
    }
}
{% endhighlight %}

Nous pouvons écrire :

{% highlight java %}
Name res = parser.parse("Eric Clapton", Factory::createName);
{% endhighlight %}


## Référence vers une méthode d’instance

Toujours avec la même syntaxe, nous pouvons donner une référence vers une méthode d’instance, donc une référence vers un objet existant. Prenons une factory légèrement modifiée (plus de mot-clé “static”) :

{% highlight java %}
public class Factory {
    public Name createName(String firstName, String lastName) {
        return new Name(firstName, lastName);
    }
}
{% endhighlight %}

Nous pouvons alors écrire :

{% highlight java %}
Factory factory = new Factory();
Name res = parser.parse("Eric Clapton", factory::createName);
{% endhighlight %}


## Expression lambda

Enfin, nous pouvons passer une expression lambda :

{% highlight java %}
Name res = parser.parse("Eric Clapton", (s1, s2) -> new Name(s1, s2));
{% endhighlight %}

Ou alors, avec notre factory :

{% highlight java %}
Name res = parser.parse("Eric Clapton", (s1, s2) -> Factory.createName(s1, s2));
{% endhighlight %}

# Package java.util.function

Dans notre exemple, nous avons créé notre propre interface fonctionnelle. Pour les cas les plus simples – et certainement les plus courants – ce n’est pas nécessaire. En effet, le package java.util.function fait son apparition dans le JDK et reçoit de nombreuses interfaces fonctionnelles de base : http://download.java.net/jdk8/docs/api/java/util/function/package-summary.html

Les interfaces définies avec des types génériques sont :

- Consumer<T> : opération qui accepte un unique argument (type T) et ne retourne pas de résultat.

{% highlight java %}
void accept(T);
{% endhighlight %}

- Function<T,R> : opération qui accepte un argument (type T) et retourne un résultat (type R).

{% highlight java %}
R apply(T);
{% endhighlight %}

- Supplier<T> : opération qui ne prend pas d’argument et qui retourne un résultat (type T).

{% highlight java %}
T get();
{% endhighlight %}

Notons l’interface Predicate qui est une spécialisation de Function visant à tester une valeur et retourner un booléen.

{% highlight java %}
boolean test(T);
{% endhighlight %}

Enfin, de nombreuses autres interfaces fonctionnelles sont définies avec des types de base : IntConsumer, LongToIntFunction, DoubleSupplier, etc.

# Exemple avec plusieurs méthodes

Nous avons vu qu’une interface fonctionnelle ne peut contenir qu’une seule méthode abstraite. Impossible, donc, d’annoter l’interface suivante avec @FunctionalInterface sous peine d’obtenir une erreur de compilation :

{% highlight java %}
private interface Operation<T>
{
    public T function();
    public void onSuccess(T res);
    public void onError(Exception ex);
}
{% endhighlight %}

En Java &lt; 8, nous aurions écrit :

{% highlight java %}
public <T> void doSomething(Operation<T> operation) {
    try {
        T res = operation.function();
        operation.onSuccess(res);
    } catch (Exception ex) {
        operation.onError(ex);
    }
}
{% endhighlight %}

Avec un appel très verbeux :

{% highlight java %}
doSomething(new Operation<Object>() {
    @Override
    public Object function() {
        return 42;
    }
    @Override
    public void onSuccess(Object res) {
        System.out.println(res);
    }
    @Override
    public void onError(Exception ex) {
        System.err.println("Error: " + ex.getMessage());
    }
});
{% endhighlight %}

En Java 8, nous pouvons nous passer complètement de l’interface Operation et utiliser une interface fonctionnelle par méthode. Et, puisque c’est possible, nous allons exploiter le package java.util.function. Notre méthode devient :

{% highlight java %}
public <T> void doSomething(Supplier<T> function, Consumer<T> onSuccess, Consumer<Exception> onError) {
   try {
       T res = function.get();
       onSuccess.accept(res);
   } catch (Exception ex) {
       onError.accept(ex);
   }
}
{% endhighlight %}

Et l’appel est grandement simplifié :

{% highlight java %}
doSomething(
    () -> 42,
    System.out::println,
    ex -> System.err.println("Error: " + ex.getMessage()));
{% endhighlight %}

# Conclusion

Le principe d’interface fonctionnelle permet de se passer des classes anonymes dans un grand nombre de cas. Du point de vue du code appelé, les choses restent simples : on continue à appeler une méthode d’une interface. C’est du côté du code appelant que la lisibilité du code est grandement améliorée . L’emprunts aux langages fonctionnels est ici une grande réussite.