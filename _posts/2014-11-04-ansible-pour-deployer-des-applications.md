---
layout: post
title:  "Ansible pour déployer des applications"
date:   2014-11-04 11:00:00
tags: ansible
language: FR
---
[Ansible](http://www.ansible.com/) est un excellent outil de provisioning. L'outil n'est donc a priori pas prévu pour déployer des applications. Ce post traite des problèmes que cela pose et d'une manière de les résoudre.

L'exercice a porté sur le déploiement d'une webapp Java, et donc d'un WAR, dans un Tomcat.

# Version initiale et ses défauts

La première version d'un playbook Ansible que j'ai écrite était la reproduction de scripts de déploiement "maison". Cette version était impérative et assez brutale, à savoir :

- on vérifie si une instance Tomcat existe
- si l'instance existe, on l'arrête et on la supprime
- on télécharge Tomcat et on l'installe
- on télécharge le WAR et on le déploie dans Tomcat
- on démarre Tomcat

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
      url=http://www.us.apache.org/dist/tomcat/tomcat-7/v7.0.56/bin/apache-tomcat-7.0.56.tar.gz
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

En clair, si notre webapp est déjà déployée et que nous lançons notre playbook, **Ansible ne devrait rien modifier et Tomcat ne devrait pas redémarrer**. Ce n'est pas le cas puisque, à chaque exécution, l'instance Tomcat va être détruite et re-créée.

Nous allons donc chercher à déployer Tomcat ainsi que le WAR uniquement en cas de besoin.

## Problématique des mises à jour de l'application

Dans notre recherche d'idempotence, il faut toutefois prendre en compte la possibilité de mettre à jour l'application :

- si la version déployée est identique à la version décrite dans le playbook, aucune modification ne doit être effectuée
- si la version déployée est différente de la version décrite dans le playbook, la nouvelle webapp doit être déployée et Tomcat doit être redémarré.

# 


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
    changed_when: False

  - name: compute the MD5 of the existing WAR
    stat:
      path=/PROD/mywebapp/webapps/mywebapp.war
      get_md5=True
    register: app_war_stat
    changed_when: False

  #- debug: var=tmp_war_stat.md5
  #- debug: var=app_war_stat.md5

  - name: copy the WAR
    shell: cp /tmp/mywebapp-1.0.war /PROD/mywebapp/webapps/mywebapp.war
    when: not app_war_stat.stat.md5 is defined or not tmp_war_stat.stat.md5 == app_war_stat.stat.md5
    notify: restart tomcat
{% endhighlight %}


Le code est disponible : https://github.com/aseigneurin/ansible-sandbox