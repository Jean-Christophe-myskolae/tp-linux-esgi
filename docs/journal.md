# Journal de bord

## Préparation — 29/09 au 01/10/2026

### Actions
- Création des VMs rhel9-lab (ID 200) et ubuntu24-lab (ID 201) sur Proxmox VE 9
- Snapshot `avant-installation` pris sur les deux VMs, disque vierge

### Incidents et corrections
- Disque créé sur un stockage LVM classique (`vg-vmstore-1To`) : snapshots impossibles et allocation non dynamique. Corrigé en déplaçant le disque vers le stockage LVM-thin (`vg-vmstore-1To-thin`).
- Routage désactivé sur l'hôte (`net.ipv4.ip_forward = 0`) malgré une règle MASQUERADE existante : le NAT ne pouvait pas fonctionner. Corrigé via `/etc/sysctl.d/99-ip-forward.conf`.

### Choix et écarts
- Réseau NAT : `vmbr1` (192.168.2.0/24), le pont `vmbr0` étant interdit par le sujet
- Réseau interne : `vmbr2` (OVS), VLAN 42, 10.42.0.0/24
- RHEL 9 remplacé par Rocky Linux 9 (clone compatible autorisé)
- Le schéma LVM imposé totalise 43 Gio pour des disques de 30/25 Gio : 25Gio

### Temps passé
- 1h30

## Partie B.1 — Justification du partitionnement

**1. Pourquoi séparer /var et /var/log de / ?**
Les journaux grossissent sans limite. S'ils sont sur /, un disque plein bloque tout le système (services qui plantent, connexion impossible). Exemple : une application en erreur qui écrit en boucle dans ses logs, ou une attaque par force brute SSH qui remplit /var/log/auth.log. Avec /var/log séparé, seul ce volume sature et le système continue de fonctionner.

**2. Pourquoi /boot reste une partition classique ?**
Au démarrage, GRUB doit lire /boot pour charger le noyau avant que LVM soit actif. Une partition simple reste lisible même si LVM pose problème, ce qui permet de démarrer et de réparer.

**3. Pourquoi 2 Gio de swap pour 2 Gio de RAM ?**
Sans swap, une saturation mémoire fait tuer des processus par le noyau (OOM killer). Avec 8 Gio, la machine ne plante pas mais devient inutilisable car le disque est beaucoup plus lent que la RAM. 1x la RAM est un filet de sécurité raisonnable pour absorber un pic.

**4. Pourquoi ne pas allouer tout le disque ?**
Avec LVM, agrandir un volume est simple et se fait à chaud, alors que réduire est risqué (impossible en XFS). L'espace libre dans vg_sys permet d'agrandir le volume qui en aura besoin, au moment voulu (TP 2).

**5. Quel volume agrandir en premier et comment ?**
lv_log (/var/log), le plus exposé à la saturation. Opération : `lvextend` pour agrandir le volume logique puis agrandissement du système de fichiers (`xfs_growfs` ou `resize2fs`), ou directement `lvextend -r`, sans démontage donc sans coupure de service.

## Séance 1 — 02/10/2026

### Partie A — Préparation de l'hyperviseur et des machines
- rhel9-lab (ID 200) : seconde interface réseau absente au moment de l'installation (perdue lors d'un retour à l'instantané initial, pris avant son ajout). Ajoutée à chaud : `net1` sur `vmbr2`, VLAN 42, modèle VirtIO.
- Interfaces finales : `net0` → `vmbr1` (NAT), `net1` → `vmbr2` VLAN 42 (lab-interne).
- Intérêt de l'instantané `avant-installation` : revenir à une machine vierge, configurée mais jamais installée, en cas d'erreur d'installation. Il a servi pendant cette séance (voir incidents).

### Partie B — Partitionnement LVM (rhel9-lab, Rocky Linux 9.8)
Schéma réalisé avec l'installateur Anaconda, en mode personnalisé, schéma LVM :

| Volume | Taille | Point de montage | Système de fichiers |
|---|---|---|---|
| sda1 | 1 Gio | /boot | xfs |
| vg_sys-lv_root | 20 Gio | / | xfs |
| vg_sys-lv_var | 10 Gio | /var | xfs |
| vg_sys-lv_log | 5 Gio | /var/log | xfs |
| vg_sys-lv_home | 5 Gio | /home | xfs |
| vg_sys-lv_swap | 2 Gio | swap | swap |
| espace libre vg_sys | ~7 Gio | — | réservé au TP 2 |

- Table de partitions MSDOS, démarrage BIOS (SeaBIOS) : pas de partition EFI nécessaire.
- Groupe de volumes `vg_sys` configuré avec la politique « Aussi grand que possible », pour que l'espace non alloué reste à l'intérieur du groupe.
- Schéma contrôlé par l'enseignant avant écriture : [à compléter : oui / non, heure]

### Partie C — Paramètres définis pendant l'installation
- Profil : Minimal Install, sans ensemble facultatif
- Nom d'hôte : `rhel9-lab`
- Fuseau horaire : Europe/Paris, synchronisation réseau activée
- Clavier : français
- Réseau :
  - `ens18` (NAT) : 192.168.2.20/24, passerelle 192.168.2.1, DNS 1.1.1.1
  - `ens19` (lab-interne) : 10.42.0.20/24, sans passerelle (réseau isolé, la route par défaut passe uniquement par ens18)
