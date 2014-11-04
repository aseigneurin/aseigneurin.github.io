---
layout: post
title:  "Ansible pour déployer des applications"
date:   2014-11-04 11:00:00
tags: ansible
language: FR
---
[Ansible](http://www.ansible.com/) est un excellent outil de provisioning. L'outil n'est a priori pas prévu pour déployer des applications bien que l'on soit fortement tenté de l'utiliser dans ce but. Ce post traite des problèmes que cela pose et d'une manière de les résoudre.

<img src="/images/ansible_logo_black_square.png" style="float:right; padding-left: 20px; padding-bottom: 20px; width: 250px"/>

L'exercice a porté sur le déploiement d'une webapp Java - et donc d'un WAR - dans un Tomcat.

# Version initiale et ses défauts

La première version d'un playbook Ansible que j'ai écrite était la reproduction de scripts de déploiement "maison". Cette version était impérative, assez brutale et pas vraiment idiomatique :

- on vérifie si une instance Tomcat existe
- si l'instance existe, on l'arrête et on la supprime
- on télécharge Tomcat et on l'installe
- on télécharge le WAR et on le déploie dans Tomcat
- on démarre Tomcat.

Voici un extrait du playbook Ansible résultant (sans variables pour simplifier la lecture) :

{% highlight yaml %}
  - name: Test si une instance existe
    stat: path=/PROD/mywebapp/bin/catalina.sh
    register: p

  - name: Arrêt de l'instance
    command: /PROD/mywebapp/bin/catalina.sh stop
    when: p.stat.isreg is defined and p.stat.isreg == true

  - name: Suppression de l'instance
    command: rm -rf /PROD/mywebapp removes=/PROD/mywebapp

  - name: Téléchargement de Tomcat
    get_url:
      url=http://archive.apache.org/dist/tomcat/tomcat-7/v7.0.56/bin/apache-tomcat-7.0.56.tar.gz
      dest=/tmp/apache-tomcat-7.0.56.tar.gz

  - name: Extraction de Tomcat
    command: chdir=/tmp tar xvf /tmp/apache-tomcat-7.0.56.tar.gz creates=/tmp/apache-tomcat-7.0.56

  - name: Déploiement de Tomcat
    command: cp -r /tmp/apache-tomcat-7.0.56 /PROD/mywebapp creates=/PROD/mywebapp

  - name: Téléchargement du WAR
    get_url:
      url=http://.../mywebapp-1.0.war
      dest=/tmp/mywebapp-1.0.war

  - name: Copie de la webapp dans l'instance
    command: mv /tmp/mywebapp-1.0.war /PROD/mywebapp/webapps/mywebapp.war

  - name: Démarrage de l'instance
    command: nohup /PROD/mywebapp/bin/catalina.sh start
    environment:
      CATALINA_HOME: /PROD/mywebapp
      CATALINA_PID: /PROD/mywebapp/logs/tomcat.pid

  - name: Attente du démarrage de l'instance
    wait_for: port=8080
{% endhighlight %}

## Défaut : le playbook n'est pas idempotent

Le défaut principal de la version ci-dessus est qu'elle n'est pas *idempotente*. Pour rappel, l'idempotence est définie comme suit sur [Wikipedia](http://fr.wikipedia.org/wiki/Idempotence) :

> En mathématiques et en informatique, le concept d’idempotence signifie essentiellement qu'une opération a le même effet qu'on l'applique une ou plusieurs fois, ou encore qu'en la réappliquant on ne modifiera pas le résultat.

En clair, si notre webapp est déjà déployée et que nous lançons notre playbook, **Ansible ne devrait rien modifier et Tomcat ne devrait pas redémarrer**. Ce n'est pas le cas puisque, à chaque exécution, l'instance Tomcat est détruite et re-créée.

Nous allons donc chercher à déployer Tomcat ainsi que le WAR uniquement en cas de besoin.

## Problématique des mises à jour de l'application

Dans notre recherche d'idempotence, il faudra toutefois prendre en compte la possibilité de mettre à jour l'application :

- si la version déployée est identique à la version décrite dans le playbook, aucune modification ne doit être effectuée ;
- si la version déployée est différente de la version décrite dans le playbook, la nouvelle webapp doit être déployée et Tomcat doit être redémarré.

## Multiples installations de Tomcat

Enfin, notre playbook crée une nouvelle installation de Tomcat pour chaque application déployée.

Il est possible de :

- déployer une unique installation de Tomcat :
  - un répertoire partagé contenant les répertoires `bin` et `lib`
  - variable `CATALINA_HOME` pointant vers ce répertoire
- créer une *instance* Tomcat par webapp :
  - un répertoire dédié contenant les répertoires `conf`, `logs`, `webapps` et `work`
  - variable `CATALINA_BASE` pointant vers ce répertoire

# Version modifiée

## Installation de Tomcat

Nous commençons donc par créer une liste de tâches repsonsables de l'installation *partagée* de Tomcat.

La première étape vérifie si Tomcat est installé (module [stat](http://docs.ansible.com/stat_module.html)). Puis, s'il n'est pas installé (directive `when: not st.stat.exists`), Tomcat est téléchargé et installé.

{% highlight yaml %}
  - name: check tomcat
    stat:
      path=/usr/share/tomcat7/bin
    register: st

  - name: download tomcat
    get_url:
      url=http://archive.apache.org/dist/tomcat/tomcat-7/v7.0.56/bin/apache-tomcat-7.0.56.tar.gz
      dest=/tmp/apache-tomcat-7.0.56.tar.gz
    when: not st.stat.exists

  - name: extract tomcat
    unarchive:
      src=/tmp/apache-tomcat-7.0.56.tar.gz
      copy=no
      dest=/tmp
    when: not st.stat.exists

  - name: copy tomcat
    shell: cp -r /tmp/apache-tomcat-7.0.56/* /usr/share/tomcat7 creates=/usr/share/tomcat7/bin
    when: not st.stat.exists
{% endhighlight %}

## Création d'une instance Tomcat

Tomcat étant installé *system-wide*, nous pouvons créer une instance dédié à notre application.

Nous créons l'arborescence nécessaire et nous déployons les fichiers de configuration à partir de templates (module [template](http://docs.ansible.com/template_module.html)). Nous déployons également un script d'init et enregistrons un service qui pourra être manipulé avec le module [service](http://docs.ansible.com/service_module.html).

Ici, nous n'utilisons que les modules [file](http://docs.ansible.com/file_module.html) et [template](http://docs.ansible.com/template_module.html) qui n'effectuent des modifications que lorsque celles-ci sont nécessaires, garantissant ainsi l'idempotence.

{% highlight yaml %}
{% raw %}
  - name: create tomcat instance
    file:
      name={{item}}
      state=directory
      owner=tomcat7
      group=tomcat7
    with_items:
      - "/PROD/mywebapp"
      - "/PROD/mywebapp/conf"
      - "/PROD/mywebapp/logs"
      - "/PROD/mywebapp/webapps"
      - "/PROD/mywebapp/work"

  - name: setup conf for tomcat instance
    template:
      src=tomcat-conf/{{item}}
      dest=/PROD/mywebapp/conf/{{item}}
      owner=tomcat7
      group=tomcat7
    with_items:
      - catalina.policy
      - catalina.properties
      - context.xml
      - logging.properties
      - server.xml
      - tomcat-users.xml
      - web.xml

  - name: install tomcat init script
    template:
      src=tomcat.sh
      dest=/etc/init.d/tomcat-mywebapp
      mode=0755
    notify:
      - register tomcat init script (add)
      - register tomcat init script (level)
{% endraw %}
{% endhighlight %}

Notez que les variables `CATALINA_HOME` et `CATALINA_BASE` sont définies dans le script d'init :

{% highlight bash %}
#CATALINA_HOME is the location of the bin files of Tomcat  
export CATALINA_HOME=/usr/share/tomcat7
 
#CATALINA_BASE is the location of the configuration files of this instance of Tomcat
export CATALINA_BASE=/PROD/mywebapp
{% endhighlight %}

## Déploiement de l'application

Enfin, nous pouvons déployer notre application. Pour rappel, nous souhaitons installer le WAR uniquement si celui-ci a été modifié. Nous allons donc comparer la somme MD5 du WAR installé (s'il existe) avec celle du WAR à déployer. Nous utilisons pour cela le module [stat](http://docs.ansible.com/stat_module.html) avec l'option `get_md5=True`.

Nous utilisons un [handler](http://docs.ansible.com/playbooks_intro.html#handlers-running-operations-on-change) pour redémarrer l'instance Tomcat dans le cas où le WAR est mis à jour (directive `notify: restart tomcat`).

{% highlight yaml %}
  - name: get the WAR
    get_url:
      url=http://.../mywebapp-1.0.war
      dest=/tmp/mywebapp-1.0.war

  - name: compute the MD5 of the new WAR
    stat:
      path=/tmp/mywebapp-1.0.war
      get_md5=True
    register: tmp_war_stat

  - name: compute the MD5 of the existing WAR
    stat:
      path=/PROD/mywebapp/webapps/mywebapp.war
      get_md5=True
    register: app_war_stat

  #- debug: var=tmp_war_stat.md5
  #- debug: var=app_war_stat.md5

  - name: copy the WAR
    shell: cp /tmp/mywebapp-1.0.war /PROD/mywebapp/webapps/mywebapp.war
    when: not app_war_stat.stat.md5 is defined or not tmp_war_stat.stat.md5 == app_war_stat.stat.md5
    notify: restart tomcat
{% endhighlight %}

Le handler de (re)-démarrage de Tomcat :

{% highlight yaml %}
  - name: restart tomcat
    service:
      name=tomcat-mywebapp
      state=restarted
{% endhighlight %}

Notez que, pour éviter tout redémarrage de Tomcat lors de la modification du WAR, l'`autodeploy` est désactivé dans le `server.xml` :

{% highlight yaml %}
      <Host ... autoDeploy="false">
{% endhighlight %}

## Exécution du playbook

Au premier run, Tomcat est installé, une instance Tomcat est créée, et la webapp est déployée :

<a class="image-popup" href="/images/ansible-screenshot1.png">
  <img src="/images/ansible-screenshot1.png" width="50%">
</a>

Si on relance le playbook sans avoir effectué de modification, aucun changement n'est effectué :

<a class="image-popup" href="/images/ansible-screenshot2.png">
  <img src="/images/ansible-screenshot2.png" width="50%">
</a>

Enfin, si on met à jour le WAR à déployer, le minimum d'actions est effectuée (copie du WAR et redémarrage de Tomcat) :

<a class="image-popup" href="/images/ansible-screenshot3.png">
  <img src="/images/ansible-screenshot3.png" width="50%">
</a>

# Conclusion

Le but premier d'Ansible n'est pas déployer des applications mais de provisionner des machines. Pourtant, pour peu qu'on prenne la peine de respecter les conventions (idempotence) et qu'on utilise pleinement les fonctionnalités de l'outil (modules, handlers), on peut déployer très simplement une application. Objectif atteint.

**Le playbook complet est disponible [sur Github](https://github.com/aseigneurin/ansible-sandbox).**