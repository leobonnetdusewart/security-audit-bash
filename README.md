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

Tout d'abord, il est essentiel de rappeler qu'un setuid est un mécanisme qui, posé sur un fichier exécutable, permet d'octroyer des permissions particulières, en plus des permissions "classiques" de lecture et d'écriture. Quand setuid est activé sur un exécutable, tout utilisateur capable de lancer ce fichier l'exécute automatiquement avec les privilèges du propriétaire du fichier (souvent root) et/ou de son groupe. Cependant, le setuid représente un risque de sécurité élevé pour les systèmes mal configurés. “Si un programme disposant de l'autorisation setuid présente des vulnérabilités ou est mal configuré, il peut être exploité par des utilisateurs malveillants pour obtenir un accès non autorisé à des données sensibles ou effectuer des actions non autorisées avec des privilèges élevés.” (Guide Complet Sur Setuid | Lenovo France, s. d.). 

Un des risques cruciaux relatifs à ce type de fichiers réside dans leur légitimité. En effet, un attaquant ayant déjà obtenu un accès initial à une machine peut lui-même poser le bit setuid sur un binaire qui ne l'avait pas à l'origine, afin de se garantir un moyen simple de récupérer les privilèges root plus tard, même après avoir perdu son accès initial (Red Canary, s. d.). C’est une technique classique de maintien d'accès (persistence) en cybersécurité. 

D’après The Linux Documentation Project, un très bon moyen de détecter une attaque locale sur un système est donc de vérifier fréquemment l’intégrité des binaires de haute importance, tels que les setuid (Files And File System Security, s. d.). Il est important également de préciser qu’un setuid illégitime n’est pas forcément malveillant par nature et a très bien pu être ajouté par un administrateur, ici on cherche surtout à prévenir les attaques et à indiquer les points de vigilance, pas à juger si un fichier est malveillant ou non. Le script va donc permettre d'analyser les setuid présents dans le système audité et de définir s'ils sont rattachés à un paquet officiel ou non, auquel cas potentiellement problématiques quant à la sécurité du système. 

**Choix technique et justification**

La première substitution de commande dans la variable *"std"* permet d'afficher les chemins relatifs aux fichiers avec le bit setuid d'activé, 4000 étant la représentation octale de ce dernier. L’utilisation de l’option “-xdev” n’est pas anodine. En effet, la première version de la commande ne présentait pas cette caractéristique, et le script analysait une quantité faramineuse de fichiers, prenant un temps considérable avant de donner un résultat. “-xdev” permet de résoudre ce souci, en empêchant find de “descendre” dans un système de fichiers différent de celui du point de départ.

Une nouvelle fois, l'utilisation d'un tableau associatif est retenue, avec pour clé, le gestionnaire de paquets défini dès la première étape du script, et avec comme valeur, une commande propre au gestionnaire. Ces commandes servent toutes le même objectif, couplées avec le chemin d'un fichier, elles permettent de définir si un fichier est rattaché à un paquet officiel.

Une boucle va ensuite tester chaque fichier avec la commande variable *$com "$fichier"*, c'est donc l'accouplement de la commande présente en valeur dans notre tableau, ainsi que le chemin du fichier pour lequel elle s'applique. La variable spéciale *$?* est à nouveau utilisée, testant si la commande précédente renvoie un code de sortie différent de 0 (relatif donc à un échec). Ces fichiers, dont la commande renverrait donc un échec, sont qualifiés donc comme potentiellement illégitimes étant donné qu'ils ne sont rattachés à aucun paquet officiel.

```bash
std=$(find / -xdev -perm -4000 -type f 2>/dev/null)

declare -A proprio
proprio[yum]="rpm -qf"
proprio[pacman]="pacman -Qo"
proprio[zypp]="zypper what-provides"
proprio[apt-get]="dpkg -S"
proprio[apk]="apk info --who-owns"

if [[ -v "proprio[$gestionnaire]" ]]; then
com=${proprio[$gestionnaire]}
fi

for fichier in $std
do
resultat=$(eval $com "$fichier" 2>/dev/null)
if [[ $? -ne 1 ]]; then
compteur=$((compteur + 1))
fi
done

if [[ $compteur -eq 0 ]]; then
echo "[OK] Aucun binaire setuid non officiel détecté."
else
echo "[ATTENTION] $compteur binaire(s) setuid non rattaché(s) à un paquet officiel ont été détectés sur ce système. Vérification manuelle requise."
```
**Limites connues**