- Compte root : désactivé dès l'installation
- Compte nominatif : `jcandelaria`, administrateur (membre du groupe wheel)

### Incidents et corrections
1. **Installation lancée avec le partitionnement automatique.** Seule la création de l'utilisateur avait été configurée ; Anaconda a appliqué ses valeurs par défaut (partitionnement automatique, pas de nom d'hôte ni de réseau). Correction : arrêt de la VM, retour à l'instantané `avant-installation`, réinstallation complète.
2. **Disque trop petit pour le schéma imposé.** Le schéma totalise 43 Gio pour un disque de 30 Gio. Correction : disque agrandi à chaud de 30 à 50 Gio depuis Proxmox (Disk Action → Resize). Les tailles des volumes du sujet sont conservées, avec ~7 Gio libres dans vg_sys.
3. **Nouvelle taille non prise en compte par l'installateur.** Malgré une nouvelle analyse des disques, Anaconda n'affichait que 30 Gio disponibles sur 50 (table de partitions lue avant l'agrandissement). Correction : redémarrage de la VM, l'installateur relit le disque à 50 Gio.
4. **Volumes répartis dans deux groupes de volumes.** Le groupe créé par défaut s'appelait `rlm`, avec une politique de taille automatique : l'espace libre restait hors du groupe, et après renommage seul `lv_root` était dans `vg_sys`, les autres volumes étant restés dans `rlm`. Correction : groupe renommé `vg_sys`, politique « Aussi grand que possible », puis chaque volume rattaché manuellement à `vg_sys`. Vérification dans le résumé des changements : un seul groupe `vg_sys` contenant tous les volumes.
5. **Point d'attention :** l'instantané `avant-installation` a été pris avec un disque de 30 Gio. Un retour à cet instantané pourrait ramener le disque à son ancienne taille. Un nouvel instantané sera pris après l'installation.

### Résultat
- Installation de rhel9-lab : [à compléter : terminée / en cours]
- Vérifications B.3 (`lsblk -f`, `df -hT`, `swapon --show`, `findmnt --target /var/log`, `vgs`) : [à compléter, sorties dans tp01/preuves/]
- ubuntu24-lab : [à compléter : état]

### Temps passé
- [à compléter]

- ## Travail hors séance — 04/10/2026

### Vérifications après installation de rhel9-lab
- Connexion SSH depuis l'hôte Proxmox (192.168.2.1) vers rhel9-lab (192.168.2.20) avec le compte nominatif : OK
- Réseau : `ens18` 192.168.2.20/24 et `ens19` 10.42.0.20/24, toutes deux UP. Une seule route par défaut, via 192.168.2.1 sur ens18
- Partitionnement conforme au schéma (`lsblk -f`, `df -hT`, `swapon --show`, `findmnt`, `vgs`, `lvs`) : vg_sys avec 1 PV, 5 LV et 7 Gio libres, swap de 2 Gio actif, /var/log monté depuis vg_sys-lv_log
- Système : Rocky Linux 9.8 (Blue Onyx), aucune unité en échec (`systemctl --failed`)
- Locale fr_FR.UTF-8, clavier fr-oss, fuseau Europe/Paris, horloge synchronisée (NTP actif)

### Mises à jour (C.4)
- `sudo dnf -y update` : 104 paquets installés ou mis à jour, dont un nouveau noyau
- `sudo dnf -y install epel-release` : dépôt EPEL activé (visible dans `dnf repolist`)
- Redémarrage sur le nouveau noyau (5.14.0-687.10.1 → 5.14.0-687.54.1), puis `systemctl --failed` : aucune unité en échec
- Remarque : l'installation a été réalisée en séance le 02/10 (entrée n°1 de `dnf history`), les mises à jour ont été appliquées le 04/10 hors séance (entrées n°2 et 3).

### Incidents et corrections
1. **Nom d'hôte resté à `localhost`** après l'installation : le nom saisi dans l'installateur n'a pas été appliqué. Correction : `sudo hostnamectl set-hostname`.
2. **Faute de frappe dans le nom d'hôte** : `rhe19-lab` (chiffre 1) au lieu de `rhel9-lab` (lettre L), repérée dans le prompt. Corrigée avec la même commande et vérifiée avec `hostnamectl`.
3. **Résolution DNS impossible** : `ping 1.1.1.1` répondait mais `ping rockylinux.org` échouait (« Nom ou service inconnu »). Diagnostic : NAT fonctionnel, aucun serveur DNS configuré. Correction : `nmcli connection modify ens18 ipv4.dns "1.1.1.1 9.9.9.9"` puis `nmcli connection up ens18`, vérification dans `/etc/resolv.conf`.
4. **Image d'installation toujours attachée** à la VM après l'installation (`sr0` visible dans `lsblk`) : risque de redémarrer sur l'installateur. Correction dans Proxmox : lecteur CD/DVD passé à « Do not use any media ».

### Instantané
- `apres-installation` : pris le 04/10, VM éteinte, après mises à jour et redémarrage. Point de retour avant le durcissement de la partie D.

### Temps passé
- 1h30
