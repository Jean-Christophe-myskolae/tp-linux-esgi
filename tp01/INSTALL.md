# Installation de rhel9-lab et ubuntu24-lab

## 1. Périmètre
| Machine | Distribution | Version | Noyau |
|---|---|---|---|
| rhel9-lab | Rocky Linux (clone compatible RHEL 9) | 9.8 (Blue Onyx) | 5.14.0-687.54.1.el9_8.x86_64 |
| ubuntu24-lab | Ubuntu Server | 24.04 LTS | [à compléter] |

Rocky Linux est utilisé à la place de RHEL 9 (clone compatible autorisé par le sujet, sans abonnement Red Hat).

## 2. Ressources des machines virtuelles

Hyperviseur : Proxmox VE 9, stockage LVM-thin (`vg-vmstore-1To-thin`) pour l'allocation dynamique et les instantanés.

| Paramètre | rhel9-lab | ubuntu24-lab |
|---|---|---|
| ID Proxmox | 200 | 201 |
| vCPU | 2 | 2 |
| Mémoire | 2048 Mio | 2048 Mio |
| Disque | 50 Gio, SCSI (VirtIO SCSI single), alloué dynamiquement | 50 Gio, idem |
| Micrologiciel | SeaBIOS | SeaBIOS |
| Interface 1 (NAT) | `ens18` → `vmbr1`, 192.168.2.20/24, passerelle 192.168.2.1 | `vmbr1`, 192.168.2.21/24, passerelle 192.168.2.1 |
| Interface 2 (lab-interne) | `ens19` → `vmbr2` VLAN 42, 10.42.0.20/24, sans passerelle | `vmbr2` VLAN 42, 10.42.0.21/24, sans passerelle |
| DNS | 1.1.1.1, 9.9.9.9 | 1.1.1.1, 9.9.9.9 |

### Configuration réseau de l'hôte Proxmox
- **NAT** : `vmbr1`, pont sans port physique, IP de l'hôte 192.168.2.1/24.
  - Traduction d'adresses : règle `-A POSTROUTING -s 192.168.2.0/24 -o vmbr0 -j MASQUERADE` dans `/etc/iptables/rules.v4`
  - Routage activé : `net.ipv4.ip_forward=1` dans `/etc/sysctl.d/99-ip-forward.conf`
- **lab-interne** : pont Open vSwitch `vmbr2` sans port physique, VLAN 42 dédié, réseau 10.42.0.0/24.
- Le pont `vmbr0` (relié au réseau local) n'est pas utilisé : le mode pont est interdit par le sujet.

### Instantanés
- `avant-installation` : VM créée, disque vierge
- `apres-installation` : système installé et à jour

## 3. Partitionnement appliqué

Table de partitions MSDOS (démarrage BIOS, pas de partition EFI).

| Volume | Taille obtenue | Point de montage | Système de fichiers |
|---|---|---|---|
| /dev/sda1 | 1 Gio | /boot | xfs |
| vg_sys-lv_root | 20 Gio | / | xfs |
| vg_sys-lv_var | 10 Gio | /var | xfs |
| vg_sys-lv_log | 5 Gio | /var/log | xfs |
| vg_sys-lv_home | 5 Gio | /home | xfs |
| vg_sys-lv_swap | 2 Gio | swap | swap |
| Espace libre de vg_sys | 7 Gio | — | réservé à l'agrandissement (TP 2) |

**Écart avec le sujet** : disque de 50 Gio au lieu de 30 Gio. Le schéma imposé totalise 43 Gio et ne tient pas sur 30 Gio. Les tailles des volumes du sujet sont conservées, la marge libre est de 7 Gio.

**Point de vigilance (Anaconda)** : le groupe de volumes doit être renommé `vg_sys` avec la politique de taille « Aussi grand que possible ». Avec la politique automatique, l'espace non alloué reste hors du groupe et `vgs` affiche 0 Gio libre.

Justification des choix : voir `docs/journal.md`, partie B.1.

## 4. Installation

| Paramètre | rhel9-lab |
|---|---|
| Image | Rocky Linux 9.8 Minimal (empreinte SHA-256 vérifiée) |
| Profil | Minimal Install, aucun ensemble facultatif |
| Nom d'hôte | rhel9-lab |
| Locale | fr_FR.UTF-8 |
| Clavier | fr-oss |
| Fuseau horaire | Europe/Paris |
| Horloge | synchronisée par NTP (chronyd) |
| Compte root | désactivé dès l'installation |
| Compte administrateur | `jcandelariasureta`, membre du groupe wheel |

**À vérifier après installation** : le nom d'hôte et le DNS saisis dans Anaconda n'ont pas été appliqués. Commandes de correction :
```bash
sudo hostnamectl set-hostname rhel9-lab
sudo nmcli connection modify ens18 ipv4.dns "1.1.1.1 9.9.9.9"
sudo nmcli connection up ens18
```

Après installation, retirer l'image du lecteur CD/DVD (Proxmox → Hardware → CD/DVD Drive → Do not use any media).

## 5. Mises à jour

```bash
sudo dnf -y update
sudo dnf -y install epel-release
sudo reboot
sudo dnf history | head -20
sudo dnf repolist
```

Extrait de l'historique :
