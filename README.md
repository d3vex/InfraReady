# InfraReady

InfraReady est le projet fil rouge du module Administration Linux et Windows, en Bachelor 3 Cybersécurité. Le point de départ est une entreprise fictive dont les serveurs Linux et Windows sont en partie administrés à la main. Le dépôt rassemble l'inventaire, les playbooks, les configurations de supervision et la documentation nécessaires pour rejouer l'installation et les opérations courantes.

Le projet est public : [github.com/d3vex/infraready](https://github.com/d3vex/infraready).

## Le besoin

Une configuration réalisée à la main fonctionne une fois, puis devient difficile à rejouer, à auditer et à transmettre. Les procédures changent d'une machine à l'autre, les mises à jour sont suivies de façon irrégulière et un incident demande souvent de reconstruire le contexte avant de pouvoir agir.

Le laboratoire doit servir de base reproductible. Les configurations doivent pouvoir être rejouées, les opérations documentées et l'état des serveurs visible depuis un tableau de bord. C'est le cadre du projet : installer, configurer, automatiser, superviser et documenter une infrastructure Linux et Windows.

## Architecture du laboratoire

Le laboratoire de référence tient sur trois machines virtuelles VMware, sur le réseau host-only `192.168.50.0/24`. Une carte NAT sert uniquement aux téléchargements et aux mises à jour.

| Machine | Système | Rôle | Adresse |
|---|---|---|---|
| WIN-DC01 | Windows Server 2025 | AD DS, DNS, comptes, groupes, journaux | 192.168.50.10/24 |
| LINUX01 | Ubuntu Server 24.04 LTS | nginx, comptes et permissions, stockage, node_exporter | 192.168.50.20/24 |
| OPS01 | Ubuntu Server 24.04 LTS | Ansible, Prometheus, Grafana | 192.168.50.30/24 |

Le domaine Active Directory est `infraready.test`, avec `INFRAREADY` comme nom NetBIOS. Le DNS interne est porté par WIN-DC01. Les clients n'utilisent que ce résolveur ; les requêtes vers les noms externes sont transférées par le serveur DNS.

Un second environnement RHEL 9, derrière un pfSense et des VLAN, a servi à Loan pour les tests multi-distribution. Il réutilise le même plan d'adressage afin de comparer les procédures. Les captures RHEL 9 du dossier sont prises avec `root` : cet environnement est un conteneur LXC, pas une VM, et `root` y est le compte par défaut. Un compte `administrateur` nominatif a bien été créé et ajouté au groupe `sudo`. Dans un cas réel, les commandes autorisées seraient limitées dans `sudoers`.

## Comment le laboratoire fonctionne

Le réseau interne est isolé du réseau de l'établissement. Les adresses sont fixes, le DHCP de VMnet2 est désactivé et chaque VM possède une seconde carte pour la sortie NAT. Cette séparation permet de tester le DNS interne sans dépendre d'un résolveur externe.

Ansible est installé sur OPS01. L'inventaire décrit les groupes `linux` et `windows`. Les playbooks Linux utilisent une connexion SSH par clé ed25519 ; les playbooks Windows passent par WinRM et NTLM. Les variables de connexion Windows sont séparées, et le mot de passe est stocké dans un fichier Ansible Vault chiffré. Les rôles installent et supervisent les exporters, et un playbook `update.yml` applique les mises à jour selon la famille de distribution.

Prometheus collecte les métriques de LINUX01 et de WIN-DC01 via `node_exporter` et `windows_exporter`. Grafana utilise Prometheus comme source de données et affiche deux dashboards de la bibliothèque Grafana : Node Exporter Full (ID 1860) et Windows Exporter Dashboard 2025 (ID 23942). Les JSON sont versionnés dans `monitoring/dashboards/`.

GitHub Actions vérifie le dépôt à chaque push et à chaque pull request. Le workflow installe `ansible-core`, `ansible-lint` et `yamllint`, installe les collections déclarées dans `requirements.yml`, puis lance les contrôles de lint et de configuration. Il ne se connecte pas aux VM : un runner hébergé ne peut pas joindre le réseau VMware privé.

## Ce qui est en place

Le domaine `infraready.test` est opérationnel, avec AD DS et DNS sur WIN-DC01. Les groupes `GRP-Admins` et `GRP-Users` et les comptes de test sont créés. Les journaux Windows et Linux sont consultables.

LINUX01 sert la page nginx en HTTP 200, le groupe `webadmins` et l'utilisateur `alice` disposent des droits prévus sur `/srv/data` et `/mnt/data`, et un disque de 5 Go est formaté en ext4 puis monté par UUID dans `/etc/fstab`. Le montage est toujours actif après redémarrage et les permissions ont été testées dans les deux sens.

Les playbooks Linux et Windows passent au second passage en `changed=0`. Les rôles `node_exporter` et `windows_exporter` sont idempotents. Prometheus reçoit les métriques des deux hôtes, Grafana les affiche et une alerte CPU a été testée au-delà de 70 % pendant cinq minutes.

Deux incidents ont été provoqués et documentés : l'arrêt de `node_exporter` sur LINUX01 et l'arrêt du service DNS sur WIN-DC01. Chaque rapport suit la même méthode : symptôme, tests et journaux, cause, correction, vérification du retour à la normale.

## Démarrage rapide

Prérequis : VMware Workstation Pro, les trois VM du laboratoire, Ansible 2.16 ou version compatible sur OPS01, une clé SSH pour LINUX01 et WinRM configuré sur WIN-DC01.

```bash
git clone https://github.com/d3vex/infraready.git
cd infraready
ansible-galaxy collection install -r requirements.yml

ansible-inventory -i inventory/hosts.yml --graph
ansible -i inventory/hosts.yml linux -m ping
ansible -i inventory/hosts.yml windows -m ansible.windows.win_ping --ask-vault-pass

ansible-playbook -i inventory/hosts.yml playbooks/linux.yml
ansible-playbook -i inventory/hosts.yml playbooks/windows.yml --ask-vault-pass
ansible-playbook -i inventory/hosts.yml playbooks/install_exporter.yml --ask-vault-pass
ansible-playbook -i inventory/hosts.yml playbooks/update.yml
```

Les adresses de l'inventaire sont celles du laboratoire. Il faut les adapter avant de rejouer les playbooks sur un autre réseau. Le mot de passe du vault n'est pas dans le dépôt ; il est fourni au moment de l'exécution avec `--ask-vault-pass`.

## Structure du dépôt

```text
infraready/
├── README.md
├── requirements.yml
├── inventory/
├── playbooks/
│   └── roles/
├── monitoring/
│   ├── prometheus.yml
│   └── dashboards/
├── docs/
└── .github/workflows/
```

## Documentation

Le dossier technique complet est dans [`docs/dossier.md`](docs/dossier.md), avec sa version PDF [`docs/dossier.pdf`](docs/dossier.pdf). Les fiches séparées couvrent l'architecture, l'adressage IP, la sécurité, la maintenance et les incidents :

- `docs/architecture.md`
- `docs/adressage-ip.md`
- `docs/securite.md`
- `docs/maintenance.md`
- `docs/incident.md`

## Limites

Le projet est un laboratoire, pas une infrastructure de production. WinRM reste en HTTP sur le réseau privé, Ansible se connecte avec le compte `Administrateur` du domaine, Prometheus et Grafana n'ont pas de TLS, les dashboards viennent de la bibliothèque Grafana et il n'existe pas de politique de sauvegarde indépendante. Ces points sont détaillés dans le dossier technique.

## Auteurs

Alexandre BOZZI et Loan Mata, Bachelor 3 Cybersécurité, Ynov, 2026.
