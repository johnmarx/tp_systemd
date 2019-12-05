# TP1 - systemd

- [TP1 - systemd](#tp1---systemd)
- [Intro](#intro)
  - [Objectifs du TP](#objectifs-du-tp)
  - [Prerequisites](#prerequisites)
- [0. Préparation de la machine](#0-pr%c3%a9paration-de-la-machine)
- [I. `systemd`-basics](#i-systemd-basics)
  - [1. First steps](#1-first-steps)
  - [2. Gestion du temps](#2-gestion-du-temps)
  - [3. Gestion de noms](#3-gestion-de-noms)
  - [4. Gestion du réseau (et résolution de noms)](#4-gestion-du-r%c3%a9seau-et-r%c3%a9solution-de-noms)
    - [NetworkManager](#networkmanager)
    - [`systemd-networkd`](#systemd-networkd)
    - [`systemd-resolved`](#systemd-resolved)
  - [5. Gestion de sessions `logind`](#5-gestion-de-sessions-logind)
  - [6. Gestion d'unité basique (services)](#6-gestion-dunit%c3%a9-basique-services)
- [II. Boot et Logs](#ii-boot-et-logs)
- [III. Mécanismes manipulés par systemd](#iii-m%c3%a9canismes-manipul%c3%a9s-par-systemd)
  - [1. cgroups](#1-cgroups)
  - [2. D-Bus](#2-D-Bus)
  - [3. Restriction et isolation](#3-restriction-et-isolation)
- [IV. systemd units in-depth](#iv-systemd-units-in-depth)
  - [1. Exploration de services existants](#1-exploration-de-services-existants)
  - [2. Création de service simple](#2-cr%c3%a9ation-de-service-simple)
  - [3. Sandboxing (heavy security)](#3-sandboxing-heavy-security)
  - [4. Event-based activation](#4-event-based-activation)
    - [A. Activation via socket UNIX](#a-activation-via-socket-unix)
    - [B. Activation automatique d'un point de montage](#b-activation-automatique-dun-point-de-montage)
    - [C. Timer `systemd`](#c-timer-systemd)
  - [5. Ordonner et grouper des services](#5-ordonner-et-grouper-des-services)
- [Conclusion](#conclusion)

# Intro

## Objectifs du TP

Le but de ce TP est d'apprécier un peu plus précisément le rôle de `systemd` au sein d'un système GNU/Linux ainsi que de certaines fonctionnalités qu'il apporte pour la gestion de la machine.  

Rien est à connaître par coeur dans ce TP, mais utiliser *systemd* permet de mieux cerner le rôle d'un OS, et d'explorer des fonctionnalités avancées de gestion de services. Car c'est ça qu'on veut nan, fournir du service ?  

Il est donc question ici d'un approfondissement technique pur, sur **une techno qui est au coeur de tous les systèmes GNU/Linux les plus utilisés aujourd'hui** (Debian, RedHat, Arch, autres). Comprendre comment fonctionne l'immense API fournie par systemd, permettant de manipuler totalement un OS.

C'est utile en administration système simple, et pour comprendre comment fonctionne ce que vous installez.  
C'est aussi utile pour prendre du recul et de la conscience sur ce qu'est un OS ; et c'est aussi un outil très utilisé dans le monde du Cloud, pour provisionner des machines (cloud-init, Ignition), monitorer des machines (utilisation native des cgroups), gérer les démons liés à utilisation distribuée (conteneurisation, démon réseau) ou centraliser la gestion des logs (journald).

## Prerequisites

* manipulation basique de la ligne de commande GNU/Linux
* connaissances élémentaires en système, réseau et sécurité
* outil de virtualisation fonctionnel
  * peu m'importe la techno du moment que vous savez l'utiliser
  * `.iso` de Fedora Server 31

# 0. Préparation de la machine

* installer Fedora Server 31 
* créer un utilisateur admin (*sudoers*) et avoir une session SSH fonctionnelle
* installer Docker (en suivant la documentation officielle)
* désactiver SELinux pour le moment
  * commande `setenforce`
  * fichier `/etc/selinux/config`

# I. `systemd`-basics

Le but de cette partie est d'aborder `systemd` tranquillement. On va aussi voir comment `systemd` a centralisé la gestion du système sur plusieurs plans, en s'articulant autour d'outils tiers : 
* gestion de services
* gestion de nom de domaines
* gestion du temps
* gestion du réseau
* gestion de la résolution de noms
* gestion des sessions

## 1. First steps

> Kernel processes are listed within brackets in the output of `ps -ef`. For example `[kthreadd]` (PID 2). 

* vérifier que la version de `systemd` est > 241
  * `systemctl --version = 243 `
* 🌞 s'assurer que `systemd` est PID1 `pidof systemd 902 1 `
* 🌞 check tous les autres processus système (**PAS les processus kernel**)
  * décrire brièvement au moins 5 autres processus système
  * ` ps aux | less `
  *  Colone User : definie quel utilisateur à lancer le process. Colone PID id du process, %cpu, %meme VSZ = Virtual Memory Size RSS= Resident Set Size. 
  * State: 
  * S Interuptible sleep
  * s Is a session leader
  * I multi-thread process
  * R Running or runnable
  * L Has a pages locked in memory `
d'autre state existe. 
## 2. Gestion du temps 

La gestion du temps est désormais gérée avec `systemd` avec la commande `timedatectl`.
* `timedatectl` sans argument fournit des informations sur la machine
* 🌞 déterminer la différence entre Local Time, Universal Time et RTC time
  * expliquer dans quels cas il peut être pertinent d'utiliser le RTC time
* timezones
  * `timedatectl list-timezones` et `timedatectl set-timezone <TZ>` pour définir de timezones
Local time heure local
Universal time; heure international 
RTC: Heure de la machine, souvent donné par une carte physique. (par exemple les pile sur les carte mère joue se rôle.) Cette heure est donc toujours disponible et la mesure est précise. Cette heure peut etre modifié et synchronisé dans le cadre d'un cluster. La précission permet de lancer des action synchronisé a la microseconde.

  * 🌞 changer de timezone pour un autre fuseau horaire européen
  * `timedatectl set-timezone Europe/Berlin
  * 
* on peut activer ou désactiver l'utilisation de la syncrhonisation NTP avec `timedatectl set-ntp 0`
  * 🌞 désactiver le service lié à la synchronisation du temps avec cette commande, et vérifier à la main qu'il a été coupé
  * `sudo tcpdump -n "broadcast or multicast" | grep NTP`

## 3. Gestion de noms

La gestion de nom est aussi gérée par `systemd` avec `hostnamectl`.  
En réalité, `hostnamectl` peut récupérer plus d'informations que simplement le nom de domaine : `hostnamectl` sans arguments pour les voir.
* changer les noms de la machine avec `hostnamectl set-hostname`
* il est possible de changer trois noms avec `--pretty`, `--static` et `--transient`
* 🌞 expliquer la différence entre les trois types de noms. Lequel est à utiliser pour des machines de prod ?
* --pretty c'est juste pour faire jolie
* --static fichier /etc/hostname, utilisable pour définir la machine dans un groupe par exemple 
* --transient Nom liée au kernel, utile pour conserver l'identité, mais non utilisable "directement"
* on peut aussi modifier certains autre noms comme le type de déploiement : `hostnamectl set-deployment <NAME>`

> Cette commande est très utile pour inventorier un parc de machines, toutes les informations système élémentaires, pouvant aider à sa classification sont présentes.

## 4. Gestion du réseau (et résolution de noms)

Pour gérer la stack réseau, deux outils sont livrés avec `systemd` :
* `NetworkManager`
  * souvent activé par défaut
  * réagit dynamiquement aux changements du réseau (mise à jour de `/etc/resolv.conf` en fonction des réseaux connectés par exemple)
  * idéal pour un déploiement desktop
  * expose une API D-Bus
* `systemd-networkd`
  * permet une grande flexibilité de configuration
    * configuration de plusieurs interfaces simultanément (wildcards)
    * fonctionnalités avancées
  * utilise une syntaxe standard `systemd`
  * complètement intégré à `systemd` (gestion, logs, etc)
  * idéal en déploiement cloud

### NetworkManager

NetworkManager est l'utilitaire réseau souvent démarré par défaut sur tous les OS GNU/Linux équipés de `systemd`. Il est utilisé pour configurer au cas par cas les interfaces réseaux d'une machine.
* il pilote les fichiers existants et introduit des fonctionnalités supplémentaires
  * il conserver et pilote le fichier `/etc/resolv.conf` par exemple
* il existe des outils pour interagir avec les interfaces qu'il gère
  * similaire à la suite `iproute2` (`ip a`, `ip route show`, `ip neigh show`, `ip net add`, etc)
  * comme l'outil en ligne de commande `nmcli`

Utilisation basique en ligne de commande :
* lister les interfaces et des informations liées
  * `nmcli con show`
  * `nmcli con show <INTERFACE>`
* modifier la configuration réseau d'une interface
  * éditer le fichier de configuratation d'une interface `/etc/sysconfig/network-scripts/ifcfg-<INTERFACE_NAME>`
  * recharger NetworkManager : `sudo systemctl reload NetworkManager` (relire les fichiers de configuration)
  * redémarrer l'interface `sudo nmcli con up <INTERFACE_NAME>`
* 🌞 afficher les informations DHCP récupérées par NetworkManager (sur une interface en DHCP)

Sinon une bonne interface curses des familles : avec la commande `nmtui`

> Avec le gestionnaire de paquet `dnf`, vous pouvez utiliser `dnf provides <COMMANDE>` pour trouver le nom du paquet à installer pour avoir une commande donnée

### `systemd-networkd`

Il est aussi possible que la configuration des interfaces réseau et de la résolution de noms soit enitèrement gérée par `systemd`, à l'aide du démon `systemd-networkd`.  

Dans le cas d'utilisation de `systemd-networkd`, il est préférable de désactiver NetworkManager, afin d'éviter les conflits et l'ajout de complexité superflue :
* 🌞 stopper et désactiver le démarrage de `NetworkManager`
-   `sudo systemctl stop NetworkManager`
-   `sudo systemctl disable NetworkManager`
* 🌞 démarrer et activer le démarrage de `systemd-networkd`
* * -   `sudo systemctl start systemd-networkd`
-   `sudo systemctl enable systemd-networkd`

Il est alors possible de configurer des interfaces réseau dans `/etc/systemd/network` avec des fichiers `.network`.  
C'est le rôle du démon `systemd-networkd` que de surveiller ces fichiers et réagir aux changement d'état des cartes réseaux (comme un redémarrage).

La structure des fichiers `/etc/systemd/network/*.network` (le nom des fichiers est arbitraire) est la suivante : 
```
[Match]
Key=value

[Network]
Key=Value
```

La section `[Match]` permet de cibler une ou plusieurs interfaces (regex shells, ou liste avec des espaces) selon plusieurs critères comme le nom, l'ID udev, l'adresse MAC, etc. 

Par exemple, pour configurer une interface avec une IP statique : 
```
[Match]
Key=enp0s8

[Network]
Address=192.168.1.110/24
DNS=1.1.1.1
```

> Oh un fichier de configuration d'interface qui fonctionnent sous tous les OS GNU/Linux (qui sont équipés de `systemd`) 🔥🔥🔥

La [doc officielle détaille](https://www.freedesktop.org/software/systemd/man/systemd.network.html) (**faites-y un tour**) l'ensemble des clauses possibles et peut amener à réaliser des configurations extrêmement fines et poussées : 
* configuration de zone firewall, de serveurs DNS, de serveurs NTP, etc. par interface
* utilisation des protocoles liés aux VLAN, VXLAN, IPVLAN, VRF, etc
* configuration de fonctionnalités réseau comme le bonding/teaming, bridge, MacVTap etc
* création de tunnel ou interface virtuelle
* détermination manuelle de voisins L2 (table ARP)
* etc

* 🌞 éditer la configuration d'une carte réseau de la VM avec un fichier `.network`

> A noter qu'un outil comme `nmtui` verra les configurations réalisées avec NetworkManager et avec `systemd-networkd`, car ils pilotent tous les deux la même API. 

### `systemd-resolved`

L'activation de `systemd-resolved` permet une résolution des noms de domaines avec un serveur DNS local sandboxé (sur certaines distributions c'est fait par défaut). Certains bénéfices de l'utilisation de `systemd-resolved` sont :
* configuration de DNS par interface
  * aucune requête sur des DNS potentiellement injoignables 
    * = pas de leak d'infos
    * = optimisation du temps de lookup
* résolution robuste avec un serveur DNS local sandboxé
* support natif de fonctionnalités comme DNSSEC, DNS over TLS, caching DNS

* 🌞 activer la résolution de noms par `systemd-resolved` en démarrant le service (maintenant et au boot)
  * vérifier que le service est lancé
  * -   `sudo systemctl start systemd-resolved`
-   `sudo systemctl enable systemd-resolved`
-   `sudo systemctl status systemd-resolved`
* 🌞 vérifier qu'un serveur DNS tourne localement et écoute sur un port de l'interfce localhost (avec `ss` par exemple)
* `ss -4 state listening`
  * effectuer une commande de résolution de nom en utilisant explicitement le serveur DNS mis en place par `systemd-resolved` (avec `dig`)
  * effectuer une requête DNS avec `systemd-resolve`
* on peut utiliser `resolvectl` pour avoir des infos sur le serveur local

> `systemd-resolved` apporte beaucoup de fonctionnalités comme du caching, le support de DNSSEC ou encore de DNS over TLS. Une fois qu'il est en place, tout passe par lui, le fichier `/etc/resolv.conf est donc obsolète`

* 🌞 Afin d'activer de façon permanente ce serveur DNS, la bonne pratique est de remplacer `/etc/resolv.conf` par un lien symbolique pointant vers `/run/systemd/resolve/stub-resolv.conf`
* * -   `ln –s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`
* 🌞 Modifier la configuration de `systemd-resolved`

  * elle est dans `/etc/systemd/resolved.conf`
  * ajouter les serveurs de votre choix
  * vérifier la modification avec `resolvectl`
* 🌞 mise en place de DNS over TLS
  * renseignez-vous sur les avantages de DNS over TLS
  * effectuer une configuration globale (dans `/etc/systemd/resolved.conf`)
    * compléter la clause `DNS` pour ajouter un serveur qui supporte le DNS over TLS (on peut en trouver des listes sur internet)
    * utiliser la clause `DNSOverTLS` pour activer la fonctionnalité
      * valeur `opportunistic` pour tester les résolutions à travers TLS, et fallback sur une résolution DNS classique en cas d'erreur
      * valeur `yes` pour forcer les résolutions à travers TLS
  * prouver avec `tcpdump` que les résolutions sont bien à travers TLS (les serveurs DNS qui supportent le DNS over TLS écoutent sur le port 853)
* 🌞 activer l'utilisation de DNSSEC

## 5. Gestion de sessions `logind`

`logind` est le nom du démon qui gère les sessions.  

On peut le manipuler avec `loginctl`. Rien de fou ici (si on omet les détails techniques de gestion de session), je vous laisse explorer un peu la ligne de commande.

## 6. Gestion d'unité basique (services)

La principale entité que `systemd` gère sont les *unités systemd* ou *systemd units*.  
Les unités sont définies dans des fichiers texte et permettent de manipuler différents éléments du système : 
* services
* point de montage
* carte réseau
* autres. (later)

Les paths où sont définis les unités sont les suivants, du moins prioritaire, au plus prioritaires (non-exhaustif) :
* `/usr/lib/systemd/system` : utilisé par la plupart des installations par défaut
  * faites un tour et regardez un peu ce qui se balade là-bas
* `/run/systemd/system` : utilisé au runtime d'un service
* `/etc/systemd/system` : dossier dédié à la modification par l'administrateur

> Donc si on veut ajouter une nouvelle unité, c'est dans `/etc/systemd/system`.

---

Manipulation d'unité `systemd`

* lister les unités `systemd` actives de la machine
  * `systemctl list-units`
  * ou seulement `systemctl`
  * on peut ajouter des options pour filtrer par type
  * pour ne lister que les services : `systemctl list-units -t service`

> Beaucoup de commandes `systemd` qui écrivent des choses en sortie standard sont automatiquement munie d'un pager (pipé dans `less`). On peut ajouter l'option `--no-pager` pour se débarasser de ce comportement

* pour obtenir plus de détails sur une unitée donnée
  * `systemctl is-active <UNIT>`
    * détermine si l'unité est actuellement en cours de fonctionnement
  * `systemctl is-enabled <UNIT>`
    * détermine si l'unité est liée à un target (généralement, on s'en sert pour activer des unités au démarrage)
  * `systemctl status <UNIT>`
    * affiche l'état complet d'une unité donnée
    * comme le path où elle est définie, si elle a été enable, tous les processus liés, etc.
  * `systemctl cat <UNIT>`
    * affiche les fichiers qui définissent l'unité

Les outils de l'écosystème GNU/Linux ont été modifiés pour être utilisés avec `systemd`
* par exemple `ps`
* on peut utiliser `ps` pour trouver l'unité associée à un processus donné : `ps -e -o pid,cmd,unit`
* on peut aussi effectuer un `systemctl status <PID>` qui nous retournera l'état de l'unité qui est responsable de ce PID
  * les logs sont fournis par *journald*, les stats système par les mécanismes de *cgroups* (on y revient plus tard)
* 🌞 trouver l'unité associée au processus `chronyd`
* `└─system.slice` `├─httpd.service` `├─chronyd.service` `│ └─613 /usr/sbin/chronyd -u chrony`

# II. Boot et Logs

`systemd` est le PID 1 sur les distributions GNU/Linux qui l'utilisent. C'est à dire qu'il est le premier processus lancé, et qu'il s'occupe de lancer les processus système.  
Il est connu pour accélérer considérablement le boot des machines GNU/Linux, à l'aide de deux principes (déjà existants auparavant) :
* parallélisation des tâches
* event-based
  * c'est à dire que certaines tâches ne s'effectueront que si un évènement est réalisé
  * par exemple, le démon bluetooth ne s'activera que s'il y a une requête Bluetooth (sur le socket dédié)

Cette place privilégiée lui permet d'être présent dans les premiers instants du boot de l'OS et ainsi de récupérer des métriques intéressantes. 

* 🌞 générer un graphe de la séquence de boot
  * `systemd-analyze plot > graphe.svg`
  * très utile pour du débug
  * déterminer le temps qu'a mis `sshd.service` à démarrer
* on peut aussi utiliser `systemd-analyse blame` en ligne de commande

---

`systemd` embarque un démon de journalisation `journald`. Il centralise tous les logs de la machines de la façon la plus exhaustive possible. Il rend notamment possible la génération du graphe avec `systemd-analyze plot` en récupérant les logs au chargement du kernel.  

Les logs sont stockés dans un format binaire inexploitable à la main. Plutôt relou. L'avantage est que les logs deviennent exploitables de façon très flexibles avec des commandes dédiées : 
```
$ sudo journalctl -xe

# Logs kernel (similaire à dmesg)
$ sudo journalctl -k 

# Logs d'une unité spécifique ou process spécifique
$ sudo journalctl -xe -u sshd.service
$ sudo journalctl -xe _PID=1
# Pour plus de filtres
$ man systemd.journal-fields 

# Logs en temps réel
$ journalctl -xe -f

# Logs avec des ordres dans le temps
$ sudo journalctl -xe -u sshd.service --since yesterday
$ sudo journalctl -xe -u sshd.service  --since "2019-11-28 01:00:00"

# JSON output
$ sudo journalctl -xe -u sshd.service  --since "2019-11-28 01:00:00" --output=json
```

L'avantage des logs binaires, c'est qu'on peut les plier dans tous les sens. Utiles pour query à la main, mais encore plus pour exporter dans des outils de centralisation ou analyse de logs (environnement Cloud avec Kubernetes, graphes d'erreur avec Grafana ou analyse avec un SOC).

# III. Mécanismes manipulés par systemd

## 1. cgroups

Lire [le passage du cours dédié aux cgroups](../cours/os_kernel_systemd.md#cgroups) est préférable.

Les cgroups sont désormais profondément intégrés aux systèmes GNU/Linux et `systemd` a été construit avec les cgroups en tête.

* un *scope* systemd est un cgroup qui contient des processus non-gérés par `systemd`
* un *slice* systemd est un cgroup qui contient des processus directement gérés par `systemd`

* `systemd-cgls`
  * affiche une structure arborescente des cgroups
* `systemd-cgtop`
  * affiche un affichage comme `top` avec l'utilisation des ressources en temps réel, par cgroups
* `ps -e -o pid,cmd,cgroup`
  * ajoute l'affichage des cgroups à `ps`
* `cat /proc/<PID>/cgroup`

Prenez le temps de vous balader un peu avec ces commandes, et d'explorer `/sys/fs/cgroup` pour voir les restrictions mises en place.

* 🌞 identifier le cgroup utilisé par votre session SSH
  * identifier la RAM maximale à votre disposition (dans `/sys/fs/cgroup`)
  * sshd.service-->1060 /usr/bin/sshd 
* 🌞 modifier la RAM dédiée à votre session utilisateur
  * `systemctl set-property <SLICE_NAME> MemoryMax=512M`
  * vérifier le changement
    *  `/sys/fs/cgroup`
* la commande `systemctl set-property` génère des fichiers dans `/etc/systemd/system.control/`
  * 🌞 vérifier la création du fichier
  * on peut supprimer ces fichiers pour annuler les changements
 
> Pour voir les propriétés natives de restrictions possibles (en plus de *MemoryMax* : `$ man systemd.resource-control`)

## 2. D-Bus

D-Bus a lui aussi une [section dédiée dans le cours](../cours/os_kernel_systemd.md#D-Bus).

> Il existe des outils sympas comme `d-feet` pour explorer les bus de D-Bus avec une GUI.  

> Vous pouvez aussi utiliser **Wireshark** pour explorer les échanges D-Bus (dump avec `tcpdump` puis analyse avec Wireshark, ou analyse directe avec Wireshark pour ceux qui ont un GNU/Linux en desktop).

Les commandes pour manipuler D-Bus sont `dbus-monitor` (écouter les évènements) et `dbus-send` (envoyer des signaux ou appeler des méthodes).  
`systemd` livre aussi un binaire très puissant pour inspecter et interagir avec les différents bus de l'OS : `busctl`

* `busctl` permet d'obtenir des informations sur les différents bus de l'OS, y compris le bus `system` et `user` de D-Bus
* `dbus-monitor --system` pour observer le bus système
  * vous pouvez facilement générer des messages en faisant des actions avec `systemctl` par exemple
  * vous pouvez tester sur votre hôte d'augmenter/baisser la luminosité de l'écran, ou brancher une clé USB

**Pour un exemple plus détaillé avec `busctl`, voir la [section dédiée dans le cours](../cours/os_kernel_systemd.md#D-Bus).**

---

Exemple : est-ce que NetworkManager est activé ?
```
# Avec dbus-send
sudo dbus-send --system --print-reply \
        --dest=org.freedesktop.NetworkManager \
        /org/freedesktop/NetworkManager \
        org.freedesktop.DBus.Properties.Get \
        string:"org.freedesktop.NetworkManager"  \
        string:"NetworkingEnabled"

# Avec busctl
sudo busctl get-property \
        org.freedesktop.NetworkManager \
        /org/freedesktop/NetworkManager \
        org.freedesktop.NetworkManager \
        NetworkingEnabled
```

---

En utilisant `dbus-monitor` ou `busctl monitor`
* 🌞 observer, identifier, et expliquer complètement un évènement choisi

En utilisant D-Feet, `busctl` ou `dbus-send`
* 🌞 UPDATE : récupérer la valeur d'une propriété d'un objet D-Bus
* 🌞 envoyer un signal OU appeler une méthode, à l'aide des commandes ci-dessus

Pour le fun...
* essayer de générer des évènements autrement pour voir un peu ce qu'il se passe
  * en rechargeant la conf réseau
  * en stoppant/redémarrant des services `systemd`
  * etc.

> C'est plus facile d'utiliser D-Bus de façon programmatique (Python ou C ont par exemple de très bonnes libs) que depuis la ligne de commande.

## 3. Restriction et isolation

On peut lancer un processus dans un service temporaire avec `systemd-run` et lui appliquer des restrictions cgroups. Utile pour sandboxer un programme, et éventuellement tracer des informations, ou logger ses activités dans le journal système. 

---

* lancer un processus sandboxé, et tracé, avec `systemd-run`
  * un shell par exemple, pour faire des tests réseaux `sudo systemd-run --wait -t /bin/bash`
  * un service est créé : `systemctl status <NAME>`
  * un cgroup est créé : `systemd-cgls` pour le repérer, puis `/sys/fs/cgroup` pour voir les restrictions appliquées
  * 🌞 identifier le cgroup utilisé
* il est aussi possible d'ajouter directement des restrictions cgroups, ou isolations namespaces
  * 🌞 lancer une nouvelle commande : ajouter des restrictions cgroups
    * `sudo systemd-run -p MemoryMax=512M <PROCESS>`
    * par exemple : `sudo systemd-run -p MemoryMax=512M --wait -t /bin/bash`
  * 🌞 lancer une nouvelle commande : ajouter un traçage réseau
    * `sudo systemd-run -p IPAccounting=true <PROCESS>`
    * par exemple : `sudo systemd-run -p IPAccounting=true --wait -t /bin/bash`
      * effectuer un ping
      * quitter le shell
      * observer les données récoltées
  * 🌞 lancer une nouvelle commande : ajouter des restrictions réseau
    * `-p IPAddressAllow=192.168.56.0/24` + `-p IPAddressDeny=any`
    * prouver que les restrictions réseau sont effectives

---

Il est préférable de lire [la partie du cours dédiée aux namespaces](./../cours/os_kernel_systemd.md#namespaces).

Lancer un processus complètement sandboxé (conteneur ?) avec `systemd-nspawn` :
* `sudo systemd-nspawn --ephemeral --private-network -D / bash`
  * vérifier que `--private-network` a fonctionné : `ip a`
  * 🌞 expliquer cette ligne de commande
  * 🌞 prouver qu'un namespace réseau différent est utilisé
    * pour voir les namespaces utilisés par un processus donné, on peut aller voir dans `/proc`
    * `ls -al /proc/<PID>/ns` : montre les liens vers les namespaces utilisés (identifiés par des nombres)
    * si le nombre vu ici est différent du nombre vu pour un autre processus alors ils sont dans des namespaces différents
  * 🌞 ajouter au moins une option pour isoler encore un peu plus le processus lancé (un autre namespace par exemple)

# IV. systemd units in-depth

Les unités sont au coeur du fonctionnement de `systemd`. Les services ne sont qu'un type d'unité `systemd`, il en existe d'autres. Dans cette partie on va manipuler :
* service
* timer
* socket
* path
* automount

> Pour une liste complète : `systemctl -t help`

Il est possible de dumper l'état de tous les services lancés par `systemd` avec `$ systemd-analyze dump`

## 1. Exploration de services existants

```
# Liste des service existants
$ systemctl list-units --all
$ systemctl --all

# Filtrer avec le type
$ sudo systemctl -t service --all

# Obtenir des infos sur une unité
$ sudo systemctl status <UNIT>

# Lire le fichier lié à l'unité
$ sudo systemctl cat <UNIT>

# Ajouter de la configuration dans un fichier drop-in
# Idéal pour modifier une unité du système mais conserver nos changements après update
$ sudo systemctl edit <UNIT>

# Modifier complètement l'unité
$ sudo systemctl edit --full <UNIT>
```

* faites des tests et observez la structure des unités, toujours identiques :
```
[Unit]
Description=XXX
# Clauses de dépendances, ordres de démarrage

[<TYPE>]
```

* pour un service, on a souvent
```
[Unit]
Description=XXX
# Clauses de dépendances, ordres de démarrage

[Service]
Type=(forking|simple|exec)
ExecStart=<CMD>
ExecStop=<CMD>
```

* `Type`
  * `exec` : l'unité s'attend à lancer une commande puis quitter
  * `simple` : l'unité s'attend à être un démon, qui doit rester actif
  * `forking` : l'unité s'attend à être un démon qui crée de nouveaux processus (comme un serveur Apache par exemple)
* `ExecStart` : commande qui permet de lancer le service
* `ExecStop` : commande qui permet de stopper le service
  * dans les cas les plus simples, `systemd` gère l'extionction des processus lui-même grâece au monitoring cgroup (il détermine le numéro du père des processus)

* 🌞 observer l'unité `auditd.service`
  * trouver le path où est défini le fichier `auditd.service`
  * expliquer le principe de la clause `ExecStartPost`
  * expliquer les 4 "Security Settings" dans `auditd.service`

## 2. Création de service simple

🌞 Créer un fichier dans `/etc/systemd/system` qui comporte le suffixe `.service` :
* doit posséder une description
* doit lancer un serveur web
* doit ouvrir un port firewall quand il est lancé, et le fermer une fois que le service est stoppé
* doit être limité en RAM

> `/etc/systemd/system` est le path dédié aux modifications ou ajouts d'unités `systemd` par l'administrateur

Beuacoup beaucoup d'autres options sont disponibles pour un service, comme la définition de variables d'environnement par service, ou l'utilisation d'un utilisateur spécifique. Inspirez-vous des fichiers de service existants. 

* 🌞 UPDATE : faites en sorte que le service se lance au démarrage
  * `systemctl enable` ?...
  * expliquer ce que fait cette commande concrètement
    * la section `[Install]` des unités est nécessaire pour que `enable` fonctionne
  * vous pouvez connaître le *target* (ensemble d'unités) utilisé au boot avec `systemctl get-default`

## 3. Sandboxing (heavy security)

`systemd` propose des fonctionnalités très avancées, fines, granulaires et bas niveau pour sécuriser une unité. Nous allons jouer avec un `service`.  

Utiliser la commande `systemd-analyze security <SERVICE>` sur votre service précémment créé. Cette commande permet d'analyser l'unité et d'afficher un score de sécurité, en fonction d'un certains nombres de critères.  
Les critères correspondent à des clauses que l'on peut rajouter dans le fichier `.service`.  

La plupart se base sur les mécanismes suivants :
* cgroups
  * on a déjà vu : regroupement de processus, et restriction d'accès au système
  * on peut explorer les cgroups gérés par `systemd` avec `systemd-cgls` et `systemd-cgtop`
  * **le pseudo-filesystem `/sys/fs/cgroup` nous permet d'accéder à toutes les informations concernant les cgroups**
* namespaces
  * déjà vu plus haut aussi 
  * **on peut regarder dans quels namespaces se trouve un processus avec un `ls -al /proc/<PID>/ns`**
* capabilities
  * droits particuliers attribués à des processus
  * c'est la décomposition en plusieurs pouvoirs de tous les pouvoirs de `root`
  * par exemple
    * `CAP_NET_BIND_SERVICE` : écouter sur un port en dessous de 1024 
    * `CAP_CHOWN` : permet de changer arbitrairement les propriétaires des fichiers
    * `CAP_DAC_OVERRIDE` : bypass les checks de permissions classiques sur les fichiers ("DAC" pour "Discretionary access acontrol", dans notre cas c'est les droits rwx.)
* seccomp
  * permet de filtrer les appels système qu'émet un processus
  * les appels système sont des fonctions du kernel, qui permettent d'effectuer de lui demander d'effectuer des actions
  * il en existe moins de 200 sur la plupart des OS, qui permettent de faire tout ce qu'un OS sait faire (lire/écrire un fichier, ouvrir un port, lancer des processus, etc.)

---

Essayez d'obtenir le meilleur score avec `systemd-analyze security <SERVICE>`, en comprenant ce que vous faites.
* 🌞 Expliquer au moins 5 cinq clauses de sécurité ajoutées
  * Mettez en place au moins une mesure liée aux cgroups
    * vous pouvez vérifier que c'est le cas en regardant dans `/sys/fs/cgroup`
  * Mettez en place au moins une mesure liée aux namespaces
    * vous pouvez vérifier que c'est le cas en regardant dans `/proc/<PID>/ns`

> L'utilisation de telles mesures de sécurité fait penser au fonctionnement des conteneurs. C'est effectivement similaire mais beaucoup plus fin et granulaire. Aussi, l'approche reste orientée sur un service système, et pas un conteneur à proprement dit.

---

Cas concrets :
* éviter tout cryptolocker avec `PrivateDev` (par exemple pour les navigateurs web ou client mail)
* réduire la surface d'exposition de certains processus qui se servent de `/tmp` pour communiquer avec `PrivateTmp`
* restreint un serveur à n'effectuer aucune connexion exceptée sur un LAN donné avec `IPAddressDeny=0.0.0.0` + `IPAddressAllow=172.17.0.0/24`
* empêcher un processus de créer de nouveaux processus (empêche toute `()` bomb notamment) en filtrant les appels système, limitant le nombre de `tasks` (terme générique pour threads et processus confondus)
* and the list goes on

En vérité, bon nombre de ces fonctionnalités pourraient être activées par défaut sur la plupart des unités que l'on manipule. Par exemple, des mesures comme `PrivateTmp`, `ProtectHome`, `ProtectSystem` (entre autres) pourraient être appliquées quasi-systématiquement. Cela a déjà été discuté dans plusieurs talks autour de `systemd`, les deux principales raisons :
* ces features sont ajoutées petit à petit depuis la créations de `systemd`
* les activer petit à petit casseraient certaines applications

## 4. Event-based activation 

`systemd` est aussi connu pour apporter une gestion des services, et des unités en général, basée sur les évènements.

### A. Activation via socket UNIX

Faire en sorte que le serveur web ne se lance que quand il y a des requêtes sur le port qui lui est dédié.

Principe de mise en place :
```
# Création d'un fichier .service
$ sudo vim /etc/systemd/system/bap.service

# Création d'un fichier .socket correspondant
$ sudo vim /etc/systemd/system/bap.socket

# Syntaxe du fichier
$ cat /etc/systemd/system/bap.socket
[Unit]
Description=Bap socket

[Socket]
ListenStream=0.0.0.0:80

ExecStartPre=/usr/bin/firewall-cmd --add-port 80/tcp
ExecStartPre=/usr/bin/firewall-cmd --add-port 80/tcp --permanent
ExecStopPost=/usr/bin/firewall-cmd --remove-port 80/tcp
ExecStopPost=/usr/bin/firewall-cmd --remove-port 80/tcp --permanent

[Install]
WantedBy=sockets.target
```

🌞 Faire en sorte que Docker démarre tout seul s'il est sollicité
* avoir installer `docker`
* vérifier que le service `docker` est éteint (`systemctl is-active docker`)
* création d'un fichier `/etc/systemd/system/docker.socket`
* faire en sorte que le socket écoute sur le socket UNIX utilisé par docker
* activer le socket `systemd` et prouver que le démon `docker` se lance uniquement lorsque le socket est sollicité

### B. Activation automatique d'un point de montage

Il est aussi possible d'activer et désactiver automatiquement des points de montage lorsqu'ils sont sollicités. Pour cela, on va utiliser `automount`.

Mettre en place un automount d'un point de montage NFS
* mettre en place une deuxième VM qui expose un dossier en NFS
* vérifier le bon fonctionnement du point de montage sur la machine client
  * monter le partage NFS sur `/data/nfs`
* sur le client, créer un fichier `.automount` dans `/etc/systemd/system`

```
[Unit]
Description=Mount NFS directory

Requires=network-online.target
After=network-online.service

[Automount]
Where=/data/backup

[Install]
WantedBy=multi-user.target
```

* et un fichier `.mount`
```
[Unit]
Description=Remote cifs backup mount script
Requires=network-online.target
After=network-online.service

[Mount]
What=NFS_SERVER_IP:PATH
Where=/data/backup
Options=noauto,x-systemd.automount
Type=nfs
```

🌞 **Prouver le bon fonctionnement de l'automount
***
`Le timer permet de planifier des tache tout comme cron.
systemctl list-timer liste les timer en cours
                 status $timer_name info
                 cat $timer_name définie la structure du timer

### C. Timer `systemd`

Les timers `systemd` ont un fonctionnement similaire au démon cron : ils permettent de planifier des tâches. Cela dit, les tâches lancées seront des services systemd, on profite alors du logging et monitoring natif du système.

* `systemctl list-timers` permet de lister les timers en cours
* `systemctl status <TIMER NAME` permet d'obtenir plus d'infos
* `systemctl cat <TIMER_NAME` pour voir la construction d'un timer existant

Mise en place :
* 🌞 Créer un script simpliste qui archive un dossier sur le path créé dans le 2. : `/data/nfs`
* 🌞 Créer un fichier `.service` qui lance la backup
* 🌞 Créer un fichier `.timer` qui programme la backup tous les jours à heure fixe
  * en utilisant la clause `OnCalendar` (voir [la doc officielle](https://www.freedesktop.org/software/systemd/man/systemd.time.html) pour les valeurs possibles)

## 5. Ordonner et grouper des services

La gestion des unités `systemd` est organisée de façon séquentielle et hiérarchique. C'est à dire, par exemple, qu'on peut ordonner à une unité de ne démarrer qu'après qu'une autre unité ait été activée.

Pour afficher les dépendances d'une unité donnée, on peut utiliser `systemctl list-dependencies`.
* `systemctl list-dependencies systemd-udevd.service`
  * le démon `udevd` étant une brique centrale de l'OS, elle est démarrée très tôt et n'a que peu de dépendances

On distinguera :
* d'une part, le fait de grouper des unités, avec des `target` `systemd`
  * `systemctl -t target` pour lister les targets actifs
  * `systemctl -t target --all` pour aussi obtenir les targets inactifs
  * l'appartenance à un `target` est définie dans le fichier de l'unité, dans la section [Install]
  * permet de grouper logiquement des unités
   * un `.target` qui regroupe nos applications web et une base de données par exemple
   * `systemctl isolate shutdown.target` a le même effet que `init 0`
* et d'autre part gérer les dépendances entre les unités
  * cela se fait avec des clauses comme `Require`, `Before`, `After` etc
  * ces clauses sont dans les fichiers de définitions d'unités `systemd`, dans la section `[Unit]`

> Les `target` remplacent l'utilisation des runlevels. On peut, à la place d'`init`, utiliser la commande `systemctl isolate`.`systemctl isolate` permet de lire le contenu d'un target, de lancer les applications correspondantes, et de mettre fin à toutes les autres qui ne sont pas concernées par le `target`.

Par exemple, l'unité `docker.service`
```
# /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target docker.socket firewalld.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd://
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

* le service `docker.service` appartient au `target` `multi-user.target`
  * l'équivalent du runlevel 5
* le service `docker.service` doit se lancer après `firewalld.service` et `network.service`
  * mais aussi après la création du socket lié à la communication avec le démon docker `docker.socket`
  * on peut d'ailleurs visualiser la définition de ce socket avec les comandes `systemctl status docker.socket` ou `systemctl cat docker.socket`

--- 

* 🌞 créer un `target` systemd (inspirez-vous des `target` existants, et de la doc)
  * ce `target` doit démarrer deux autres unités de votre choix
  * l'une des deux unités doit absolument démarrer après l'autre
  * par exemple, un service Web qui démarre après que son socket ait été créé
  * ou une base de données qui démarre avant le service Web

# Conclusion

`systemd`...
* est un gestionnaire de services, gère le "system-space"
* abstrait les ressources de l'OS avec le concept d'unités
* améliore la vitesse de boot
  * à la manière d'OpenRC
* est très orienté *event-based*
  * il est très flexible
  * la gestion des dépendances et relations entre unités est centrale
* permet une gestion unifiée de l'OS
* fait appel à des fonctionnalités natives de l'OS et du kernel pour apporter un haut niveau de sécurité, de façon granulaire
  * cgroups, namespaces, seccomp, capabilities, bind-mount, etc.
* embarque des outils existants et de nouveaux outils afin de former un écosystème cohérent pour gérer le "system-space"
  * gestion de DNS : `systemd-resolved`
  * gestion d'interfaces réseau : `systemd-networkd`
  * gestion de temps : `systemd-timesyncd`
  * gestion de session : `systemd-logind`
  * gestion de logs : `systemd-journald`
  * gestion de devices : `udev`
  * IPC : grosse utilisation de [D-Bus](../cours/os_kernel_systemd.md#D-Bus)
  * access control : MAC mechanisms (SELinux, Smack, AppArmor)
  * CLI très riche (`systemd-*`, `systemctl`, `timedatectl`, `resolvectl`, etc.)
* un pied à l'étrier pour des environnements cloud-native

