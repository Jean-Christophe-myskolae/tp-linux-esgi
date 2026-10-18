# Installation de rhel9-lab et ubuntu24-lab

## 1. Périmètre
| Machine | Distribution | Version | Noyau |
|---|---|---|---|
| rhel9-lab | Rocky Linux (clone compatible RHEL 9) | 9.8 (Blue Onyx) | 5.14.0-687.54.1.el9_8.x86_64 |
| ubuntu24-lab | Ubuntu Server | 24.04.5 LTS (Noble Numbat) | 6.8.0-147-generic |

- Rocky Linux est utilisé à la place de RHEL 9 (clone compatible autorisé, sans abonnement Red Hat).
- Ubuntu a été installé depuis l'image 24.04.3 ; le système est en 24.04.5 après mises à jour.

## 2. Ressources des machines virtuelles

Hyperviseur : Proxmox VE 9, stockage LVM-thin (`vg-vmstore-1To-thin`) pour l'allocation dynamique et les instantanés.

| Paramètre | rhel9-lab | ubuntu24-lab |
|---|---|---|
| ID Proxmox | 202 | 201 |
| vCPU | 2 | 2 |
| Mémoire | 2048 Mio | 2048 Mio |
| Disque | 50 Gio, SCSI (VirtIO SCSI single), alloué dynamiquement | idem |
| Micrologiciel | SeaBIOS | SeaBIOS |
| Interface 1 (NAT) | `ens18` → `vmbr1`, 192.168.2.20/24, passerelle 192.168.2.1 | `ens18` → `vmbr1`, 192.168.2.21/24, passerelle 192.168.2.1 |
| Interface 2 (lab-interne) | `ens19` → `vmbr2` VLAN 42, 10.42.0.20/24, sans passerelle | `ens19` → `vmbr2` VLAN 42, 10.42.0.21/24, sans passerelle |
| DNS | 1.1.1.1, 9.9.9.9 | 1.1.1.1, 9.9.9.9 |

### Configuration réseau de l'hôte Proxmox
- **NAT** : `vmbr1`, pont sans port physique, IP de l'hôte 192.168.2.1/24.
  - Traduction d'adresses : `-A POSTROUTING -s 192.168.2.0/24 -o vmbr0 -j MASQUERADE` dans `/etc/iptables/rules.v4`
  - Routage activé : `net.ipv4.ip_forward=1` dans `/etc/sysctl.d/99-ip-forward.conf`
- **lab-interne** : pont Open vSwitch `vmbr2` sans port physique, VLAN 42 dédié, réseau 10.42.0.0/24.
- Le pont `vmbr0` (relié au réseau local) n'est pas utilisé : le mode pont est interdit par le sujet.
- **Point de vigilance** : l'ordre des interfaces dans Proxmox détermine leur nom dans la VM (`net0` → `ens18`, `net1` → `ens19`). Vérifier l'adresse MAC dans l'installateur.

### Instantanés
| Machine | Instantanés |
|---|---|
| rhel9-lab | `avant-installation`, `apres-installation` |
| ubuntu24-lab | `avant-installation`, `installation-brute`, `apres-installation` |

## 3. Partitionnement appliqué

### rhel9-lab — table MSDOS, xfs
| Volume | Taille obtenue | Point de montage | Système de fichiers |
|---|---|---|---|
| /dev/sda1 | 1 Gio | /boot | xfs |
| vg_sys-lv_root | 20 Gio | / | xfs |
| vg_sys-lv_var | 10 Gio | /var | xfs |
| vg_sys-lv_log | 5 Gio | /var/log | xfs |
| vg_sys-lv_home | 5 Gio | /home | xfs |
| vg_sys-lv_swap | 2 Gio | swap | swap |
| Espace libre de vg_sys | 7 Gio | — | réservé au TP 2 |

### ubuntu24-lab — table GPT, ext4
| Volume | Taille obtenue | Point de montage | Système de fichiers |
|---|---|---|---|
| /dev/sda1 | 1 Mio | — | BIOS grub spacer |
| /dev/sda2 | 1 Gio | /boot | ext4 |
| vg_sys-lv_root | 20 Gio | / | ext4 |
| vg_sys-lv_var | 10 Gio | /var | ext4 |
| vg_sys-lv_log | 5 Gio | /var/log | ext4 |
| vg_sys-lv_home | 5 Gio | /home | ext4 |
| vg_sys-lv_swap | 2 Gio | swap | swap |
| Espace libre de vg_sys | 7 Gio | — | réservé au TP 2 |

