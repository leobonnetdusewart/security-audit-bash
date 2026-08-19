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
**Pourquoi cette étape ?**

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
**Pourquoi cette étape ?**

Lorsqu'un attaquant s'infiltre dans un réseau, son objectif est de s’assurer qu’il possède assez de permissions pour atteindre ses objectifs, allant à l’encontre des intérêts du propriétaire. Red Canary, dans son rapport sur les techniques de menaces, insiste sur l'importance de la mise à jour des fichiers possédant le bit setuid (Red Canary, s. d.). En effet, les vulnérabilités liées aux setuids sont fréquemment révélées et corrigées, ce type de fichiers pouvant être exploité à des fins malveillantes s'ils présentent une vulnérabilité (notamment une escalade de privilèges non autorisée), on tient à les tenir à jour. 

Un exemple concret: Heartbleed, une vulnérabilité critique de la librairie OpenSSL, permettant à des attaquants d'avoir accès à des données sensibles depuis la mémoire d'un serveur vulnérable. Bien que cette vulnérabilité ait été découverte et corrigée en 2014, elle reste encore aujourd'hui un problème pour les systèmes non à jour, notamment dans le secteur public, financier et de la snaté, reposant fréquemment sur une architecture dépassée et ne supportant pas les dernières mises à jour OpenSSL (Linuxvox, 2026). 

**Choix technique et justification**

Pour rester cohérent et garder un fil conducteur vis-à-vis de la première partie du script, on va ici aussi utiliser un tableau associatif. Cette fois-ci la clé est le gestionnaire de paquets (préalablement défini) et la valeur associée est une commande permettant de visualiser les paquets nécessitant une mise à jour. Le script présente également une condition, enregistrant la commande associée au gestionnaire de paquet dans la variable "maj". 

On utilise ensuite la commande "eval", cruciale pour cette partie du script, étant donné qu'on désire que la commande soit traitée correctement, comme si on la rentrait manuellement dans bash. C'est particulièrement pertinent pour que la redirection : "2>/dev/null" soit bien prise en compte, ce qui n'était pas le cas initialement.

L'utilisation de la variable spéciale "$?" permet de tester si la dernière commande exécutée par le script renvoie une erreur ou non. Si c'est le cas, un message d'erreur sera renvoyé, sinon le script poursuit. C'est important dans ce contexte de tester la fonctionnalité de la commande "eval $maj" pour ne pas qu'on interprète un message d'erreur comme entrée pour la commande "echo "$cbmaj" | wc -l".

Pour calculer le nombre de paquets nécessitant une mise à jour, on utilise "wc -l" qui va compter le nombre de lignes que va renvoyer la commande. C'est cohérent, étant donné que les paquets nécessitant une mise à jour sont affichés un par un, chacun sur sa propre ligne.

Pour finir, une simple condition afin d'afficher à l'utilisateur les informations pertinentes quant à l'audit. Notez ici qu'on affiche "nblignes - 1", correspondant en réalité à la ligne "Listing... Done" s'affichant mais ne représentant pas un paquet nécessitant une mise à jour.

```Bash
declare -A commandesOS
commandesOS[yum]="yum check-update 2>/dev/null"
commandesOS[pacman]="pacman -Qu 2>/dev/null"
commandesOS[zypp]="zypper list-updates 2>/dev/null"
commandesOS[apt-get]="apt list --upgradable 2>/dev/null"
commandesOS[apk]="apk list -u 2>/dev/null"

if [[ -v "commandesOS[$gestionnaire]" ]]; then
maj=${commandesOS[$gestionnaire]}
fi

cbmaj=$(eval $maj)

if [[ $? -ne 0 ]]; then
echo "[ERREUR] Impossible de vérifier les mises à jour pour le gestionn>
else
nblignes=$(echo "$cbmaj" | wc -l)
if [[ $nblignes -eq 1 ]]; then
echo "[OK] Aucune mise à jour de sécurité en attente pour le gestionnai>
else
echo "[ATTENTION] $((nblignes - 1)) paquet(s) en attente de mise à jour>
fi
fi
```
**Limites connues**

Pour des raisons techniques, l'hypothèse selon laquelle une seule ligne d'en-tête présente quand on exécute la commande n'est, à ce jour, vérifiée uniquement avec le gestionnaire "apt". Ce code suit donc le format de la commande "apt list --upgradable" mais ne garantit pas que les autres suivent exactement ce format. Le code devrait s'adapter à chaque distribution Linux prise en charge.

Etant donné que ce script est utilisé dans le cadre d'un audit, aucune modification du système ne doit être effectuée, il doit simplement l'analyser. On suppose donc que l'utilisateur a rafraichî manuellement le catalogue de paquets au préalable, si ce n'est pas le cas, cette partie du script pourrait se révéler non pertinente.

### Légitimité des binaires setuid
**Pourquoi cette étape ?**

Tout d'abord, il est essentiel de rappeler qu'un setuid est un mécanisme, qui posé sur un fichier exécutable permet d'octroyer des permissions particulières, en plus des permissions "classiques" de lecture et d'écriture. Quand setuid est activé sur un exécutable, tout utilisateur capable de lancer ce fichier l'exécute automatiquement avec les privilèges du propriétaire du fichier (souvent root) et/ou de son groupe. Cependant, le setuid représente un risque de sécurité élevé pour les systèmes mal configurés. “Si un programme disposant de l'autorisation setuid présente des vulnérabilités ou est mal configuré, il peut être exploité par des utilisateurs malveillants pour obtenir un accès non autorisé à des données sensibles ou effectuer des actions non autorisées avec des privilèges élevés.” (Guide Complet Sur Setuid | Lenovo France, s. d.). 

Un des risques cruciaux relatifs à ce type de fichiers réside dans leur légitimité. En effet, un attaquant ayant déjà obtenu un accès initial à une machine peut lui-même poser le bit setuid sur un binaire qui ne l'avait pas à l'origine, afin de se garantir un moyen simple de récupérer les privilèges root plus tard, même après avoir perdu son accès initial (Red Canary, s. d.). C’est une technique classique de maintien d'accès (persistence) en cybersécurité. 

### Analyse des comptes à privilèges (UID/GID/shell)