Comme évoqué plus tôt, ce script permet d'identifier combien de setuid ne sont pas rattachés à des paquets officiels. Cependant il ne permet pas de qualifier la légitimité d'un fichier dans la mesure où ce dernier a été rajouté manuellement par un administrateur. Le script compense cette limite en alertant l'utilisateur plutôt qu'en tranchant lui-même. Il permet donc d'envoyer un "rappel" à l'utilisateur pour qu'il vérifie ses setuid et qu'il s'assure qu'ils sont bien légitimes (dans le cas où des fichiers non rattachés à des paquets sont détectés).  

L'utilisation de l'option *xdev*, bien que nécessaire pour garantir la performance du script, limite le champ de recherche de *find*, qui pourrait potentiellement laisser des binaires setuid cachés non détectés.

Pour finir, la détection du gestionnaire de paquet de la première étape du script est essentielle à l'exécution de cette partie. Dans le cas où l'utilisateur utiliserait une distribution différente des 5 couvertes ici, cette partie ne pourrait donc pas être appliquée sur son système. 

### Analyse des comptes à privilèges (UID/GID/shell)
**Pourquoi cette étape ?**

Le fichier /etc/passwd est une base de données comportant des informations réparties en 7 champs distincts, concernant les utilisateurs du système. D'après Wikipédia, les champs affichent les informations suivantes, dans l'ordre: 
- Le nom de l'utilisateur (login name)
- Les informations relatives au mot de passe de l'utilisateur (généralement "x", le mot de passe étant stocké dans un autre fichier)
- L'identifiant utilisateur, aussi appelé "UID"
- L'identifiant de groupe, aussi appelé "GID"
- Un champ dédié à un commentaire concernant la personne ou le compte
- Le chemin vers le répertoire personnel de l'utilisateur
- le programme qui est lancé chaque fois que l'utilisateur se connecte au système. Pour un utilisateur interactif, il s'agit généralement d'une interface en ligne de commande.
(Wikipédia, s. d.)

Les trois champs qui vont nous intéresser et être pertinents pour notre audit, sont ceux relatifs aux droits des utilisateurs (UID/GID) et celui relatif au shell de l'utilisateur. 

Il est crucial de vérifier qu'aucun utilisateur ne possède les mêmes droits que root (UID=0) pour garantir la sécurité du système. En effet, un accès total et sans restriction à ce dernier permettrait à un utilisateur de lire, modifier ou supprimer absolument n'importe quel fichier sur le système. Un UID de 0 permettrait également à un attaquant de créer, modifier ou ajouter des utilisateurs, agir sur leurs mots de passe, supprimer des logs (donc des traces de son intrusion), ou encore d'installer n'importe quel logiciel avec n'importe quel niveau de privilèges. Ça représenterait donc un danger imminent pour le propriétaire du système, d'où l'importance de le détecter.

Pour les mêmes raisons, le script va également identifier les groupes d'utilisateurs possédant les droits root (GID=0), pour éviter qu'un groupe ait accès à des fichiers/dossiers réservés à l'administration. Appartenir au groupe root est bien moins "grave" qu'avoir un UID de 0 et peut totalement être légitime pour un administrateur, il reste néanmoins intéressant de vérifier manuellement la légitimité des utilisateurs possédant cette caractéristique.

Vérifier l'existence de shells interactifs pour des utilisateurs système est également cohérent avec notre objectif d'audit. En effet, ces comptes sont censés exister pour faire tourner un service précis et non pas pour qu'un humain/attaquant s'y connecte. La présence d'un shell interactif pour un de ces comptes (/bin/bash au lieu de /usr/sbin/nologin), pourrait représenter une véritable porte d'entrée exploitable. On va donc croiser deux critères : la présence d'un shell interactif, et un UID inférieur à 1000, cette combinaison permet d'identifier spécifiquement les comptes système qui ne devraient pas avoir cette capacité de connexion. 