**Écart avec le sujet** : disques de 50 Gio au lieu de 30 et 25 Gio. Le schéma imposé totalise 43 Gio. Les tailles des volumes du sujet sont conservées, la marge libre est de 7 Gio.

**Démarrage** : BIOS (SeaBIOS) sur les deux machines, donc pas de partition EFI. Ubuntu utilise une table GPT et ajoute une partition « BIOS grub spacer » de 1 Mio nécessaire à GRUB ; Rocky utilise une table MSDOS qui n'en a pas besoin.

**Points de vigilance**
- Anaconda (Rocky) : renommer le groupe de volumes `vg_sys` et choisir la politique de taille « Aussi grand que possible », sinon l'espace non alloué reste hors du groupe.
- Subiquity (Ubuntu) : choisir « Custom storage layout ». L'option par défaut « Use an entire disk » crée un schéma non conforme.

Justification des choix : voir `docs/journal.md`, partie B.1.

## 4. Installation

| Paramètre | rhel9-lab | ubuntu24-lab |
|---|---|---|
| Installateur | Anaconda (graphique) | Subiquity (texte) |
| Profil | Minimal Install | Ubuntu Server (minimized) |
| Nom d'hôte | rhel9-lab | ubuntu24-lab |
| Locale | fr_FR.UTF-8 | fr_FR.UTF-8 |
| Clavier | fr-oss | fr |
| Fuseau horaire | Europe/Paris | Europe/Paris (corrigé après installation) |
| Synchronisation de l'heure | chronyd | systemd-timesyncd (ntp.ubuntu.com) |
| Compte root | désactivé à l'installation | verrouillé par défaut |
| Compte administrateur | `jcandelariasureta`, groupe wheel | `jcandelariasureta`, groupe sudo |
| Serveur SSH | installé par défaut | coché à l'installation (OpenSSH server) |

**Corrections nécessaires après installation**

rhel9-lab : nom d'hôte et DNS saisis dans Anaconda non appliqués.
```bash
sudo hostnamectl set-hostname rhel9-lab
sudo nmcli connection modify ens18 ipv4.dns "1.1.1.1 9.9.9.9"
sudo nmcli connection up ens18
```

ubuntu24-lab : fuseau horaire non proposé par l'installateur (UTC par défaut).
```bash
sudo timedatectl set-timezone Europe/Paris
```

Sur les deux machines : retirer l'image du lecteur CD/DVD après installation (Proxmox → Hardware → CD/DVD Drive → Do not use any media).

## 5. Mises à jour

### rhel9-lab
```bash
sudo dnf -y update
sudo dnf -y install epel-release
sudo reboot
```
| ID | Commande | Date et heure | Paquets |
|---|---|---|---|
| 1 | installation | 2026-10-02 18:45 | 335 |
| 2 | -y update | 2026-10-04 05:33 | 104 |
| 3 | -y install epel-release | 2026-10-04 06:00 | 1 |

Dépôts actifs : appstream, baseos, extras, epel. Nouveau noyau installé, redémarrage effectué.

### ubuntu24-lab
```bash
sudo apt update
sudo apt -y full-upgrade
sudo apt -y autoremove
sudo reboot
```
Extrait de `/var/log/apt/history.log` :
```
Start-Date: 2026-10-18  13:02:34   install grub-pc            (installateur, heure UTC)
Start-Date: 2026-10-18  13:03:26   install linux-generic      (installateur, heure UTC)
Start-Date: 2026-10-18  13:04:17   install openssh-server     (installateur, heure UTC)
Start-Date: 2026-10-18  13:06:52   unattended-upgrades        (automatique, heure UTC)
Start-Date: 2026-10-18  17:52:51   apt -y full-upgrade        (heure de Paris)
Start-Date: 2026-10-18  17:55:46   apt -y autoremove          (heure de Paris)
```
69 paquets mis à jour, 2 nouveaux, pas de nouveau noyau. Aucune unité en échec après redémarrage sur les deux machines.

Sorties complètes : `tp01/preuves/rhel9-lab.txt` et `tp01/preuves/ubuntu24-lab.txt`.
