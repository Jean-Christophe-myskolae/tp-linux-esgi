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
