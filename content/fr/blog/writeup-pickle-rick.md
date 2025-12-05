---
title: "Write-up : Pickle Rick sur TryHackMe"
date: 2025-12-05
draft: false
categories: ["writeups"]
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