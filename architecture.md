# Architecture

## Vue d'ensemble

L'infrastructure InfraReady est un laboratoire de trois machines virtuelles hébergées dans VMware Workstation sur un même poste. Elle reproduit un petit système d'information d'entreprise : un annuaire Windows, un serveur Linux applicatif et un serveur d'administration qui automatise et surveille le tout.

| Machine | Système | Rôle |
|---|---|---|
| WIN-DC01 | Windows Server 2025 | Contrôleur de domaine : Active Directory (AD DS), DNS, comptes et groupes, journaux Windows |
| LINUX01 | Ubuntu Server 24.04 LTS | Serveur Linux : service web nginx, stockage, comptes, permissions, mises à jour, journaux |
| OPS01 | Ubuntu Server 24.04 LTS | Poste d'administration : contrôleur Ansible, Git, Ansible Vault, puis Prometheus et Grafana |

Le domaine Active Directory du laboratoire est `infraready.test`. WIN-DC01 est aussi le serveur DNS du laboratoire : LINUX01 et OPS01 lui envoient les requêtes concernant `infraready.test`.

![Schéma réseau](screenshots/architecture.png)

## Principes de conception

**Deux réseaux séparés.** Chaque VM possède deux cartes réseau :

- **VMnet2 (host-only, 192.168.50.0/24)** : réseau privé du projet, isolé du réseau de l'établissement. Toutes les communications entre les VM passent par là : administration Ansible, DNS, futures métriques.
- **VMnet8 (NAT, 192.168.28.0/24)** : sert uniquement à sortir sur Internet pour installer des paquets et appliquer les mises à jour. Aucune passerelle n'est configurée sur VMnet2.

**Un contrôleur unique pour l'automatisation.** OPS01 pilote les deux autres machines et ne reçoit aucune configuration en retour :

- vers LINUX01, par **SSH** (port 22) avec authentification par clé ;
- vers WIN-DC01, par **WinRM** (port 5985, authentification NTLM).

Le mot de passe de l'administrateur Windows est chiffré avec Ansible Vault. Le mot de passe `sudo` est saisi à l'exécution et n'est jamais stocké.

**Un DNS central.** WIN-DC01 héberge la zone `infraready.test`. Sur Ubuntu, seules les requêtes pour `infraready.test` sont envoyées au DC (`~infraready.test` sur la carte du lab) ; le reste passe par la carte NAT.

**Supervision prévue.** Prometheus, installé sur OPS01, interrogera Node Exporter sur LINUX01 (port 9100) et Windows Exporter sur WIN-DC01 (port 9182). Grafana lira Prometheus pour afficher le tableau de bord. Ces éléments sont en cours de mise en place.

## Choix et limites

- Les trois rôles sont répartis sur trois VM, conformément à l'architecture de référence du sujet : aucune fusion de rôles n'a été nécessaire.
- WinRM en HTTP avec NTLM convient à un réseau de laboratoire isolé ; en production, on utiliserait HTTPS (port 5986).
- Le pipeline GitHub Actions se limite à la validation du dépôt : un runner hébergé par GitHub ne peut pas atteindre le réseau VMware privé.
