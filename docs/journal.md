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
-

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
