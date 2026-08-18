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

Ici et dans le reste du script, le choix d'un tableau associatif a été retenu, par son aspect pratique. Etant donné que chaque distribution possède un fichier unique, le script va tester l'existence de chacun de ces fichiers pour révéler la distribution. Dans ce contexte précis, chaque fichier signature permet de définir la distribution Linux du système (et donc son gestionnaire de paquets associé), une structure clé-valeur est donc cohérente avec cette relation. Ce tableau permet également une meilleure lisibilité qu'un enchainement de conditions et est plus extensible, très pratique si le script devait prendre en charge de nouvelles distributions. Après avoir établi ce dernier, une boucle sera utilisée pour tester l'existence des fichiers. 

La syntaxe : *${!informationsOS[@]}* permet de faire référence à toutes les clés du tableau, soit les chemins relatifs aux fichiers uniques. 

Pour finir, une dernière condition, agissant dans le cas où aucun fichier unique sur les chemins ci-après n'est trouvé. Cette dernière est essentielle, pour indiquer à l'utilisateur que la distribution de son système n'est pas prise en charge par le script et que son exécution serait donc non pertinente.

```bash
declare -A informationsOS
informationsOS[/etc/redhat-release]=yum
informationsOS[/etc/arch-release]=pacman
informationsOS[/etc/SuSE-release]=zypp
informationsOS[/etc/debian_version]=apt-get
informationsOS[/etc/alpine-release]=apk

for f in ${!informationsOS[@]}
do
    if [[ -f $f ]];then
gestionnaire=${informationsOS[$f]}
echo Gestionnaire de paquet: $gestionnaire
    fi
done

if [[ -z "$gestionnaire" ]]; then
echo "[INCONNU] Aucun gestionnaire de paquets reconnu n'a été détecté sur ce système. L'audit des mises à jour ne peut pas être effectué."
fi
```

**Limites connues**

Comme évoqué, une des limites du script réside dans la non-exhaustivité du tableau. En effet ce dernier couvre uniquement les distributions les plus répandues de Linux. Des systèmes plus récents tels que Fedora, NixOS, Void Linux etc. ne sont pas pris en charge. Cette non-exhaustivité a pour conséquence directe que, si aucune distribution n'est reconnue par le script, une grande partie de ce dernier ne peut pas s'exécuter et sera donc non pertinent. 

### Vérification des mises à jour

### Légitimité des binaires setuid

### Analyse des comptes à privilèges (UID/GID/shell)
