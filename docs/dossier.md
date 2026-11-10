# Dossier technique — InfraReady

### Déploiement, automatisation et supervision d'une infrastructure Linux / Windows

**Bachelor 3 — Cybersécurité**  
**Informatique, Infrastructure et Réseau**

Alexandre BOZZI · Loan Mata

Année universitaire 2026

Dépôt GitHub : [github.com/d3vex/infraready](https://github.com/d3vex/infraready)

![Architecture du laboratoire InfraReady](images/0.%20Schema%20architecture.png)

*Schéma — Architecture du laboratoire InfraReady.*

---

## Sommaire

1. [Contexte et besoin](#1-contexte-et-besoin)
2. [Architecture et plan réseau](#2-architecture-et-plan-reseau)
3. [Installation des systèmes](#3-installation-des-systemes)
4. [Administration Windows](#4-administration-windows)
5. [Administration Linux](#5-administration-linux)
6. [Automatisation avec Ansible](#6-automatisation-avec-ansible)
7. [Supervision avec Prometheus et Grafana](#7-supervision-avec-prometheus-et-grafana)
8. [Maintenance et mises à jour](#8-maintenance-et-mises-a-jour)
9. [GitHub et intégration continue](#9-github-et-intégration-continue)
10. [Diagnostic d'incidents](#10-diagnostic-dincidents)
11. [Sécurité, limites et bilan](#11-securite-limites-et-bilan)
12. [Annexes](#12-annexes)

---

## 1. Contexte et besoin <a id="1-contexte-et-besoin"></a>

InfraReady est une entreprise fictive dont l'infrastructure repose sur des serveurs Linux et Windows. Jusqu'ici, une bonne partie de la configuration se fait à la main. Les procédures ne sont pas réellement standardisées, les mises à jour sont suivies de manière irrégulière et, quand un service tombe, il faut souvent repartir de zéro pour comprendre ce qui s'est passé.

La direction informatique veut disposer d'une plateforme de laboratoire qui serve de base reproductible : les configurations doivent pouvoir être rejouées, les opérations d'administration documentées et l'état des serveurs visible depuis un tableau de bord. C'est le cadre du projet fil rouge : concevoir, déployer, automatiser, superviser et documenter une infrastructure Linux/Windows.

Nous avons travaillé en binôme sur huit séances, avec VMware Workstation Pro comme hyperviseur. Le laboratoire reste volontairement isolé du réseau de l'établissement. Les adresses, le domaine et les mots de passe sont des valeurs de test ; aucun secret réel n'apparaît dans ce dossier ni dans le dépôt Git.

### 1.1. Ce que le projet devait couvrir

| Domaine | Attendu | Où il est traité |
|---|---|---|
| A. Installation et réseau | Trois VM, adressage fixe, DNS, connectivité | Chapitres 2 et 3 |
| B. Administration Windows | AD DS, DNS, comptes, groupes, journaux | Chapitre 4 |
| C. Administration Linux | Comptes, permissions, service web, stockage, journaux | Chapitre 5 |
| D. Automatisation | Inventaire, playbooks Linux et Windows, vérifications | Chapitre 6 |
| E. Maintenance | Mises à jour, espace disque, précautions | Chapitre 8 |
| F. Supervision | Prometheus, exporters, Grafana, alerte | Chapitre 7 |
| G. GitHub et CI | Dépôt organisé, workflow de validation | Chapitre 9 |
| H. Incident | Symptôme, tests, cause, correction, retour à la normale | Chapitre 10 |

### 1.2. Les choix structurants

Trois décisions ont conditionné le reste du projet.

D'abord, nous avons gardé **trois machines séparées** plutôt que de tout regrouper sur un seul Ubuntu. Le sujet l'autorisait en cas de ressources limitées, mais la séparation reproduit mieux une vraie architecture : un contrôleur de domaine Windows, un serveur Linux pour les services et un serveur d'administration pour Ansible et la supervision. Les ressources de l'hôte ont suffi.

Ensuite, nous avons choisi un **réseau host-only** pour le trafic interne et une **carte NAT** pour les téléchargements. Le réseau privé isole le laboratoire ; le NAT sert uniquement à installer les paquets et les mises à jour. Ce n'est pas un détail : c'est ce qui permet de tester le DNS interne sans dépendre d'un résolveur externe.

Enfin, nous avons fait le choix d'**Ansible dès que possible**, y compris pour Windows via WinRM. Certaines opérations ont d'abord été faites à la main pour comprendre ce qui se passait, puis reprises dans un playbook. C'est plus long au départ, mais c'est la seule façon d'obtenir un résultat rejouable.

### 1.3. Deux distributions Linux, deux environnements

Le laboratoire de référence est construit sur **Ubuntu Server 24.04 LTS**. En parallèle, Loan Mata a exploité un second environnement en **RHEL 9**, derrière un pare-feu **pfSense** et des VLAN. Les deux distributions ont réellement servi : RHEL 9 s'est montré plus économe en ressources qu'Ubuntu sur la machine de Loan, et il a ouvert une deuxième piste de recherche intéressante pour vérifier la portabilité des playbooks (APT/DNF, systemd, SELinux) et le comportement de la supervision sur une distribution différente.

Ce dossier distingue donc deux périmètres :

- le **laboratoire de référence** à trois VM (WIN-DC01, LINUX01, OPS01), décrit au chapitre 2 et utilisé pour les captures Ubuntu/Windows ;
- l'**environnement de validation RHEL 9** de Loan, derrière pfSense, utilisé pour les tests multi-distribution et les captures RHEL 9 des sections 6.8, 7 et 8.1.

Les deux environnements réutilisent le même plan d'adressage `192.168.50.0/24`, volontairement, pour comparer les procédures à configuration égale. Les captures RHEL 9 ne doivent donc pas être lues comme une quatrième VM du laboratoire de référence.

Les captures RHEL 9 de ce dossier sont prises avec le compte `root`. C'est un choix de commodité, pas un choix de sécurité : l'environnement RHEL 9 est un conteneur LXC, et non une VM, et l'utilisateur par défaut y est `root` ; nous avons donc utilisé ce compte pour aller plus vite pendant la configuration. Dans un cas réel, nous aurions utilisé un compte nominatif avec élévation contrôlée. Un compte `administrateur` a bien été créé et ajouté au groupe `sudo` (image 1 bis) : il permet une administration nominative, distincte de `root`. Pour un vrai moindre privilège, il faudrait encore restreindre les commandes autorisées dans `sudoers` ; les captures RHEL 9 montrent `root` uniquement pour la phase de configuration.

![Création de l'utilisateur administrateur](images/1bis.%20Creation%20utilisateur%20administrateur.png)

*Image 1 bis — Création de l'utilisateur administrateur et ajout au groupe sudo.*

---

## 2. Architecture et plan réseau <a id="2-architecture-et-plan-reseau"></a>

![Architecture du laboratoire InfraReady](images/0.%20Schema%20architecture.png)

*Schéma — Les trois VM, le réseau privé VMnet2 et la sortie NAT.*

### 2.1. Les machines

| Nom | Système | Rôle | Adresse | Ressources |
|---|---|---|---|---|
| **WIN-DC01** | Windows Server 2025 | AD DS, DNS, comptes et groupes, journaux Windows | 192.168.50.10/24 | 4 vCPU, 6 Go de RAM, 50 Go |
| **LINUX01** | Ubuntu Server 24.04 LTS | Service web nginx, comptes et permissions, stockage, node_exporter | 192.168.50.20/24 | 2 vCPU, 4 Go de RAM, 30 Go + disque de 5 Go |
| **OPS01** | Ubuntu Server 24.04 LTS | Ansible, Prometheus, Grafana | 192.168.50.30/24 | 4 vCPU, 6,8 Go de RAM, 30 Go |

Les valeurs viennent des écrans de configuration VMware. Les dashboards Grafana affichent les valeurs remontées par les exporters, qui ne correspondent pas toujours à la configuration finale des VM (par exemple 2 Gio de RAM pour LINUX01 ou 8 Gio pour WIN-DC01) : les captures n'ont pas toutes été prises au même moment, après les ajustements de ressources. Ce sont les valeurs des VM qui font foi pour l'inventaire.

Ce tableau décrit le laboratoire de référence. L'environnement RHEL 9 de validation (section 1.3) n'est pas compté dans ces trois VM : il est décrit séparément et réutilise le même plan d'adressage pour la comparaison des procédures.

### 2.2. Le réseau

Le laboratoire utilise deux réseaux VMware :

- **VMnet2** : réseau host-only, `192.168.50.0/24`, DHCP local désactivé. C'est le réseau interne du projet.
- **VMnet8** : réseau NAT, utilisé uniquement pour la sortie Internet (dépôts APT/DNF, téléchargements, mises à jour).

Dans le laboratoire de référence, chaque VM possède donc deux cartes : une sur VMnet2, une sur VMnet8. Sur Linux, la carte NAT est `ens33` et la carte privée `ens37`. Sur Windows, `Ethernet1` correspond au NAT et `Ethernet0` au réseau privé. Ce point a son importance : une erreur de carte et le DNS ne répond plus.

Le plan d'adressage est fixe :

| Machine | Adresse | Passerelle | DNS |
|---|---|---|---|
| WIN-DC01 | 192.168.50.10/24 | aucune sur VMnet2 | 192.168.50.10 (lui-même) |
| LINUX01 | 192.168.50.20/24 | aucune sur VMnet2 | 192.168.50.10 |
| OPS01 | 192.168.50.30/24 | aucune sur VMnet2 | 192.168.50.10 |

Le domaine Active Directory est `infraready.test`, avec `INFRAREADY` comme nom NetBIOS. Le choix de `.test` est volontaire : c'est un domaine de laboratoire, qui ne doit pas être confondu avec un domaine public.

L'environnement RHEL 9 de validation est, lui, derrière un pfSense avec des VLAN ; il réutilise le même plan d'adressage, mais il ne fait pas partie du schéma VMware à trois VM.

### 2.3. Les flux autorisés

| Source | Destination | Port | Usage |
|---|---|---|---|
| OPS01 | LINUX01 | TCP 9100 | Collecte node_exporter |
| OPS01 | WIN-DC01 | TCP 9182 | Collecte windows_exporter |
| OPS01 | LINUX01 | TCP 22 | Administration SSH |
| OPS01 | WIN-DC01 | TCP 5985 | WinRM pour Ansible |
| LINUX01 / OPS01 | WIN-DC01 | TCP/UDP 53 | Résolution DNS |
| VM du labo | Dépôts officiels | TCP 443 | Mises à jour et téléchargements via NAT |

Nous n'avons pas ouvert de plage « tout autoriser ». Les flux de supervision sont limités aux adresses du laboratoire, et WinRM reste sur le réseau privé. Le RDP vers Windows n'est pas nécessaire au fonctionnement du projet ; la console VMware suffit pour l'administration.

### 2.4. Pourquoi ces choix

Le réseau host-only évite que les VM soient joignables depuis le LAN de l'établissement. Le NAT est nécessaire pour `apt`, `dnf` et les téléchargements, mais il ne doit pas servir à exposer les services. Autrement dit, Internet sert à installer, pas à publier.

Les adresses fixes simplifient le DNS et la supervision : Prometheus scrape des adresses connues, et le contrôleur de domaine ne dépend pas d'un bail DHCP. Le DHCP de VMnet2 est donc désactivé. C'est aussi une source d'erreur classique en laboratoire : si le DHCP reste actif, une VM peut prendre une adresse hors plan et le DNS devient incohérent.

---

## 3. Installation des systèmes <a id="3-installation-des-systemes"></a>

### 3.1. VMware et les ISO

Les VM ont été créées dans VMware Workstation Pro à partir des ISO officielles : Windows Server 2025 Evaluation avec Desktop Experience, et Ubuntu Server 24.04 LTS. Les ressources ont été ajustées au fil des besoins : 50 Go pour Windows, 30 Go pour les deux Ubuntu, plus un disque de 5 Go ajouté ensuite à LINUX01.

Un snapshot `00-installation-propre` a été pris après l'installation et les premières mises à jour. Nous avons gardé ce réflexe pour la suite : avant un changement de rôle ou une manipulation de stockage, un snapshot nommé et daté évite de reconstruire la VM en cas d'erreur.

![Configuration de la VM Windows Server](images/1.%20Configuration%20VM%20Windows%20Server.png)

*Image 1 — Configuration matérielle de la VM Windows Server 2025.*

![Windows Server à jour](images/2.%20Windows%20Server%20a%20jour.png)

*Image 2 — Windows Update confirme que le serveur est à jour.*

![Configuration de la VM LINUX01](images/3.%20Configuration%20VM%20LINUX01.png)

*Image 3 — Paramètres de la VM LINUX01.*

![Configuration de la VM OPS01](images/4.%20Configuration%20VM%20OPS01.png)

*Image 4 — Paramètres de la VM OPS01.*

### 3.2. Le réseau privé

Le réseau VMnet2 a été créé dans le Virtual Network Editor. Il est de type host-only, en `192.168.50.0/24`, avec le DHCP local désactivé. La case « Use local DHCP service » est décochée, et les adresses sont posées à la main dans les systèmes.

![Réseau privé VMnet2](images/5.%20Reseau%20prive%20VMnet2.png)

*Image 5 — VMnet2 est un réseau host-only en 192.168.50.0/24, sans DHCP.*

Côté Windows, l'adresse `192.168.50.10` est configurée sur `Ethernet0`, et le client DNS pointe vers `192.168.50.10`. L'interface NAT `Ethernet1` garde une adresse en DHCP, ce qui permet les téléchargements sans toucher au réseau privé.

![Configuration DNS Windows](images/6.%20Configuration%20DNS%20Windows.png)

*Image 6 — Le client DNS de WIN-DC01 pointe vers le futur serveur DNS.*

![Adresses IP Windows](images/7.%20Adresses%20IP%20Windows.png)

*Image 7 — Deux interfaces : le réseau privé en 192.168.50.10 et le NAT en 192.168.28.137.*

Côté Ubuntu, Netplan configure deux interfaces. `ens33` reste en DHCP pour le NAT ; `ens37` reçoit l'adresse fixe et le DNS interne. La commande `netplan try` permet de revenir en arrière si la configuration coupe la connexion, ce qui est rassurant quand on travaille à distance.

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      addresses:
        - 192.168.50.20/24
      nameservers:
        addresses:
          - 192.168.50.10
```

![Configuration Netplan LINUX01](images/8.%20Configuration%20Netplan%20LINUX01.png)

*Image 8 — Netplan de LINUX01 : NAT en DHCP, réseau privé en 192.168.50.20/24.*

![Vérification réseau LINUX01](images/9.%20Verification%20reseau%20LINUX01.png)

*Image 9 — L'adresse 192.168.50.20/24 est bien appliquée sur ens37.*

![Configuration Netplan OPS01](images/10.%20Configuration%20Netplan%20OPS01.png)

*Image 10 — Netplan de OPS01 : adresse 192.168.50.30/24 et DNS 192.168.50.10.*

![Vérification réseau OPS01](images/11.%20Verification%20reseau%20OPS01.png)

*Image 11 — L'adresse 192.168.50.30/24 est bien appliquée sur ens37.*

### 3.3. Les premiers tests

Le premier test est simple : un ping entre les trois VM. Depuis LINUX01, les pings vers `192.168.50.10` et `192.168.50.30` passent avec 0 % de perte. Cela prouve que le réseau VMnet2 et l'adressage sont cohérents, pas encore que les services fonctionnent.

![Test de connectivité des VM](images/12.%20Test%20connectivite%20des%20VM.png)

*Image 12 — Les pings entre LINUX01, WIN-DC01 et OPS01 passent.*

---

## 4. Administration Windows <a id="4-administration-windows"></a>

### 4.1. Installation d'AD DS et promotion

Le rôle Active Directory Domain Services a été installé depuis le Gestionnaire de serveur, avec les outils d'administration. La promotion crée une nouvelle forêt `infraready.test`, avec le nom NetBIOS `INFRAREADY` et l'installation DNS intégrée. Le mot de passe DSRM a été défini au moment de la promotion et stocké hors du dépôt.

![Installation AD DS](images/13.%20Installation%20AD%20DS.png)

*Image 13 — Installation du rôle AD DS sur WIN-DC01.*

Après le redémarrage, la connexion se fait avec le compte de domaine `INFRAREADY\admin`. Le Gestionnaire de serveur confirme la présence des rôles AD DS et DNS. Le domaine est opérationnel, et les clients du laboratoire pointent tous vers `192.168.50.10`.

![Promotion contrôleur de domaine](images/14.%20Promotion%20controleur%20de%20domaine.png)

*Image 14 — Connexion au domaine après la promotion.*

![Rôles AD DS et DNS](images/15.%20Roles%20AD%20DS%20et%20DNS.png)

*Image 15 — AD DS et DNS sont installés sur le même serveur.*

### 4.2. Comptes, groupes et droits

Deux groupes globaux ont été créés : `GRP-Admins` et `GRP-Users`. Les comptes de test `Test User1`, `Test User2` et `Test Admin1` ont été créés en PowerShell, avec `Read-Host -AsSecureString` pour saisir le mot de passe sans l'écrire en clair dans l'historique. L'affectation des membres utilise `Add-ADGroupMember` :

```powershell
New-ADGroup -Name "GRP-Admins" -GroupScope Global
New-ADGroup -Name "GRP-Users" -GroupScope Global

New-ADUser -Name "Test Admin1" -SamAccountName "tadmin1" -Enabled $true `
  -AccountPassword (Read-Host -AsSecureString "Mot de passe")
# même principe pour Test User1 (tuser1) et Test User2 (tuser2)

Add-ADGroupMember -Identity "GRP-Admins" -Members "tadmin1"
Add-ADGroupMember -Identity "GRP-Users" -Members "tuser1","tuser2"

Get-ADGroupMember -Identity "GRP-Admins" | Select-Object Name
Get-ADGroupMember -Identity "GRP-Users" | Select-Object Name
Get-ADUser -Filter 'Name -like "Test*"' | Select-Object Name, SamAccountName, Enabled
```

| Groupe | Membres | Usage |
|---|---|---|
| `GRP-Admins` | Test Admin1 | Compte d'administration |
| `GRP-Users` | Test User1, Test User2 | Comptes standards |

![Création des groupes et utilisateurs AD](images/16.%20Creation%20groupes%20et%20utilisateurs%20AD.png)

*Image 16 — Création des groupes GRP-Admins / GRP-Users et des comptes de test.*

La console `dsa.msc` permet de vérifier que les comptes et les groupes sont bien dans le domaine. Les commandes `Get-ADUser`, `Get-ADGroup` et `Get-ADGroupMember` confirment les comptes, les deux groupes et leurs membres ; `Get-ADGroup -Filter 'Name -like "GRP-*"'` vérifie aussi leur portée `Global`.

![Console AD](images/17.%20Console%20AD.png)

*Image 17 — Les comptes du domaine dans Utilisateurs et ordinateurs Active Directory.*

![Vérification des groupes AD](images/18.%20Verification%20groupes%20AD.png)

*Image 18 — Les groupes GRP-Admins et GRP-Users sont présents.*

Le modèle de droits retenu reste simple, volontairement : `GRP-Admins` regroupe les comptes d'administration, `GRP-Users` les comptes standards. Les droits sont portés par l'appartenance aux groupes, pas par des autorisations posées au cas par cas sur chaque compte. La matrice détaillée des droits reste à formaliser (voir section 11.2), mais le principe est déjà appliqué. C'est un principe de base, mais il est facile de le contourner quand on est pressé. Nous avons préféré garder ce modèle lisible plutôt que d'empiler des exceptions.

Le compte `Administrateur` du domaine est utilisé par Ansible pour la connexion WinRM. C'est acceptable dans un laboratoire isolé, beaucoup moins en production. Nous revenons sur ce point dans les limites.

### 4.3. Vérification DNS

Une fois le contrôleur de domaine en place, nous avons vérifié la résolution depuis les clients. Sur OPS01, `resolvectl status` indique bien `192.168.50.10` comme serveur et `infraready.test` comme domaine de recherche. Depuis LINUX01, `nslookup win-dc01.infraready.test` renvoie `192.168.50.10`.

![Configuration DNS clients](images/19.%20Configuration%20DNS%20clients.png)

*Image 19 — OPS01 utilise le DNS interne et le domaine infraready.test.*

![Résolution DNS du domaine](images/20.%20Resolution%20DNS%20du%20domaine.png)

*Image 20 — Le nom win-dc01.infraready.test est résolu en 192.168.50.10.*

Le client n'a qu'un seul résolveur configuré : `192.168.50.10`. Pour les noms externes, le serveur DNS Windows transfère les requêtes qu'il ne connaît pas vers `1.1.1.1`, défini comme forwarder sur WIN-DC01. Les clients ne joignent donc jamais directement un résolveur public, ce qui évite de contourner les règles internes. Une alternative serait de déclarer un second contrôleur de domaine en réplication comme DNS secondaire : si le premier AD tombe, le second continuerait à appliquer la même politique. Nous ne l'avons pas mis en place dans le temps du projet.

### 4.4. Journaux Windows

Les journaux utiles sont consultables dans l'Observateur d'événements, ou en PowerShell avec `Get-WinEvent`. Pendant l'incident DNS, c'est surtout l'état du service et les tests de résolution qui ont orienté le diagnostic ; les journaux système restent la première source à consulter. La commande de base est :

```powershell
Get-WinEvent -LogName System -MaxEvents 30
Get-Service DNS
Get-NetTCPConnection -LocalPort 53
```

---

## 5. Administration Linux <a id="5-administration-linux"></a>

### 5.1. nginx

nginx a été installé sur LINUX01 avec `apt`, puis activé au démarrage. La page par défaut répond en HTTP 200 sur `192.168.50.20`, depuis la VM elle-même comme depuis le réseau privé. Le service tourne avec un processus maître et des workers, et `journalctl -u nginx` permet de retrouver les arrêts et redémarrages.

```bash
sudo apt install nginx -y
sudo systemctl enable --now nginx
curl -I http://192.168.50.20
```

![Installation nginx](images/21.%20Installation%20nginx.png)

*Image 21 — nginx est activé et répond en HTTP 200 sur LINUX01.*

![Page nginx depuis le réseau](images/22.%20Page%20nginx%20depuis%20le%20reseau.png)

*Image 22 — La page d'accueil de nginx est accessible depuis le navigateur.*

![Démarrage nginx](images/24.%20Demarrage%20nginx.png)

*Image 23 — Démarrage du service et vérification des processus nginx.*

![Statut nginx](images/25.%20Statut%20nginx.png)

*Image 24 — nginx est actif ; curl renvoie le HTML de la page par défaut.*

![Journal nginx](images/26.%20Journal%20nginx.png)

*Image 25 — Le service est enabled et journalctl retrace son historique.*

### 5.2. Groupe webadmins et permissions

Un groupe `webadmins` a été créé, avec `alice` comme membre. Le dossier applicatif `/srv/data` appartient à `root:webadmins` avec le mode `0770` : le propriétaire et le groupe peuvent lire, écrire et traverser ; les autres n'ont aucun droit. `alice` peut donc travailler dans le dossier, tandis qu'un utilisateur hors du groupe est refusé.

![Groupe webadmins et alice](images/27.%20Groupe%20webadmins%20et%20alice.png)

*Image 26 — Le groupe webadmins contient alice ; /srv/data est en root:webadmins 0770.*

### 5.3. Ajout et montage du disque

Un second disque SCSI de 5 Go a été ajouté à LINUX01. Après le scan, il apparaît en `/dev/sdb`, distinct du disque système `/dev/sda` de 30 Go. Le disque est formaté directement en ext4, sans table de partitions : pour un disque de données simple, cela suffit et évite une couche inutile.

L'UUID obtenu est `b645ea10-9d47-4015-be21-fdf77ed5047f`. Le montage est ajouté dans `/etc/fstab` par UUID, pas par nom de périphérique : c'est plus robuste si l'ordre des disques change. Le point de montage `/mnt/data` appartient à `root:webadmins` en `0770`.

```bash
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /mnt/data
sudo mount /dev/sdb /mnt/data
sudo blkid /dev/sdb
# /etc/fstab
UUID=b645ea10-9d47-4015-be21-fdf77ed5047f  /mnt/data  ext4  defaults  0  2
sudo mount -a
df -h /mnt/data
```

![Ajout du disque virtuel](images/28.%20Ajout%20disque%20virtuel%20LINUX01.png)

*Image 27 — Ajout d'un second disque SCSI à LINUX01.*

![Disque de 5 Go](images/29.%20Disque%20de%205%20Go.png)

*Image 28 — Le disque virtuel fait 5 Go.*

![Détection du disque sdb](images/30.%20Detection%20du%20disque%20sdb.png)

*Image 29 — /dev/sdb apparaît à côté du disque système /dev/sda.*

![Inventaire du disque système](images/31.%20Inventaire%20du%20disque%20systeme.png)

*Image 30 — Inventaire du disque système : partition d'amorçage et partition ext4 montée sur /.*

![Création du système de fichiers ext4](images/32.%20Creation%20du%20systeme%20de%20fichiers%20ext4.png)

*Image 31 — mkfs.ext4 sur /dev/sdb et UUID du système de fichiers.*

![Premier montage du disque](images/33.%20Premier%20montage%20du%20disque.png)

*Image 32 — Montage manuel sur /mnt/data et vérification avec df -h.*

![Montage persistant fstab](images/34.%20Montage%20persistant%20fstab.png)

*Image 33 — Montage par UUID dans /etc/fstab, puis mount -a.*

La capture conserve l'avertissement de systemd « fstab has been modified… » ; `systemctl daemon-reload` a été exécuté sans erreur pour recharger la configuration. Cet avertissement n'indique pas un échec du montage.

Le test de permissions a été fait dans les deux sens. `alice`, membre de `webadmins`, peut créer un fichier dans `/mnt/data`. `nobody`, qui n'est pas dans le groupe, reçoit un `Permission denied`. C'est exactement le comportement attendu : le contrôle d'accès n'est pas seulement écrit dans un fichier de configuration, il est vérifié.

![Permissions du dossier partagé](images/35.%20Permissions%20du%20dossier%20partage.png)

*Image 34 — alice peut écrire dans /mnt/data ; le dossier reste root:webadmins 0770.*

![Test des permissions](images/36.%20Test%20permissions%20utilisateur.png)

*Image 35 — Un utilisateur hors du groupe ne peut pas écrire : Permission denied.*

Après redémarrage, `df -h /mnt/data` montre que le montage est toujours là. C'est le test qui valide réellement la persistance ; un montage manuel qui fonctionne ne prouve rien sur la tenue au reboot.

![Montage après redémarrage](images/37.%20Montage%20apres%20redemarrage.png)

*Image 36 — Le montage est toujours actif après redémarrage.*

### 5.4. Journaux Linux

Les journaux consultés sont ceux d'APT, de nginx et de l'authentification :

```bash
tail -20 /var/log/apt/history.log
journalctl -u nginx --no-pager | tail -10
sudo tail -20 /var/log/auth.log
```

Les horodatages sont en UTC. C'est visible sur les captures et cela peut surprendre quand on compare avec l'heure locale. Nous avons pris l'habitude de convertir avant de corréler deux événements.

![Analyse des journaux](images/38.%20Analyse%20des%20journaux.png)

*Image 37 — Extrait de journalctl et de /var/log/auth.log : nginx et sessions SSH.*

---

## 6. Automatisation avec Ansible <a id="6-automatisation-avec-ansible"></a>

### 6.1. Installation et connexion SSH

Ansible est installé sur OPS01. La version utilisée est Ansible 2.16.3, avec Python 3.12.3. La connexion vers LINUX01 se fait par clé ed25519 : `ssh-keygen`, puis `ssh-copy-id server@192.168.50.20`. À partir de là, plus de mot de passe pour les commandes SSH.

```bash
sudo apt install ansible -y
ssh-keygen -t ed25519
ssh-copy-id server@192.168.50.20
ansible --version
```

Les collections nécessaires (`ansible.posix`, `ansible.windows`, `microsoft.ad`) sont installées avec `ansible-galaxy collection install -r requirements.yml` ; le fichier `requirements.yml` est versionné à la racine du dépôt.

![Installation d'Ansible](images/39.%20Installation%20Ansible.png)

*Image 38 — Ansible 2.16.3 sur OPS01.*

![Connexion SSH sans mot de passe](images/40.%20Connexion%20SSH%20sans%20mot%20de%20passe.png)

*Image 39 — Connexion SSH à LINUX01 avec la clé ed25519.*

### 6.2. Inventaire

L'inventaire est un fichier YAML unique, `inventory/hosts.yml`. Il déclare le groupe `linux` avec `linux01`, puis le groupe `windows` avec `win-dc01`. Les variables de connexion Linux sont l'adresse, l'utilisateur `server` et la clé privée. Les variables Windows sont dans `group_vars/windows/vars.yml`.

```yaml
all:
  children:
    linux:
      hosts:
        linux01:
          ansible_host: 192.168.50.20
          ansible_user: server
          ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    windows:
      hosts:
        win-dc01:
          ansible_host: 192.168.50.10
```

![Inventaire Ansible Linux](images/41.%20Inventaire%20Ansible%20Linux.png)

*Image 40 — Le fichier hosts.yml déclare linux01.*

![Test ansible ping](images/42.%20Test%20ansible%20ping.png)

*Image 41 — Le module ping répond pong sur linux01.*

### 6.3. Playbook Linux

Le playbook `playbooks/linux.yml` enchaîne neuf tâches : créer le groupe `webadmins`, créer `alice`, créer `/srv/data` en `root:webadmins 0770`, installer nginx, activer le service, monter le disque par UUID, appliquer les permissions du point de montage, puis vérifier que nginx répond et que le disque est monté. Les deux dernières tâches sont des vérifications : `uri` attend un code 200 et `findmnt` vérifie le montage.

```yaml
- name: Configuration de LINUX01
  hosts: linux
  become: true
  vars:
    admin_group: webadmins
    admin_user: alice
    data_dir: /srv/data
    disk_uuid: "b645ea10-9d47-4015-be21-fdf77ed5047f"
    mount_point: /mnt/data
  tasks:
    - name: Créer le groupe {{ admin_group }}
      ansible.builtin.group:
        name: "{{ admin_group }}"
        state: present
    - name: Créer l'utilisateur {{ admin_user }}
      ansible.builtin.user:
        name: "{{ admin_user }}"
        shell: /bin/bash
        groups: "{{ admin_group }}"
        append: true
        create_home: true
    - name: Créer le dossier {{ data_dir }}
      ansible.builtin.file:
        path: "{{ data_dir }}"
        state: directory
        owner: root
        group: "{{ admin_group }}"
        mode: "0770"
    - name: Installer nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600
    - name: Activer et démarrer nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
    - name: Monter le disque de données (UUID)
      ansible.posix.mount:
        src: "UUID={{ disk_uuid }}"
        path: "{{ mount_point }}"
        fstype: ext4
        opts: defaults
        state: mounted
    - name: Permissions du point de montage
      ansible.builtin.file:
        path: "{{ mount_point }}"
        state: directory
        owner: root
        group: "{{ admin_group }}"
        mode: "0770"
    - name: Vérifier que nginx répond
      ansible.builtin.uri:
        url: http://localhost
        status_code: 200
    - name: Vérifier que le disque est monté
      ansible.builtin.command: findmnt {{ mount_point }}
      changed_when: false
```

![Création du playbook Linux](images/43.%20Creation%20playbook%20Linux.png)

*Image 42 — Création du dossier playbooks et de linux.yml.*

La première exécution a donné un `changed` sur le montage. Ce n'était pas une erreur du module, mais une ligne `/etc/fstab` qui n'était pas au format attendu par Ansible. Après correction, le second passage affiche 10 tâches `ok` et `changed=0` — 10 au total, car le récapitulatif inclut la collecte de faits (`Gathering Facts`) en plus des 9 tâches explicites du playbook. C'est la définition pratique de l'idempotence : rejouer le playbook ne modifie plus rien.

![Première exécution du playbook Linux](images/44.%20Premiere%20execution%20playbook%20Linux.png)

*Image 43 — Première exécution : le montage ressort en changed.*

![Deuxième exécution du playbook Linux](images/45.%20Deuxieme%20execution%20playbook%20Linux.png)

*Image 44 — Deuxième exécution : 10 ok, 0 changed.*

### 6.4. WinRM et connexion Windows

Pour Windows, Ansible passe par WinRM. Sur WIN-DC01, `Enable-PSRemoting -Force` active le service et crée le listener HTTP sur le port 5985. Les règles de pare-feu `WINRM-HTTP-In-TCP` sont activées. Depuis OPS01, `nc -zv 192.168.50.10 5985` confirme que le port est joignable.

Côté OPS01, `python3-winrm` était déjà installé en version 0.4.3-2. L'import `winrm` fonctionne, et `ansible.windows.win_ping` répond `pong`. C'est le test le plus important avant d'écrire le playbook : si WinRM ne passe pas, aucune tâche Windows ne passera.

![Activation WinRM](images/46.%20Activation%20WinRM.png)

*Image 45 — WinRM est actif et écoute sur le port 5985.*

![Pare-feu WinRM](images/47.%20Pare-feu%20WinRM.png)

*Image 46 — Les règles de pare-feu WinRM sont activées.*

![Installation pywinrm](images/48.%20Installation%20pywinrm.png)

*Image 47 — python3-winrm est déjà présent sur OPS01.*

![Test port WinRM](images/49.%20Test%20port%20WinRM.png)

*Image 48 — Le port 5985 est joignable depuis OPS01.*

![Vérification pywinrm](images/50.%20Verification%20pywinrm.png)

*Image 49 — L'import du module Python winrm est opérationnel.*

### 6.5. Variables et Ansible Vault

Le fichier `group_vars/windows/vars.yml` contient les paramètres de connexion : `ansible_connection: winrm`, port 5985, transport NTLM, schéma HTTP, utilisateur `INFRAREADY\Administrateur`. Le mot de passe n'est pas écrit dans ce fichier : il vient de la variable `vault_windows_password`, stockée dans `vault.yml`.

```yaml
ansible_connection: winrm
ansible_port: 5985
ansible_winrm_transport: ntlm
ansible_winrm_scheme: http
ansible_user: INFRAREADY\Administrateur
ansible_password: "{{ vault_windows_password }}"
```

Le fichier `vault.yml` est chiffré avec `ansible-vault`. Les commandes utilisées sont `ansible-vault edit` pour le modifier et `ansible-vault view` pour le relire. Le mot de passe du vault n'est pas dans le dépôt ; il est fourni au moment de l'exécution avec `--ask-vault-pass`.

![Inventaire avec Windows](images/51.%20Inventaire%20avec%20Windows.png)

*Image 50 — Le groupe windows et l'hôte win-dc01 sont ajoutés à l'inventaire.*

![Création des variables Windows](images/52.%20Creation%20variables%20Windows.png)

*Image 51 — Création de group_vars/windows et de vars.yml.*

![Variables de connexion Windows](images/53.%20Variables%20connexion%20Windows.png)

*Image 52 — Les paramètres WinRM, avec le mot de passe référencé depuis le vault.*

![Création du vault Ansible](images/54.%20Creation%20du%20vault%20Ansible.png)

*Image 53 — Édition du fichier vault.yml avec ansible-vault.*

![Mot de passe du vault](images/55.%20Mot%20de%20passe%20du%20vault.png)

*Image 54 — Le secret est saisi dans le fichier déchiffré, jamais dans vars.yml.*

![Vérification du vault](images/56.%20Verification%20du%20vault.png)

*Image 55 — La commande ansible-vault view permet de relire le secret.*

![Test win_ping](images/57.%20Test%20win_ping.png)

*Image 56 — win_ping répond pong sur win-dc01.*

### 6.6. Playbook Windows

Le playbook `playbooks/windows.yml` crée un dossier de partage, crée un groupe AD, vérifie que le service DNS tourne, vérifie que le domaine répond et affiche le domaine. L'exécution présentée donne 6 tâches `ok` et `changed=0` : 5 tâches explicites, plus la collecte de faits (`Gathering Facts`).

![Création du playbook Windows](images/58.%20Creation%20playbook%20Windows.png)

*Image 57 — Création de windows.yml.*

![Vérification syntaxe Windows](images/59.%20Verification%20syntaxe%20Windows.png)

*Image 58 — Le playbook Windows passe le contrôle de syntaxe.*

![Exécution du playbook Windows](images/60.%20Execution%20playbook%20Windows.png)

*Image 59 — 6 tâches ok, 0 changed, et le domaine infraready.test est affiché.*

### 6.7. Vérifications de l'inventaire et des playbooks

Avant d'exécuter un playbook, nous vérifions l'inventaire et la syntaxe. C'est fastidieux, mais cela évite de découvrir une erreur au milieu d'une exécution.

```bash
ansible-inventory -i inventory/hosts.yml --graph
ansible-inventory -i inventory/hosts.yml --host linux01
python3 -c "import yaml; yaml.safe_load(open('inventory/hosts.yml')); print('YAML valide')"
yamllint inventory/
ansible-playbook -i inventory/hosts.yml playbooks/windows.yml --syntax-check
ansible-playbook -i inventory/hosts.yml playbooks/linux.yml --list-hosts
ansible-playbook -i inventory/hosts.yml playbooks/windows.yml --list-hosts
```

![Inventaire graphique](images/61.%20Inventaire%20graphique.png)

*Image 60 — Les groupes linux et windows avec leurs hôtes.*

![Variables chargées de l'inventaire](images/62.%20Variables%20chargees%20inventaire.png)

*Image 61 — Les variables Windows sont bien chargées, mot de passe inclus par référence au vault.*

![Détails hôte Linux](images/63.%20Details%20hote%20Linux.png)

*Image 62 — Les variables de linux01 : adresse, utilisateur et clé SSH.*

![Validation YAML](images/64.%20Validation%20YAML.png)

*Image 63 — Le fichier hosts.yml est valide.*

![Validation yamllint](images/65.%20Validation%20yamllint.png)

*Image 64 — yamllint ne signale aucune erreur.*

![Test des deux hôtes](images/66.%20Test%20des%20deux%20hotes.png)

*Image 65 — Les deux hôtes répondent : ping Linux et win_ping Windows.*

![Ciblage des playbooks](images/67.%20Ciblage%20des%20playbooks.png)

*Image 66 — Chaque playbook cible le bon groupe d'hôtes.*

### 6.8. Rôles et déploiement des exporters

Le projet Ansible est organisé en rôles. `install_exporter.yml` applique `windows_exporter` aux hôtes Windows et `node_exporter` aux hôtes Linux. Chaque rôle contient ses `tasks`, ses `handlers`, ses `vars` et, pour Windows, un template de configuration. Les handlers servent à recharger systemd et à redémarrer le service quand la configuration change.

Les captures de cette sous-section proviennent de l'environnement RHEL 9 de validation ; la cible `linux01` y est la VM RHEL, ce qui explique que la tâche APT soit `skipped`. Sur le LINUX01 Ubuntu du laboratoire de référence, la tâche APT s'exécute normalement, et `ansible.builtin.package` utilise APT au lieu de DNF.

```text
inventory/
  group_vars/
    windows/
      vars.yml
      vault.yml
  hosts.yml
playbooks/
  install_exporter.yml
  linux.yml
  roles/
    node_exporter/
      handlers/main.yml
      tasks/main.yml
      vars/main.yml
    windows_exporter/
      handlers/main.yml
      tasks/main.yml
      templates/config.yaml.j2
      vars/main.yml
  windows.yml
```

Le module `ansible.builtin.package` est utilisé pour installer `tar` et `gzip` : il choisit automatiquement APT ou DNF selon la distribution. La mise à jour du cache APT est conditionnée par `ansible_facts['os_family'] == "Debian"`. C'est ce qui permet au même playbook de tourner sur Ubuntu et sur RHEL 9.

![Mise à jour du cache APT](images/69.%20Mise%20a%20jour%20cache%20APT.png)

*Image 67 — La tâche APT ne s'exécute que sur la famille Debian.*

![Installation des paquets prérequis](images/70.%20Installation%20paquets%20prerequis.png)

*Image 68 — ansible.builtin.package installe tar et gzip.*

![Arborescence du projet Ansible](images/71.%20Arborescence%20projet%20Ansible.png)

*Image 69 — Organisation du projet en rôles, avec tasks, handlers, vars et templates.*

![Playbook des exporters](images/72.%20Playbook%20exporters.png)

*Image 70 — install_exporter.yml applique les deux rôles.*

![Installation de Windows Exporter](images/73.%20Installation%20Windows%20Exporter.png)

*Image 71 — Le rôle windows_exporter installe et démarre le service.*

![Installation de Node Exporter](images/74.%20Installation%20Node%20Exporter.png)

*Image 72 — Le rôle node_exporter crée l'utilisateur, installe le binaire et le service systemd.*

![Handlers et bilan d'exécution](images/75.%20Handlers%20et%20bilan%20execution.png)

*Image 73 — Les handlers rechargent systemd et redémarrent node_exporter.*

La première exécution a produit des `changed` normaux : création d'utilisateur, de dossier, téléchargement, installation du service. La seconde exécution donne `changed=0` sur les deux hôtes de cet environnement. Les deux tâches `skipped` sont attendues : le cache APT n'est pas utilisé sur RHEL, et l'archive de node_exporter est déjà présente avec le numéro de version dans le nom, donc Ansible saute le téléchargement. Sur Ubuntu, l'alternative manuelle est `sudo apt update` puis `sudo apt install tar gzip` ; le playbook fait la même chose avec `ansible.builtin.package`.

![Idempotence des exporters](images/76.%20Idempotence%20des%20exporters.png)

*Image 74 — Deuxième exécution : 0 changed sur linux01 et win-dc01.*

![Vérification des endpoints](images/77.%20Verification%20endpoints%20exporters.png)

*Image 75 — Les endpoints 9100 et 9182 répondent en HTTP 200.*

---

## 7. Supervision avec Prometheus et Grafana <a id="7-supervision-avec-prometheus-et-grafana"></a>

### 7.1. Prometheus

Prometheus a été installé sur l'environnement RHEL 9 de validation (hôte `AdminLinux-OPS01`), avec le paquet `prometheus` 3.15.0 du dépôt EPEL. Sur le laboratoire de référence, OPS01 est Ubuntu : l'alternative est `sudo apt install prometheus` (paquet de la distribution, version plus ancienne) ou l'installation du binaire officiel. La configuration `/etc/prometheus/prometheus.yml` et le service `prometheus` sont identiques sur les deux distributions.

Le fichier `/etc/prometheus/prometheus.yml` déclare trois jobs : `prometheus` pour lui-même, `linux` pour `192.168.50.20:9100` et `windows` pour `192.168.50.10:9182`. Le service est activé au démarrage avec `systemctl enable --now prometheus`.

```yaml
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: linux
    static_configs:
      - targets: ['192.168.50.20:9100']
  - job_name: windows
    static_configs:
      - targets: ['192.168.50.10:9182']
```

![Installation de Prometheus](images/78.%20Installation%20Prometheus.png)

*Image 76 — Installation de Prometheus 3.15.0 sur RHEL 9.*

![Configuration des cibles Prometheus](images/79.%20Configuration%20cibles%20Prometheus.png)

*Image 77 — Les jobs linux et windows pointent vers les deux exporters.*

![Démarrage de Prometheus](images/80.%20Demarrage%20Prometheus.png)

*Image 78 — Activation et démarrage immédiat du service.*

![Statut de Prometheus](images/81.%20Statut%20Prometheus.png)

*Image 79 — Prometheus est actif et écoute sur 9090.*

La page `Status > Target health` montre les trois jobs en `UP` : `prometheus`, `linux` et `windows`. Les deux exporters sont donc bien scrapés, et la capture correspond à la configuration finale.

![Cibles Prometheus](images/82.%20Cibles%20Prometheus.png)

*Image 80 — Les jobs linux, windows et prometheus sont UP dans Prometheus.*

### 7.2. Grafana

Grafana est installé sur le même hôte RHEL 9 et écoute sur le port 3000 ; Prometheus reste sur 9090. Sur Ubuntu, l'alternative d'installation est `sudo apt install grafana` après ajout du dépôt officiel Grafana ; le service `grafana-server` et le port restent identiques. La source de données est ajoutée avec l'URL `http://localhost:9090`, puis validée avec `Save & test`. Le message « Successfully queried the Prometheus API » confirme que la chaîne Grafana → Prometheus fonctionne.

![Installation de Grafana](images/83.%20Installation%20Grafana.png)

*Image 81 — Grafana est activé et écoute sur 3000 ; Prometheus sur 9090.*

![Interface Grafana](images/84.%20Interface%20Grafana.png)

*Image 82 — L'interface Grafana est accessible depuis le navigateur.*

![Ajout d'une source de données](images/85.%20Ajout%20source%20de%20donnees.png)

*Image 83 — Ajout de la première source de données.*

![Choix de Prometheus](images/86.%20Choix%20Prometheus.png)

*Image 84 — Prometheus est sélectionné comme type de source.*

![Configuration de la source Prometheus](images/87.%20Configuration%20source%20Prometheus.png)

*Image 85 — La source pointe vers http://localhost:9090.*

![Test de la source de données](images/88.%20Test%20source%20de%20donnees.png)

*Image 86 — Save & test confirme l'interrogation de l'API Prometheus.*

![Source de données validée](images/89.%20Source%20de%20donnees%20validee.png)

*Image 87 — La source Prometheus est disponible et par défaut.*

### 7.3. Tableaux de bord

Deux dashboards ont été importés depuis la bibliothèque Grafana dans l'instance de l'environnement RHEL 9 : **Node Exporter Full** (ID 1860) pour Linux et **Windows Exporter Dashboard 2025** (ID 23942) pour Windows. Ils ne sont pas parfaits, mais ils couvrent l'essentiel : CPU, mémoire, disque, réseau et disponibilité. Les JSON officiels correspondants sont versionnés dans `monitoring/dashboards/`.

| Dashboard | Cible | Ce qu'on y lit |
|---|---|---|
| Node Exporter Full | LINUX01, `192.168.50.20:9100` | CPU, charge, RAM, swap, root FS, uptime |
| Windows Exporter Dashboard 2025 | WIN-DC01, `192.168.50.10:9182` | CPU, mémoire, volumes C:/D:, réseau, processus |

![Tableaux de bord importés](images/90.%20Tableaux%20de%20bord%20importes.png)

*Image 88 — Les deux dashboards importés dans Grafana.*

![Tableau de bord Linux](images/91.%20Tableau%20de%20bord%20Linux.png)

*Image 89 — Vue complète du dashboard Node Exporter Full.*

![Tableau de bord Windows](images/92.%20Tableau%20de%20bord%20Windows.png)

*Image 90 — Vue du dashboard Windows Exporter.*

![Détails des ressources Windows](images/93.%20Details%20ressources%20Windows.png)

*Image 91 — Détail des ressources Windows : CPU, mémoire et occupation des volumes.*

![Indicateurs Linux](images/94.%20Indicateurs%20Linux.png)

*Image 92 — Vue synthétique des indicateurs Linux.*

![Espace disque Linux](images/95.%20Espace%20disque%20Linux.png)

*Image 93 — Le panneau Root FS Used indique 20,3 % d'occupation.*

Sur le dashboard Windows, la machine apparaît sous le nom `WINSERV01`. C'est le nom remonté par windows_exporter, pas le nom d'hôte Active Directory. Ce détail peut désorienter au début ; il faut se fier à l'instance `192.168.50.10:9182` pour savoir de quelle machine il s'agit.

### 7.4. Alerte CPU

Une alerte Grafana surveille la CPU de LINUX01. La règle `cpu_linux_threshold` se déclenche quand la moyenne d'utilisation dépasse 70 % sur une fenêtre de 5 minutes. Pour la tester, nous avons monté la charge CPU à environ 90 %. L'état est d'abord passé en `Pending`, puis en `Firing` après confirmation. L'historique montre la valeur `A=95.4` et l'instance `192.168.50.20:9100`.

```text
Nom       : cpu_linux_threshold
Seuil     : CPU > 70 %
État      : Pending → Firing
Instance  : 192.168.50.20:9100
```

![Historique de l'alerte CPU](images/98.%20Historique%20alerte%20CPU.png)

*Image 94 — L'historique montre le passage de Normal à Pending puis Alerting.*

![Alerte CPU en attente](images/99.%20Alerte%20CPU%20en%20attente.png)

*Image 95 — L'alerte est Pending avec une valeur mesurée d'environ 91 %.*

![Alerte CPU déclenchée](images/100.%20Alerte%20CPU%20declenchee.png)

*Image 96 — L'alerte passe en Firing pour l'instance 192.168.50.20:9100.*

Le test de charge n'était pas un faux positif : la CPU est réellement montée, l'alerte a suivi l'état réel de la machine. C'est ce qu'on demande à une alerte : être déclenchée par une mesure, pas par une configuration.

---

## 8. Maintenance et mises à jour <a id="8-maintenance-et-mises-a-jour"></a>

### 8.1. Mises à jour Linux

Deux niveaux ont été mis en place. Pour les exporters, les versions sont centralisées dans les fichiers `vars/main.yml` des rôles. Il suffit de changer le numéro de version pour déployer une nouvelle release, à condition que cette version existe réellement : sinon, le playbook échoue au téléchargement. C'est une vérification simple mais indispensable.

Pour le système, un playbook dédié `update.yml` applique les mises à jour. Il utilise `ansible.builtin.apt` avec `upgrade: dist` sur Debian/Ubuntu et `ansible.builtin.dnf` avec `name: "*"` et `state: latest` sur Rocky/RHEL. Les deux tâches sont conditionnées par `ansible_facts['os_family']`.

```yaml
- name: Mettre à jour les paquets sur Debian et Ubuntu
  ansible.builtin.apt:
    update_cache: true
    upgrade: dist
  when: ansible_os_family == "Debian"

- name: Mettre à jour les paquets sur Rocky Linux et RHEL
  ansible.builtin.dnf:
    name: "*"
    state: latest
    update_cache: true
  when: ansible_os_family == "RedHat"
```

Le test présenté a été fait sur l'environnement RHEL 9 de validation : la tâche Debian est `skipped`, la tâche DNF passe en `changed`. Au second passage, elle repasse en `ok` : le système est déjà à jour. Dans le laboratoire de référence, LINUX01 est Ubuntu : c'est la tâche APT qui s'exécute et la tâche DNF qui est ignorée. Les deux cas sont donc couverts par le même playbook.

Équivalents manuels : `sudo apt update && sudo apt upgrade -y` sur Ubuntu/Debian, `sudo dnf update -y` sur RHEL/Rocky. Le playbook applique la même logique, avec l'idempotence et le compte rendu en plus.

![Playbook de mise à jour](images/96.%20Playbook%20de%20mise%20a%20jour.png)

*Image 97 — La tâche Debian est skipped, la tâche RHEL passe en changed.*

![Mise à jour idempotente](images/97.%20Mise%20a%20jour%20idempotente.png)

*Image 98 — Deuxième exécution : plus aucun changement.*

Nous avons volontairement écarté un cron qui mettrait à jour les machines sans intervention. Sur un laboratoire, ce n'est pas très grave ; sur un serveur, une mise à jour qui redémarre un service au mauvais moment peut couper une application. Le playbook de mise à jour est donc lancé manuellement, dans une fenêtre de maintenance, après snapshot.

### 8.2. Mises à jour Windows

Windows utilise Windows Update. La vérification se fait depuis les paramètres du serveur : le système indique s'il est à jour et propose les redémarrages nécessaires. Avant un redémarrage, nous vérifions les services qui doivent repartir (AD DS, DNS, WinRM) et nous prenons un snapshot si le changement est important.

![Windows Server à jour](images/2.%20Windows%20Server%20a%20jour.png)

*Rappel — Windows Update indique que le serveur est à jour.*

### 8.3. Contrôle de l'espace disque

Deux méthodes simples : `df -h` pour l'ensemble des systèmes de fichiers, `df -h /` pour la racine, et le panneau Grafana `Root FS Used` pour le suivi continu. Sur LINUX01, la racine est à 20,3 % d'occupation ; le disque de données `/mnt/data` est presque vide. Ce n'est pas critique, mais c'est le genre d'indicateur qu'il faut regarder avant d'installer une nouvelle version.

```bash
df -h
df -h /
df -h /mnt/data
```

### 8.4. Précautions avant un redémarrage ou un changement majeur

La procédure que nous appliquons est simple :

1. Prendre un snapshot VMware nommé et daté.
2. Vérifier l'état des services et l'espace disque avant de toucher à quoi que ce soit.
3. Prévenir l'autre membre du binôme et bloquer une fenêtre de maintenance.
4. Tester le playbook sur un seul hôte avant de le généraliser.
5. Après le changement, vérifier les services, les journaux et les métriques.
6. Garder un chemin de retour : snapshot, sauvegarde de configuration, ou redéploiement du playbook précédent.

Un snapshot n'est pas une sauvegarde indépendante. Il permet de revenir en arrière sur la VM, pas de restaurer un fichier perdu. Nous le traitons comme un filet de sécurité ponctuel, pas comme une politique de sauvegarde.

---

## 9. GitHub et intégration continue <a id="9-github-et-integration-continue"></a>

### 9.1. Organisation du dépôt

Le dépôt contient l'inventaire, les playbooks, les rôles, les configurations de supervision et les documents. L'arborescence reprend celle du sujet, avec quelques adaptations :

```text
infraready/
├── README.md
├── requirements.yml
├── .ansible-lint
├── .yamllint
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── windows/
│           ├── vars.yml
│           └── vault.yml
├── playbooks/
│   ├── linux.yml
│   ├── windows.yml
│   ├── update.yml
│   ├── install_exporter.yml
│   └── roles/
│       ├── node_exporter/
│       └── windows_exporter/
├── monitoring/
│   ├── prometheus.yml
│   └── dashboards/
│       ├── README.md
│       ├── 1860-node-exporter-full.json
│       └── 23942-windows-exporter-dashboard-2025.json
├── docs/
│   ├── architecture.md
│   ├── adressage-ip.md
│   ├── securite.md
│   ├── maintenance.md
│   ├── incident.md
│   ├── dossier.md
│   └── images/
└── .github/
    └── workflows/
        └── ansible-validate.yml
```

Le dépôt est public : [github.com/d3vex/infraready](https://github.com/d3vex/infraready). Le dossier technique est dans `docs/dossier.md` et les captures dans `docs/images/`. Les commits sont restés lisibles : un commit pour l'inventaire et les playbooks, puis des commits de correction ciblés. Le fichier `vault.yml` est chiffré ; le mot de passe du vault n'est jamais commité.

![Commits GitHub](images/68.%20Commits%20GitHub.png)

*Image 99 — Les dossiers inventory et playbooks sont versionnés.*

### 9.2. Workflow de validation

Le workflow `ansible-validate.yml` s'exécute à chaque push et à chaque pull request. Il installe `ansible-core`, `ansible-lint` et `yamllint`, installe les collections déclarées dans `requirements.yml`, puis exécute `yamllint`, `ansible-lint` et un contrôle de la configuration Ansible. Il ne se connecte pas aux VM : un runner GitHub hébergé ne peut pas joindre le réseau VMware privé. La CI valide donc le dépôt, pas le déploiement.

L'historique montre une exécution en échec, puis une exécution réussie après correction du chemin des collections et du runner. C'est exactement l'intérêt du pipeline : une erreur de lint, de syntaxe ou une collection manquante est détectée avant d'arriver sur les serveurs.

![Workflow GitHub Actions](images/101.%20Workflow%20GitHub%20Actions.png)

*Image 100 — Le workflow Validate Ansible échoue, est corrigé, puis réussit.*

---

## 10. Diagnostic d'incidents <a id="10-diagnostic-dincidents"></a>

Les deux incidents ont été provoqués volontairement, sur des services arrêtés, avec un snapshot pris avant. Ils suivent la même méthode : symptôme, tests et journaux, cause, correction, vérification du retour à la normale.

### 10.1. Incident 1 — node_exporter arrêté sur LINUX01

**Auteur :** Alexandre BOZZI  
**Machine :** LINUX01 (192.168.50.20)  
**Gravité :** faible (laboratoire), métrique indisponible  
**Réversibilité :** snapshot VMware pris avant l'incident

**Symptôme.** Prometheus affiche la cible `linux` en `DOWN`, avec une erreur de connexion refusée sur `192.168.50.20:9100`. Le dashboard Grafana de LINUX01 ne remonte plus de données. Le problème est limité à la collecte de métriques : les autres services de LINUX01 ne sont pas en cause.

**Tests et journaux.** `systemctl status node_exporter` indique `inactive (dead)` depuis 08:29. `journalctl` montre les arrêts et redémarrages du service ; `ss -lntp` ne montre plus d'écoute sur le port 9100, et un `curl` vers l'endpoint échoue.

```bash
systemctl status node_exporter
journalctl -u node_exporter --no-pager | tail -20
ss -lntp | grep 9100
curl -I http://192.168.50.20:9100/metrics
```

![Cible Linux indisponible](images/102.%20Cible%20Linux%20indisponible.png)

*Image 101 — Prometheus voit la cible linux en DOWN : connexion refusée sur 9100.*

![Tableau de bord sans données](images/103.%20Tableau%20de%20bord%20sans%20donnees.png)

*Image 102 — Le dashboard Linux ne remonte plus de données.*

![Journal node_exporter](images/104.%20Journal%20node_exporter.png)

*Image 103 — journalctl montre les arrêts et redémarrages successifs du service.*

![Service node_exporter arrêté](images/105.%20Service%20node_exporter%20arrete.png)

*Image 104 — node_exporter est inactive (dead).*

**Cause.** Le service `node_exporter` a été arrêté volontairement avec `sudo systemctl stop node_exporter`. Plus aucun processus n'écoutait sur le port 9100, donc Prometheus ne pouvait plus scraper la cible.

**Correction.** Le service a été redémarré, puis vérifié :

```bash
sudo systemctl restart node_exporter
sudo systemctl enable node_exporter
systemctl status node_exporter
```

![Service node_exporter relancé](images/106.%20Service%20node_exporter%20relance.png)

*Image 105 — node_exporter est active (running) et écoute de nouveau sur 9100.*

**Vérification.** La cible est repassée en `UP`, et les métriques sont revenues dans Grafana. Un test de charge CPU à 92 % est visible sur le dashboard, ce qui prouve que la collecte fonctionne et que l'alerte peut de nouveau se déclencher.

![Retour à la normale](images/107.%20Retour%20a%20la%20normale.png)

*Image 106 — Les métriques remontent ; le test de charge à 92 % est visible.*

**Prévention.** Le service est maintenant `enabled` au démarrage. Une alerte sur `up{job="linux"} == 0` permettrait de détecter l'arrêt avant de regarder le dashboard. C'est un point que nous ajouterions si le projet continuait.

### 10.2. Incident 2 — service DNS arrêté sur WIN-DC01

**Auteur :** Loan Mata  
**Machine :** WIN-DC01 (192.168.50.10)  
**Gravité :** faible (laboratoire), résolution du domaine `infraready.test` indisponible  
**Réversibilité :** snapshot VMware de WIN-DC01 pris avant l'incident

**Symptôme.** Depuis OPS01, `nslookup win-dc01.infraready.test` échoue avec un délai dépassé. Les noms du domaine ne sont plus résolus. Le ping ICMP vers `192.168.50.10` renvoie `Destination Host Unreachable` ; ce test a été lancé alors que la VM Windows était éteinte, derrière le pfSense et ses VLAN, donc il ne reflète pas l'état du service DNS. Après redémarrage de la VM, le diagnostic s'est concentré sur le service DNS.

**Tests et journaux.** `Get-Service DNS` renvoie `Stopped`. Le port 53 n'est plus en écoute. `Resolve-DnsName` échoue. Le service n'a pas planté : il a été arrêté volontairement avec `Stop-Service -Name DNS`.

```powershell
Stop-Service -Name DNS
Get-Service DNS
Get-NetTCPConnection -LocalPort 53
Resolve-DnsName win-dc01.infraready.test
```

![Arrêt du service DNS](images/108.%20Arret%20du%20service%20DNS.png)

*Image 107 — L'incident est provoqué avec Stop-Service -Name DNS.*

![Échec de résolution DNS](images/109.%20Echec%20de%20resolution%20DNS.png)

*Image 108 — Le ping échoue car la VM est éteinte ; la résolution DNS ne répond pas non plus.*

![Service DNS arrêté](images/110.%20Service%20DNS%20arrete.png)

*Image 109 — Get-Service DNS confirme que le service est Stopped.*

**Cause.** Le service DNS a été arrêté sur le contrôleur de domaine. Plus aucun processus n'écoutait sur le port 53, donc les clients ne pouvaient plus résoudre `infraready.test`. Le message `Destination Host Unreachable` provenait de l'extinction de la VM au moment du ping, pas du service DNS ; le pfSense et les VLAN expliquent ce comportement. Le ping ICMP n'étant pas un indicateur fiable, nous avons vérifié l'état de la VM puis interrogé directement le service DNS ; c'était bien la cause directe.

**Correction.** Le service a été redémarré et configuré en démarrage automatique :

```powershell
Start-Service -Name DNS
Set-Service -Name DNS -StartupType Automatic
Get-Service DNS
```

![Redémarrage du service DNS](images/112.%20Redemarrage%20du%20service%20DNS.png)

*Image 110 — Start-Service puis démarrage automatique ; le service est Running.*

**Vérification.** `Get-NetTCPConnection -LocalPort 53` montre de nouveau des sockets en écoute sur `127.0.0.1`, `::1` et `192.168.50.10`. `Resolve-DnsName win-dc01.infraready.test` renvoie bien `192.168.50.10`. La résolution du domaine est rétablie.

![Port 53 en écoute](images/113.%20Port%2053%20en%20ecoute.png)

*Image 111 — Le serveur écoute de nouveau sur le port 53.*

![Résolution du domaine rétablie](images/114.%20Resolution%20du%20domaine%20retablie.png)

*Image 112 — Resolve-DnsName renvoie 192.168.50.10.*

La réponse DNS contient aussi un enregistrement A `192.168.28.137`, qui correspond à l'interface NAT de WIN-DC01. La VM possède deux cartes réseau (une carte NAT/bridge et une carte VMnet2 pour le réseau privé) et Windows enregistre par défaut les adresses des deux interfaces dans la zone. Ce n'est donc pas une anomalie de la zone : l'enregistrement reflète la double connectivité de la machine. Pour ne publier que l'adresse du réseau de service, il faudrait désactiver l'enregistrement DNS sur la carte NAT (`Set-DnsClient -InterfaceAlias "Ethernet1" -RegisterThisConnectionsAddress $false`) ou supprimer l'enregistrement ; nous avons conservé le comportement par défaut, qui ne gêne pas le laboratoire puisque l'inventaire Ansible utilise les adresses IP.

**Prévention.** Le service DNS est maintenant en démarrage automatique. Une sonde de supervision sur le port 53 ou sur une requête DNS connue permettrait de détecter l'arrêt. Sur un contrôleur de domaine, ce type de service ne devrait pas être arrêté sans vérifier l'impact sur les clients.

### 10.3. Ce que ces deux incidents nous ont appris

Les deux pannes se ressemblent : un service arrêté, un port qui n'écoute plus, une chaîne de supervision qui perd ses données. Mais les symptômes visibles n'étaient pas les mêmes. Pour node_exporter, Prometheus voyait immédiatement la cible en `DOWN`. Pour le DNS, le ping échouait aussi, ce qui brouillait le diagnostic : la VM était éteinte au moment de ce test, derrière le pfSense et ses VLAN. Une fois la VM redémarrée, c'est le service DNS arrêté qui expliquait la panne.

La leçon est simple : il faut distinguer la couche réseau, le port en écoute et le service applicatif. Un ping qui passe ne prouve pas qu'un service répond ; un ping qui échoue ne prouve pas que le service est la cause. Les commandes `ss`, `Get-NetTCPConnection`, `systemctl status` et `Get-Service` sont plus utiles que le ping pour ce type de diagnostic.

---

## 11. Sécurité, limites et bilan <a id="11-securite-limites-et-bilan"></a>

### 11.1. Ce qui a été mis en place

| Mesure | Mise en œuvre |
|---|---|
| Moindre privilège | Groupe `webadmins`, dossier `/srv/data` et `/mnt/data` en `0770`, `alice` membre du groupe |
| Séparation des comptes | Groupes `GRP-Admins` (Test Admin1) et `GRP-Users` (Test User1, Test User2) dans Active Directory |
| Secrets | Mot de passe Windows dans Ansible Vault, jamais dans `vars.yml` ni dans Git |
| Réseau | Réseau host-only pour le laboratoire de référence, NAT uniquement pour les téléchargements, pfSense et VLAN pour l'environnement RHEL 9 |
| WinRM | HTTP sur le réseau privé, transport NTLM, port 5985 limité au laboratoire |
| Snapshots | Snapshot avant les changements importants et avant chaque incident provoqué |
| Journalisation | `journalctl`, `/var/log/auth.log`, journaux Windows, logs Prometheus |

### 11.2. Les limites

Le projet fonctionne, mais il n'est pas une infrastructure de production. Les principales limites sont les suivantes.

**WinRM en HTTP.** Le transport NTLM sur HTTP convient à un laboratoire isolé, pas à un réseau non maîtrisé. En production, il faudrait HTTPS avec un certificat, ou Kerberos.

**Compte Administrateur.** Ansible se connecte avec `INFRAREADY\Administrateur`. C'est pratique, mais ce n'est pas le moindre privilège. Un compte de service dédié, avec les droits strictement nécessaires, serait préférable.

**Supervision sans TLS.** Prometheus et Grafana sont accessibles en HTTP sur le réseau privé, sans authentification forte. En production, il faudrait un reverse proxy, TLS et une gestion des accès.

**Dashboards importés.** Les deux tableaux de bord viennent de la bibliothèque Grafana. Ils sont utiles pour démarrer, mais ils ne sont pas adaptés à tous les besoins. Un dashboard maison avec les seuils du projet serait plus propre.

**CI limitée au dépôt.** Le runner GitHub hébergé ne peut pas joindre le réseau VMware. La CI valide le dépôt (lint, syntaxe, collections), elle ne l'exécute pas sur les VM. Un runner auto-hébergé serait possible, mais il introduirait un accès supplémentaire à sécuriser.

**Matrice de droits Windows incomplète.** Les groupes existent, mais la matrice détaillée des droits (qui peut administrer quoi, qui peut lire quoi) n'a pas été formalisée au-delà du modèle de base. C'est un axe d'amélioration.

**Pas de pare-feu local Linux.** LINUX01 et OPS01 n'ont pas de pare-feu local dédié ; la segmentation repose sur le réseau host-only et, pour l'environnement RHEL 9, sur le pfSense. Dans un contexte de production, il faudrait ajouter `ufw` ou `firewalld` avec des règles explicites. Le module porte sur l'administration système et non sur la sécurité réseau ; ce point est signalé pour rester honnête sur le périmètre.

**Pas de politique de sauvegarde.** Les snapshots VMware ne remplacent pas une sauvegarde. Le projet ne couvre pas la restauration de fichiers ou de bases.

### 11.3. Ce que nous ferions ensuite

Si le projet continuait, nous commencerions par les points qui ont le plus d'impact :

1. Passer WinRM en HTTPS avec un certificat, et créer un compte de service dédié.
2. Ajouter une alerte sur `up == 0` pour chaque cible Prometheus, et une sonde DNS.
3. Mettre Grafana et Prometheus derrière un reverse proxy avec TLS et authentification.
4. Formaliser la matrice des droits Windows et supprimer les comptes de test inutiles.
5. Ajouter une sauvegarde indépendante des données de `/mnt/data`.
6. Écrire un dashboard Grafana maison, avec les seuils du projet.

### 11.4. Bilan

Le projet remplit ses objectifs principaux. Les trois VM du laboratoire de référence démarrent, le domaine `infraready.test` est opérationnel, le DNS résout les noms du laboratoire, nginx répond, le disque est monté et persistant, les playbooks Linux et Windows s'exécutent sans erreur et sont idempotents, Prometheus collecte les métriques des deux systèmes, Grafana les affiche, une alerte CPU a été testée, le workflow GitHub Actions valide le dépôt et les deux incidents ont été diagnostiqués puis corrigés. La validation RHEL 9 a confirmé que les playbooks et les procédures restent valables sur une seconde distribution.

Ce qui nous a demandé le plus de temps, ce n'est pas l'installation des services, mais les détails de connexion : la bonne carte réseau, le bon DNS, le bon format de ligne `fstab`, le bon transport WinRM. Ce sont rarement les étapes les plus spectaculaires, mais ce sont celles qui font la différence entre une configuration qui marche une fois et une configuration qui se rejoue.

Le projet nous a aussi obligés à documenter au fur et à mesure. Les captures d'écran sont utiles, mais elles ne remplacent pas une explication : une image de `systemctl status` sans contexte ne dit pas pourquoi le service est tombé. C'est la raison pour laquelle ce dossier replace chaque preuve dans son enchaînement.

---

## 12. Annexes <a id="12-annexes"></a>

### 12.1. Inventaire du laboratoire

| Champ | WIN-DC01 | LINUX01 | OPS01 |
|---|---|---|---|
| Nom | WIN-DC01 | LINUX01 | OPS01 |
| OS | Windows Server 2025 | Ubuntu Server 24.04 LTS | Ubuntu Server 24.04 LTS |
| vCPU | 4 | 2 | 4 |
| RAM | 6 Go | 4 Go | 6,8 Go |
| Disque système | 50 Go | 30 Go | 30 Go |
| Disque de données | — | 5 Go (`/dev/sdb`, `/mnt/data`) | — |
| Réseau privé | 192.168.50.10/24 | 192.168.50.20/24 | 192.168.50.30/24 |
| Réseau NAT | DHCP (192.168.28.137) | DHCP | DHCP |
| DNS | 192.168.50.10 | 192.168.50.10 | 192.168.50.10 |
| Rôle | AD DS, DNS | nginx, node_exporter, stockage | Ansible, Prometheus, Grafana |

Ce tableau décrit le laboratoire de référence. L'environnement RHEL 9 de validation (hôte `AdminLinux-OPS01`, derrière pfSense) réutilise le même plan d'adressage mais n'est pas compté dans ces trois VM ; il est décrit en section 1.3.

### 12.2. Commandes utiles

**Réseau et DNS**

```bash
ip -br address
ip route
resolvectl status
getent hosts win-dc01.infraready.test
ping -c 3 192.168.50.10
ss -lntup
```

**Services et journaux Linux**

```bash
systemctl status nginx
systemctl status node_exporter
journalctl -u nginx --no-pager | tail -20
journalctl -u node_exporter --no-pager | tail -20
tail -20 /var/log/apt/history.log
sudo tail -20 /var/log/auth.log
```

**Windows**

```powershell
Get-NetIPAddress -AddressFamily IPv4
Get-Service DNS, WinRM
Get-NetTCPConnection -LocalPort 53, 5985
Resolve-DnsName win-dc01.infraready.test
Get-WinEvent -LogName System -MaxEvents 30
```

**Ansible**

```bash
ansible-galaxy collection install -r requirements.yml
ansible-inventory -i inventory/hosts.yml --graph
ansible -i inventory/hosts.yml linux -m ping
ansible -i inventory/hosts.yml windows -m ansible.windows.win_ping --ask-vault-pass
ansible-playbook -i inventory/hosts.yml playbooks/linux.yml --syntax-check
ansible-playbook -i inventory/hosts.yml playbooks/linux.yml
ansible-playbook -i inventory/hosts.yml playbooks/windows.yml --ask-vault-pass
ansible-playbook -i inventory/hosts.yml playbooks/update.yml
ansible-playbook -i inventory/hosts.yml playbooks/install_exporter.yml --ask-vault-pass
```

**Supervision**

```bash
curl -I http://192.168.50.20:9100/metrics
curl -I http://192.168.50.10:9182/metrics
systemctl status prometheus
systemctl status grafana-server
ss -lntp | grep -E '9090|3000'
```

### 12.3. Fiche de test

| ID | Objectif | Procédure | Résultat | Preuve |
|---|---|---|---|---|
| T-01 | Adressage IP | `ip -br a` / `Get-NetIPAddress` | Adresses conformes au plan | Images 7, 9, 11 |
| T-02 | Connectivité | `ping` entre les VM | 0 % de perte | Image 12 |
| T-03 | DNS | `nslookup` / `Resolve-DnsName` | `192.168.50.10` | Images 20, 112 |
| T-04 | Service web | `curl -I http://192.168.50.20` | HTTP 200 | Image 21 |
| T-05 | Stockage | `df -h /mnt/data` après reboot | Montage persistant | Image 36 |
| T-06 | Permissions | `sudo -u nobody touch` | Permission denied | Image 35 |
| T-07 | Ansible Linux | `ansible-playbook linux.yml` (2 passages) | 10 ok, 0 changed | Image 44 |
| T-08 | Ansible Windows | `win_ping` et playbook Windows | pong, 6 ok, 0 changed | Images 56, 59 |
| T-09 | Supervision | Target health Prometheus | 3 jobs UP (prometheus et les deux exporters) | Image 80 |
| T-10 | Alerte | Test de charge CPU | Pending puis Firing | Images 95, 96 |
| T-11 | CI | Push sur le dépôt | Workflow réussi | Image 100 |
| T-12 | Incident 1 | Arrêt de node_exporter | Cible DOWN puis UP | Images 101 à 106 |
| T-13 | Incident 2 | Arrêt du service DNS | Résolution coupée puis rétablie | Images 107 à 112 |

### 12.4. Arborescence du dépôt

```text
infraready/
├── README.md
├── requirements.yml
├── .ansible-lint
├── .yamllint
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── windows/
│           ├── vars.yml
│           └── vault.yml
├── playbooks/
│   ├── linux.yml
│   ├── windows.yml
│   ├── update.yml
│   ├── install_exporter.yml
│   └── roles/
│       ├── node_exporter/
│       │   ├── handlers/main.yml
│       │   ├── tasks/main.yml
│       │   └── vars/main.yml
│       └── windows_exporter/
│           ├── handlers/main.yml
│           ├── tasks/main.yml
│           ├── templates/config.yaml.j2
│           └── vars/main.yml
├── monitoring/
│   ├── prometheus.yml
│   └── dashboards/
│       ├── README.md
│       ├── 1860-node-exporter-full.json
│       └── 23942-windows-exporter-dashboard-2025.json
├── docs/
│   ├── architecture.md
│   ├── adressage-ip.md
│   ├── securite.md
│   ├── maintenance.md
│   ├── incident.md
│   ├── dossier.md
│   └── images/
└── .github/
    └── workflows/
        └── ansible-validate.yml
```

---

*Fin du dossier technique InfraReady.*