Le script va en plus vérifier l'existence de shells interactifs pour n'importe quel utilisateur (peu importe l'UID), particulièrement pertinent dans le scénario où un attaquant créerait un compte avec un UID normal (≥ 1000), donc invisible pour le croisement précédent, mais détectable via ce comptage total.

**Choix technique et justification**

L'utilisation de la commande *awk* est particulièrement pertinente dans le contexte actuel, tant cette commande permet, une fois couplée avec l'option *-F*, de traiter du texte structuré en colonnes/champs, en prenant en compte le séparateur de champs (ici ":"). Elle permet donc d'afficher le 3ème (UID), 4ème (GID) et 7ème champ (shell).

On met ensuite en place des compteurs, qui vont nous servir par la suite. 

La boucle while qui suit, permet de lire le résultat de la commande précédente par le biais de la redirection *done <<< "$userid"*, présente à la fin de la boucle. Les champs interprétés par *$3, $4, $7* sont respectivement stockés dans les variables *var1 var2 var3*. 

La première condition permet de calculer le nombre d'utilisateurs possédant un UID de 0, équivalent donc à root. Ce nombre est ensuite stocké dans le compteur établi au préalable dans *nb_uid_root*. 

La deuxième condition permet d'analyser le shell de chaque utilisateur, et de définir si on peut interagir avec. Pour garantir de la portabilité du script sur l'ensemble des distributions prises en charge depuis le début, on prend en compte tous les shells interactifs possibles. On ajoute une variable interne dans le cas où on peut interagir avec le shell, qu'on utilisera plus tard. Dans le même cas, on incrémente notre compteur *nb_shell_interactif*. 

La condition suivante possède la même structure que la première et veille à analyser le GID de chaque utilisateur.

Pour finir, la dernière condition permet de vérifier deux critères: 
- Si l'utilisateur possède un shell interactif (grâce à la variable *situation* définie)
- Si l'utilisateur possède un UID inférieur à 1000, signe dans la grande majorité des distributions Linux d'un utilisateur système.

Pour finir on retrouve plusieurs conditions d'affichage pour l'utilisateur du script, dépendant des différents compteurs établis. Il est important de noter que *$nb_systeme_shell_risque -gt 1* prend donc en compte "root", qui possède habituellement un shell interactif, il ne serait donc pas pertinent d'afficher un message d'erreur si on avait *$nb_systeme_shell_risque -gt 0*.

```bash
userid=$(awk -F':' '{ print $3, $4, $7}' /etc/passwd)

nb_uid_root=0
nb_shell_interactif=0
nb_gid_root=0
nb_systeme_shell_risque=0

while read -r var1 var2 var3
do
if [[ "$var1" -eq 0 ]]; then
nb_uid_root=$((nb_uid_root + 1))
fi
if [[ "$var3" == "/bin/bash" || "$var3" == "/bin/sh" || "$var3" == "/bin/csh" || "$var3" == "/bin/tcsh" || "$var3" == "/bin/dash" ]]; then
situation="pas ok"
nb_shell_interactif=$((nb_shell_interactif + 1))
else
situation="ok"
fi
if [[ "$var2" -eq 0 ]]; then
nb_gid_root=$((nb_gid_root + 1))
fi
if [[ $situation == "pas ok" && $var1 -lt 1000 ]]; then
nb_systeme_shell_risque=$((nb_systeme_shell_risque + 1))
fi
done <<< "$userid"

if [[ $nb_uid_root -eq 1 ]]; then
echo "[OK] Un seul utilisateur possède les droits root (UID 0)."
else
echo "[ATTENTION] $nb_uid_root utilisateurs possèdent les droits root (UID 0) — vérification manuelle requise."
fi

if [[ $nb_shell_interactif -ge 2 ]]; then
echo "[ATTENTION] $nb_shell_interactif utilisateurs possèdent un shell interactif — vérification manuelle requise."
else
echo "[OK] Aucun utilisateur superflu ne possède de shell interactif."
fi

if [[ $nb_gid_root -eq 1 ]]; then
echo "[OK] Un seul utilisateur appartient au groupe root (GID 0)."
else
echo "[ATTENTION] $nb_gid_root utilisateurs appartiennent au groupe root (GID 0) — vérification manuelle requise."
fi

if [[ $nb_systeme_shell_risque -gt 1 ]]; then
echo "[ATTENTION] $nb_systeme_shell_risque compte(s) système (UID < 1000) possède(nt) un shell interactif — vérification manuelle requise."
else
echo "[OK] Aucun compte système (UID < 1000) ne possède de shell interactif."
fi 
```

**Limites connues**

2. Alpine Linux n'a pas été vérifié dans nos recherches
On avait explicitement noté que la valeur pour Alpine restait "non documentée" dans les sources qu'on avait consultées — donc pour ce gestionnaire précis (apk), on ne peut pas confirmer avec certitude que 1000 est la bonne limite.

3. D'anciennes versions de certaines distributions utilisaient 500
Rappelle-toi qu'on avait noté que Red Hat utilisait historiquement 500 comme limite, avant d'aligner sa valeur par défaut sur 1000 dans ses versions plus récentes — donc un système Red Hat/CentOS ancien pourrait toujours utiliser cette ancienne convention.
