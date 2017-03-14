---
layout: post
title:  "Du dev à la prod avec Vagrant et Packer"
date:   2014-03-24 11:00:00
tags: vagrant packer aws
language: FR
origUrl: http://blog.ippon.fr/2014/03/24/du-dev-a-la-prod-avec-vagrant-et-packer/
origSource: le blog d'Ippon Technologies
---
Vous connaissez Vagrant ? [Vagrant](http://www.vagrantup.com/), c’est cet outil qui permet de démarrer des machines virtuelles avec une configuration donnée. Un très bon moyen de mettre au point et de diffuser des environnements de travail, et ce, de manière reproductible. Vagrant permet donc d’éviter le syndrome “works on my machine”.

Plusieurs cas d’usages sont possibles. Le plus courant est celui de faire démarrer un développeur from scratch. Mettez à sa disposition un Vagrantfile, laissez-lui saisir `vagrant up` et, en quelques secondes ou minutes, il disposera d’un environnement de développement avec l’ensemble des outils que vous aurez définis : Maven, ElasticSearch, NodeJS, etc.

Un autre cas d’usage est celui de reproduire un environnement complexe de production avec, par exemple, un load-balancer et plusieurs back-ends. C’est ce que permet le fonctionnement [multi-machine](http://docs.vagrantup.com/v2/multi-machine/index.html).

# Création de base box Vagrant

Dans la plupart des cas, Vagrant est utilisé à partir d’une base box telle que celles que l’on peut trouver sur [www.vagrantbox.es](http://www.vagrantbox.es/). Un des mécanismes de [provisioning](http://docs.vagrantup.com/v2/provisioning/index.html) de Vagrant (Shell, Chef Solo, Puppet, etc.) est alors utilisé pour configurer la base box.

Il peut être intéressant de constituer sa propre base box, par exemple :

- si vous voulez maîtriser le contenu de votre VM (c’est parfois une question de sécurité…) ;
- s’il n’existe pas déjà de base box répondant à vos besoins ;
- si le provisioning de votre box est long (compilation de paquets, téléchargements volumineux…) ;
- si vous voulez freezer une version exacte de votre environnement (pour ne pas dépendre d’un apt-get update, par exemple).

Les opérations permettant la création d’une base box sont documentées : il faut suivre le [guide de base](http://docs.vagrantup.com/v2/boxes/base.html) et appliquer les règles spécifiques au provider ([ici](http://docs.vagrantup.com/v2/virtualbox/boxes.html) pour VirtualBox). La procédure n’est pas très complexe mais elle est fastidieuse et surtout, une erreur est vite arrivée. C’est là qu’intervient Packer…

# Création de base box Vagrant avec Packer

[Packer](http://www.packer.io/) permet d’automatiser entièrement la création de base box Vagrant. Mais l’outil n’est pas réservé à Vagrant : il peut servir à préparer des containers Docker, des AMI pour Amazon AWS, etc.

Notez que, pour Vagrant, les base boxes sont généralement créées avec [Veewee](https://github.com/jedi4ever/veewee) (le précurseur de Packer) et il existe un [large repository de définitions](https://github.com/jedi4ever/veewee/tree/master/templates). Un outil de conversion des définitions Veewee vers des définitions Packer existe : [Veewee-to-packer](https://github.com/mitchellh/veewee-to-packer).

Une configuration Packer est définie avec un fichier JSON comportant des “builders” et des “provisioners” :

- Les builders servent à piloter les machines virtuelles. Il en existe pour VirtualBox, VMware, Amazon AWS, Google Cloud Platform, etc.
- Les provisioners servent à préparer la configuration logicielle à partir de scripts Shell ou avec Chef, Puppet, etc.

# En pratique

Dans cet exemple, nous cherchons à préparer une même configuration (Ubuntu 13.04 avec Node.JS 10.x) pour deux environnements d’exécution : VirtualBox et AWS. Nous aurons donc deux boxes Vagrant.

Notons tout de suite que la méthode d’identification sera différente suivant l’environnement :

- En local (VirtualBox), nous utiliserons la méthode traditionnelle de Vagrant, à savoir un user vagrant configuré avec une clé SSH non secure.
- Sur AWS, le mécanisme des “keypairs” sera utilisé, notre keypair étant déjà configurée dans l’interface de management AWS.

# Préparation de la box VirtualBox

La box VirtualBox sera constituée à partir d’une ISO Ubuntu que nous spécifierons par son URL sur le site de l’éditeur. Nous donnerons un fichier de “preseed” qui permettra de configurer l’installation (plus d’infos sur le preseeding : <https://help.ubuntu.com/14.04/installation-guide/amd64/apb.html>).

Le plus rapide est de partir d’une configuration Veewee existante. Téléchargeons un template Veewee pour Ubuntu depuis <https://github.com/jedi4ever/veewee/tree/master/templates/ubuntu-13.04-server-amd64>). Ce template est constitué d’une définition (“definition.rb”), d’un fichier de preseed (“preseed.cfg”) et de scripts shell :

```
$ cd ubuntu-13.04-server-amd64
$ find .
.
./apt.sh
./build_time.sh
./chef.sh
./cleanup.sh
./definition.rb
./preseed.cfg
./puppet.sh
./ruby.sh
./sudo.sh
./vagrant.sh
./vbox.sh
```

Lançons la conversion Veewee vers Packer :

```
$ veewee-to-packer definition.rb
Success! Your Veewee definition was converted to a Packer template!
The template can be found in the `template.json` file in the output
directory: output

Please be sure to run `packer validate` against the new template
to verify settings are correct.
```

Nous obtenons un fichier de définition Packer (“template.json”). Les autres fichiers sont conservés :

```
$ cd output
$ find .
.
./http
./http/preseed.cfg
./scripts
./scripts/apt.sh
./scripts/build_time.sh
./scripts/chef.sh
./scripts/cleanup.sh
./scripts/puppet.sh
./scripts/ruby.sh
./scripts/sudo.sh
./scripts/vagrant.sh
./scripts/vbox.sh
./template.json
```

La configuration doit être légèrement modifiée (bugs mineurs du convertisseur) : `virtualbox` doit être remplacé par `virtualbox-iso`, et la plupart des commands `<wait>` doivent être supprimées. La configuration du builder obtenue est alors :

```json
{
    "boot_command": [
        "<esc><esc><enter><wait>",
        "/install/vmlinuz noapic preseed/url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg ",
        "debian-installer=en_US auto locale=en_US kbd-chooser/method=us ",
        "hostname={{ .Name }} ",
        "fb=false debconf/frontend=noninteractive ",
        "keyboard-configuration/modelcode=SKIP keyboard-configuration/layout=USA keyboard-configuration/variant=USA console-setup/ask_detect=false ",
        "initrd=/install/initrd.gz -- <enter>"
    ],
    "boot_wait": "4s",
    "disk_size": 65536,
    "guest_additions_path": "VBoxGuestAdditions_{{.Version}}.iso",
    "guest_os_type": "Ubuntu_64",
    "http_directory": "http",
    "iso_checksum": "7d335ca541fc4945b674459cde7bffb9",
    "iso_checksum_type": "md5",
    "iso_url": "http://releases.ubuntu.com/13.04/ubuntu-13.04-server-amd64.iso",
    "shutdown_command": "echo 'shutdown -P now' > shutdown.sh; echo 'vagrant'|sudo -S sh 'shutdown.sh'",
    "ssh_password": "vagrant",
    "ssh_port": 22,
    "ssh_username": "vagrant",
    "ssh_wait_timeout": "10000s",
    "type": "virtualbox-iso",
    "virtualbox_version_file": ".vbox_version"
}
```

# Préparation de la box Amazon AWS

Préparons maintenant le builder pour Amazon AWS. Dans ce cas, nous ne pourrons pas utiliser l’ISO d’installation. Nous partirons d’une AMI (Amazon Machine Image) préparée par l’éditeur et que nous sélectionnerons sur [cloud-images.ubuntu.com/locator/ec2/](http://cloud-images.ubuntu.com/locator/ec2/).

Notre builder sera configurée comme suit :

```json
{
    "type": "amazon-ebs",
    "access_key": "{{user `aws_access_key`}}",
    "secret_key": "{{user `aws_secret_key`}}",
    "region": "eu-west-1",
    "source_ami": "ami-dea653a9",
    "instance_type": "t1.micro",
    "ssh_username": "ubuntu",
    "ami_name": "ubuntu-13.04__Node.JS"
}
```

# Fin de la préparation du template Packer

Il nous reste à modifier les provisioners :

- Nous pouvons retirer l’installation de Ruby, Chef et Puppet qui ne nous intéressent pas.
- Les “Guest Additions” ne doivent être installés que pour VirtualBox.
- Le user vagrant ne doit pas être créé pour AWS.
- Nous ajoutons un script pour installer Node.JS.

La configuration des provisioners devient :

```json
"provisioners": [
    {
        "execute_command": "echo 'vagrant'|sudo -S sh '{{.Path}}'",
        "scripts": [
            "scripts/build_time.sh",
            "scripts/apt.sh"
        ],
        "type": "shell"
    },
    {
        "execute_command": "echo 'vagrant'|sudo -S sh '{{.Path}}'",
        "only": [
            "virtualbox-iso"
        ],
        "scripts": [
            "scripts/vbox.sh",
            "scripts/vagrant.sh"
        ],
        "type": "shell"
    },
    {
        "execute_command": "echo 'vagrant'|sudo -S sh '{{.Path}}'",
        "scripts": [
            "scripts/sudo.sh",
            "scripts/nodejs.sh",
            "scripts/cleanup.sh"
        ],
        "type": "shell"
    }
]
```

Et notre script nodejs.sh est définit comme suit :

```
apt-get -y install software-properties-common
add-apt-repository -y ppa:chris-lea/node.js
apt-get -y update
apt-get -y install nodejs
```

Enfin, nous rajoutons un post-processeur qui créera les box Vagrant à partir de l’AMI AWS et de la VM VirtualBox :

```json
"post-processors": [
    {
        "keep_input_artifact": true,
        "type": "vagrant"
    }
]
```

Attention, sans le paramètre `keep_input_artifact` à `true`, le post-processeur supprimera l’AMI ce qui rendra la box inutilisable…

# Création de la box VirtualBox

Nous pouvons maintenant lancer la préparation des boxes. Packer lance par défaut la création de toutes les boxes en parallèle. Toutefois, pour commencer, nous allons uniquement builder l’image VirtualBox grâce au flag `-only=virtualbox-iso`.

Packer va télécharger l’ISO Ubuntu et les “Guest Additions”. Une VM sera créée dans VirtualBox. Celle-ci va booter sur l’ISO et, quand le SSH sera prêt, le provisioning sera effectué (par scripts shell, dans notre cas). Enfin, une box Vagrant sera créée.

```
$ packer build -only=virtualbox-iso template.json
virtualbox-iso output will be in this color.

==> virtualbox-iso: Downloading or copying Guest additions checksums
   virtualbox-iso: Downloading or copying: http://download.virtualbox.org/virtualbox/4.3.6/SHA256SUMS
==> virtualbox-iso: Downloading or copying Guest additions
   virtualbox-iso: Downloading or copying: http://download.virtualbox.org/virtualbox/4.3.6/VBoxGuestAdditions_4.3.6.iso
==> virtualbox-iso: Downloading or copying ISO
   virtualbox-iso: Downloading or copying: http://releases.ubuntu.com/13.04/ubuntu-13.04-server-amd64.iso
==> virtualbox-iso: Starting HTTP server on port 8081
==> virtualbox-iso: Creating virtual machine...
==> virtualbox-iso: Creating hard drive...
==> virtualbox-iso: Creating forwarded port mapping for SSH (host port 3213)
==> virtualbox-iso: Executing custom VBoxManage commands...
   virtualbox-iso: Executing: modifyvm packer-virtualbox-iso --memory 512
   virtualbox-iso: Executing: modifyvm packer-virtualbox-iso --cpus 1
==> virtualbox-iso: Starting the virtual machine...
==> virtualbox-iso: Waiting 4s for boot...
==> virtualbox-iso: Typing the boot command...
==> virtualbox-iso: Waiting for SSH to become available...
==> virtualbox-iso: Connected to SSH!
==> virtualbox-iso: Uploading VirtualBox version info (4.3.6)
==> virtualbox-iso: Uploading VirtualBox guest additions ISO...
==> virtualbox-iso: Provisioning with shell script: scripts/build_time.sh
   virtualbox-iso: [sudo] password for vagrant:
==> virtualbox-iso: Provisioning with shell script: scripts/apt.sh
...
==> virtualbox-iso: Provisioning with shell script: scripts/sudo.sh
...
==> virtualbox-iso: Provisioning with shell script: scripts/nodejs.sh
...
==> virtualbox-iso: Provisioning with shell script: scripts/vagrant.sh
...
==> virtualbox-iso: Provisioning with shell script: scripts/cleanup.sh
...
==> virtualbox-iso: Gracefully halting virtual machine...
   virtualbox-iso: [sudo] password for vagrant:
   virtualbox-iso: Broadcast message from root@packer-virtualbox-iso
   virtualbox-iso: (unknown) at 14:23 ...
   virtualbox-iso:
   virtualbox-iso: The system is going down for power off NOW!
==> virtualbox-iso: Preparing to export machine...
   virtualbox-iso: Deleting forwarded port mapping for SSH (host port 3213)
==> virtualbox-iso: Exporting virtual machine...
==> virtualbox-iso: Unregistering and deleting virtual machine...
==> virtualbox-iso: Running post-processor: vagrant
==> virtualbox-iso (vagrant): Creating Vagrant box for 'virtualbox' provider
   virtualbox-iso (vagrant): Copying from artifact: output-virtualbox-iso/packer-virtualbox-iso-disk1.vmdk
   virtualbox-iso (vagrant): Copying from artifact: output-virtualbox-iso/packer-virtualbox-iso.ovf
   virtualbox-iso (vagrant): Renaming the OVF to box.ovf...
   virtualbox-iso (vagrant): Compressing: Vagrantfile
   virtualbox-iso (vagrant): Compressing: box.ovf
   virtualbox-iso (vagrant): Compressing: metadata.json
   virtualbox-iso (vagrant): Compressing: packer-virtualbox-iso-disk1.vmdk
Build 'virtualbox-iso' finished.

==> Builds finished. The artifacts of successful builds are:
--> virtualbox-iso: 'virtualbox' provider box: packer_virtualbox-iso_virtualbox.box
```

<img src="/images/vagrant-packer.png"/>

Le fichier “packer_virtualbox-iso_virtualbox.box” a été créé. Il s’agit d’un fichier TAR. Inspectons-le :

```
$ tar xvf ../packer_virtualbox-iso_virtualbox.box
x Vagrantfile
x box.ovf
x metadata.json
x packer-virtualbox-iso-disk1.vmdk
```

```
$ cat Vagrantfile
# The contents below were provided by the Packer Vagrant post-processor
Vagrant.configure("2") do |config|
 config.vm.base_mac = "08002751E780"
end
# The contents below (if any) are custom contents provided by the
# Packer template during image build.
```

```
$ cat metadata.json
{"provider":"virtualbox"}
```

La box contient une image disque pour VirtualBox ainsi que des fichiers de configuration pour Vagrant.

# Création de la box AWS

Lançons ensuite le build de l’image AWS avec l’option `-only=amazon-ebs`. Les clés d’accès AWS seront spécifiées sur la ligne de commande.

Packer va démarrer une instance EC2 puis, quand le SSH sera prêt, lancera le provisioning. L’instance sera ensuite arrêtée pour permettre la création d’une AMI. Enfin, une box Vagrant sera créée.

```
$ packer build -only=amazon-ebs -var "aws_access_key=<...>" -var "aws_secret_key=<...>" template.json
amazon-ebs output will be in this color.

==> amazon-ebs: Creating temporary keypair: packer 52e8e83f-d8bf-1a2e-45d0-62141afa9d85
==> amazon-ebs: Creating temporary security group for this instance...
==> amazon-ebs: Authorizing SSH access on the temporary security group...
==> amazon-ebs: Launching a source AWS instance...
   amazon-ebs: Instance ID: i-8d587ac3
==> amazon-ebs: Waiting for instance (i-8d587ac3) to become ready...
==> amazon-ebs: Waiting for SSH to become available...
==> amazon-ebs: Connected to SSH!
==> amazon-ebs: Provisioning with shell script: scripts/build_time.sh
==> amazon-ebs: Provisioning with shell script: scripts/apt.sh
...
==> amazon-ebs: Provisioning with shell script: scripts/sudo.sh
...
==> amazon-ebs: Provisioning with shell script: scripts/nodejs.sh
...
==> amazon-ebs: Provisioning with shell script: scripts/vagrant.sh
...
==> amazon-ebs: Provisioning with shell script: scripts/cleanup.sh
...
==> amazon-ebs: Stopping the source instance...
==> amazon-ebs: Waiting for the instance to stop...
==> amazon-ebs: Creating the AMI: ubuntu-13.04__Node.JS
   amazon-ebs: AMI: ami-9a07f0ed
==> amazon-ebs: Waiting for AMI to become ready...
==> amazon-ebs: Terminating the source AWS instance...
==> amazon-ebs: Deleting temporary security group...
==> amazon-ebs: Deleting temporary keypair...
==> amazon-ebs: Running post-processor: vagrant
==> amazon-ebs (vagrant): Creating Vagrant box for 'aws' provider
   amazon-ebs (vagrant): Compressing: Vagrantfile
   amazon-ebs (vagrant): Compressing: metadata.json
Build 'amazon-ebs' finished.

==> Builds finished. The artifacts of successful builds are:
--> amazon-ebs: AMIs were created:

eu-west-1: ami-9a07f0ed
--> amazon-ebs: 'aws' provider box: packer_amazon-ebs_aws.box
```

Inspectons le fichier “packer_amazon-ebs_aws.box” :

```
$ tar xvf packer_amazon-ebs_aws.box
x Vagrantfile
x metadata.json
```

```
$ cat Vagrantfile
# The contents below were provided by the Packer Vagrant post-processor
Vagrant.configure("2") do |config|
 config.vm.provider "aws" do |aws|
aws.region_config "eu-west-1", ami: "ami-9a07f0ed"
 end
end
# The contents below (if any) are custom contents provided by the
# Packer template during image build.
```

```
$ cat metadata.json
{"provider":"aws"}
```

Contrairement à la box VirtualBox, cette box ne contient pas d’image disque. En revanche, nous avons bien une référence à l’AMI qui a été créée.

# Ajout des base boxes Vagrant

Les opérations de build ont créé deux fichiers de boxes : packer_virtualbox-iso_virtualbox.box et packer_amazon-ebs_aws.box. Nous pouvons les ajouter à Vagrant :

```
$ vagrant box add ubuntu-13.04-amd64-nodejs packer_virtualbox-iso_virtualbox.box
Downloading box from URL: file:/Users/aseigneurin/dev/vms/vagrant/ubuntu-13.04-server-amd64/packer_virtualbox-iso_virtualbox.box
Extracting box...te: 16.0M/s, Estimated time remaining: 0:00:01)
Successfully added box 'ubuntu-13.04-amd64-nodejs' with provider 'virtualbox'!
```

```
$ vagrant box add ubuntu-13.04-amd64-nodejs packer_amazon-ebs_aws.box
Downloading box from URL: file:/Users/aseigneurin/dev/vms/vagrant/ubuntu-13.04-server-amd64/packer_amazon-ebs_aws.box
Extracting box...e: 0/s, Estimated time remaining: --:--:--)
Successfully added box 'ubuntu-13.04-amd64-nodejs' with provider 'aws'!
```

```
$ vagrant box list
ubuntu-13.04-amd64-nodejs (aws)
ubuntu-13.04-amd64-nodejs (virtualbox)
```

Notez que plusieurs base boxes peuvent porter le même nom si leurs providers sont différents. C’est le cas ici.

Nous pouvons alors créer une box Vagrant reposant sur notre base box :

```
$ vagrant init ubuntu-13.04-amd64-nodejs
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```

Nous n’avons pas spécifié le provider (VirtualBox ou AWS). Ceci est défini avec le paramètre `--provider` de la commande `up`.

Pour lancer la VM en local, dans VirtualBox, nous lançons donc :

```
$ vagrant up --provider=virtualbox
Bringing machine 'default' up with 'virtualbox' provider...
[default] Importing base box 'ubuntu-13.04-amd64-nodejs'...
...
```

Et pour la lancer sur Amazon :

```
$ vagrant up --provider=aws
Bringing machine 'default' up with 'aws' provider...
...

There are errors in the configuration of this machine. Please fix
the following errors and try again:

AWS Provider:
* A secret access key is required via "secret_access_key"
* An AMI must be configured via "ami"
```

Plusieurs problèmes apparaissent :

- La clé d’accès AWS n’est pas configurée, ce qui est plutôt rassurant. Pour y remédier, le plugin Vagrant-AWS lit automatiquement les variables d’environnement AWS_ACCESS_KEY et AWS_SECRET_KEY. On peut donc lancer la commande suivante :

```
$ AWS_ACCESS_KEY=<...> AWS_SECRET_KEY=<...> vagrant up --provider=aws
```

- L’identifiant de l’AMI est manquant. Il s’agit probablement d’un bug du plugin puisque le fichier Vagrantfile contenu dans notre base box contenait bien cette information ainsi que le nom de la région. Il est possible de contourner ce problème en rajoutant uniquement la région dans la configuration locale :

```
config.vm.provider :aws do |aws, override|
  aws.region = "eu-west-1"
end
```

- Il nous faut également indiquer le type d’instance, le ou les security groups, la keypair, ainsi que les informations d’accès SSH :

```
aws.instance_type = "t1.micro"
aws.security_groups = "default"
aws.keypair_name = "..."
override.ssh.username = "ubuntu"
override.ssh.private_key_path = "..."
```

Une fois ces modifications effectuées, on peut alors relancer notre vagrant up :

```
$ AWS_ACCESS_KEY=<...> AWS_SECRET_KEY=<...> vagrant up --provider=aws
Bringing machine 'default' up with 'aws' provider...
[default]  -- Type: m1.small
[default]  -- AMI: ami-040ff873
[default]  -- Region: eu-west-1
...
[default] Waiting for instance to become "ready"...
[default] Waiting for SSH to become available…
[default] Machine is booted and ready for use!
...
```

Et, sur AWS comme avec VirtualBox, il est alors possible de se connecter à la VM et d’utiliser Node.JS.

Sur VirtualBox, nous obtenons :

```
$ vagrant ssh
Welcome to Ubuntu 13.04 (GNU/Linux 3.8.0-19-generic x86_64)
...
vagrant@packer-virtualbox-iso:~$ node -v
v0.10.25
```

Et sur AWS, seul le prompt change :

```
$ vagrant ssh
Welcome to Ubuntu 13.04 (GNU/Linux 3.8.0-35-generic x86_64)
...
ubuntu@ip-172-31-30-2:~$ node -v
v0.10.25
```

# Conclusion

Vagrant est un outil très pratique pour unifier les environnements de développement et de production. Packer rend réellement la chose possible en permettant d’assembler des base boxes identiques sur des environnements différents.

Alors, certes, la chose n’est pas parfaite :

- Nos boxes ne sont pas parfaitement identiques, nous sommes partis de deux sources similaires (une ISO pour VirtualBox, une AMI pour AWS) que nous avons provisionnées de la même manière. L’idéal serait de pouvoir convertir une box dans l’autre format, par exemple de convertir une image VirtualBox en AMI.
- Créer une box depuis une ISO est loin d’être trivial : la séquence de boot et le preseed sont peu ou pas documentés. Il vaut mieux dans ce cas commencer avec une image OVF telle celles qu’on peut trouver sur [cloud-images.ubuntu.com/releases/raring/release/](http://cloud-images.ubuntu.com/releases/raring/release/) (les downloads sont au bas de la page).
- Le plugin AWS pour Vagrant est encore relativement jeune et gagnerait à être stabilisé.

Toutefois, le but est atteint et c’est l’essentiel : les développeurs peuvent utiliser Vagrant avec VirtualBox tandis que les ops utilisent un provider cloud.

# Ressources

Le code utilisé dans ce post est disponible sur GitHub : [github.com/aseigneurin/vms](https://github.com/aseigneurin/vms)

Vagrant : [www.vagrantup.com/](http://www.vagrantup.com/)

Packer : [www.packer.io/](http://www.packer.io/)

AMIs Ubuntu : [cloud-images.ubuntu.com/locator/ec2/](http://cloud-images.ubuntu.com/locator/ec2/)