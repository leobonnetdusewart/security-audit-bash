# linux-security-audit
Script bash d'audit de sécurité Linux

## Présentation
Après mon début d’apprentissage autodidacte des environnements linux et de bash, notamment aux travers du Wargame Bandit (gratuit et édité par Over the Wire), j’ai réalisé que j’avais acquis de bonnes bases sur les différents aspects du réseaux linux, mais que je manquais de mise en pratique concrète, notamment en script. Ce script d’audit me permettra de développer mes compétences en code, tout en appliquant mes connaissances dans un contexte pratique.

## Fonctionnalités
liste à puces des grandes vérifications effectuées

## Prérequis
Bash, distributions Linux supportées, éventuellement les droits nécessaires pour exécuter le script

## Installation / Utilisation
Comment cloner le dépôt et lancer le script concrètement (chmod +x, ./nom_du_script)

## Détail technique

### Détection du gestionnaire de paquets
**Pourquoi cette étape est-elle primordiale ?**
Le point d'entrée de ce projet est la définition du gestionnaire de paquet pris en charge par la distribution Linux du système audité. En effet, comme évoqué plus tôt, la portabilité du script est primordiale, étant donné qu'il est courant, pour un parc informatique, d'utiliser plusieurs distributions Linux. Un outil d'audit fonctionnant sur une seule distribution aurait donc une utilité très limitée. On prendra donc en charge dans ce script les environnements suivants:
- Debian/Ubuntu
- Red Hat/CentOS
- Arch Linux
- SUSE/openSUSE
- Alpine Linux

**Choix technique et justification**
Dans ce contexte et dans le reste du script, le choix d'un tableau associatif à été retenu, par son aspect pratique et dans une démarche de cohérence et d'organisation globale.  


### Vérification des mises à jour

### Légitimité des binaires setuid

### Analyse des comptes à privilèges (UID/GID/shell)
