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

## 6. Comptes et accès distant

### Comptes
| Compte | Nom complet | UID | rhel9-lab | ubuntu24-lab |
|---|---|---|---|---|
| root | — | 0 | verrouillé (`LK`) | verrouillé (`L`) |
| jcandelariasureta | Jean-Christophe Candelaria Sureta | 1000 | groupe `wheel` | groupe `sudo` |
| mmbaya | Mbaya | 1001 | groupe `wheel` | groupe `sudo` |
| asarambounou | Adama Sarambounou | 1002 | groupe `wheel` | groupe `sudo` |

### Délégation
- Uniquement par le groupe d'administration de la distribution : `wheel` sur Rocky, `sudo` sur Ubuntu. Aucun compte avec un UID 0 autre que root.
- `sudo` exige le mot de passe de l'utilisateur (pas de `NOPASSWD`).
- Un ajout à un groupe n'est pris en compte qu'après reconnexion.

```bash
# Rocky
sudo passwd -l root
sudo useradd -m -c "Nom complet" compte
sudo passwd compte
sudo usermod -aG wheel compte

# Ubuntu
sudo passwd -l root
sudo adduser compte
sudo usermod -aG sudo compte
```

### Accès distant
- Poste d'administration : hôte Proxmox (192.168.2.1).
- Clé ed25519 protégée par une phrase de passe, déployée sur le compte `jcandelariasureta` des deux serveurs :
```bash
ssh-keygen -t ed25519 -a 100 -C "jcandelariasureta@lab" -f ~/.ssh/id_ed25519_lab
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub jcandelariasureta@192.168.2.20
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub jcandelariasureta@192.168.2.21
```
- Configuration sshd retenue, dans `/etc/ssh/sshd_config.d/00-tp01-durcissement.conf` :
```
PermitRootLogin no
```
  - fichier séparé : retour arrière par simple suppression ;
  - préfixe `00-` : sshd retient la première valeur lue, dans l'ordre alphabétique des fichiers.
- `PasswordAuthentication yes` conservé volontairement (désactivation prévue au TP 6, une fois les clés généralisées).
- Application : `sudo sshd -t` puis `sudo systemctl reload sshd` (Rocky) ou `sudo systemctl reload ssh` (Ubuntu).

### Vérification
```bash
sudo passwd -S root
sudo -l
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'   # attendu : permitrootlogin no
ssh -o BatchMode=yes root@192.168.2.20 true                        # attendu : Permission denied, code 255
ssh -i ~/.ssh/id_ed25519_lab jcandelariasureta@192.168.2.20 'echo ok'
```
Preuves : `tp01/preuves/partieD-rhel9-lab.txt`, `partieD-ubuntu24-lab.txt`, `preuves-acces-distant.txt`.

## 7. Pare-feu

| | rhel9-lab | ubuntu24-lab |
|---|---|---|
| Outil | firewalld | ufw |
| État après installation | actif (ssh, cockpit, dhcpv6-client ouverts) | installé mais inactif |
| Règles appliquées | ssh uniquement (cockpit et dhcpv6-client retirés) | OpenSSH (22/tcp) uniquement, IPv4 et IPv6 |
| Politique par défaut | zone `public` : tout ce qui n'est pas listé est rejeté | deny en entrée, allow en sortie |
| Persistance | `--permanent` + `--reload` | activé au démarrage |

```bash
# Rocky
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --permanent --remove-service=dhcpv6-client
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

# Ubuntu : autoriser SSH AVANT d'activer
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

### Méthode de vérification
1. État complet du pare-feu (pas une seule règle) : `firewall-cmd --list-all` / `ufw status verbose`.
2. **Nouvelle** connexion SSH depuis le poste d'administration après activation (une session déjà ouverte ne prouve rien).
3. Test d'un port fermé : `timeout 3 bash -c '</dev/tcp/192.168.2.20/9090'` → rejeté.
4. Connexion depuis une autre machine : ubuntu24-lab → rhel9-lab via lab-interne (10.42.0.20).

## 8. Écarts entre les deux distributions

| Opération | RHEL 9 (Rocky 9.8) | Ubuntu 24.04 | Raison |
|---|---|---|---|
| Installateur | Anaconda, graphique | Subiquity, texte | outils propres à chaque distribution |
| Profil minimal | Minimal Install | Ubuntu Server (minimized) | — |
| Table de partitions | MSDOS | GPT + partition « BIOS grub spacer » de 1 Mio | en BIOS sur GPT, GRUB a besoin d'une zone dédiée |
| Système de fichiers par défaut | xfs | ext4 | choix par défaut de chaque distribution |
| Agrandir un système de fichiers | `xfs_growfs` | `resize2fs` | outils propres à xfs et ext4 (`lvextend -r` gère les deux) |
| Nom d'hôte et DNS saisis à l'installation | non appliqués, corrigés après | appliqués | comportement constaté de l'installateur |
| Fuseau horaire | réglé dans l'installateur | non proposé, UTC par défaut | Subiquity Server ne demande pas le fuseau |
| Gestion réseau | NetworkManager (`nmcli`) | netplan + systemd-networkd | — |
| Vérifier le DNS | `cat /etc/resolv.conf` | `resolvectl status` | Ubuntu passe par un résolveur local (systemd-resolved) |
| Synchronisation de l'heure | chronyd | systemd-timesyncd | — |
| Mettre à jour | `dnf -y update` | `apt update && apt -y full-upgrade` | gestionnaires de paquets différents (rpm / deb) |
| Historique des mises à jour | `dnf history` | `/var/log/apt/history.log` | — |
| Dépôt complémentaire | EPEL (`dnf install epel-release`) | universe, actif par défaut | — |
| Mises à jour automatiques | non activées | `unattended-upgrades` actif dès le premier démarrage | — |
| Statut de root verrouillé | `LK` | `L` | notation propre à chaque `passwd` |
| `passwd -S` redirigé vers un fichier | n'écrit rien | fonctionne | comportement constaté |
| Créer un utilisateur | `useradd -m -c` + `passwd` | `adduser` (interactif) | — |
| Groupe d'administration | `wheel` (GID 10) | `sudo` (GID 27) | convention de chaque famille |
| Service SSH | `sshd` | `ssh` | nom de l'unité systemd |
| Lire `/etc/ssh/sshd_config.d/` | `sudo` requis | lisible sans `sudo` | permissions du dossier |
| Méthodes d'authentification SSH | publickey, gssapi (Kerberos), password | publickey, password | configuration par défaut |
| Pare-feu | firewalld, actif par défaut | ufw, inactif par défaut | — |
| Port fermé | rejet (« No route to host ») | paquet ignoré (délai d'attente) | politiques reject / deny |
| Contrôle d'accès obligatoire | SELinux (visible dans `id`) | AppArmor | — |

## 9. Incidents et corrections

### rhel9-lab
| Incident | Diagnostic | Correction |
|---|---|---|
| Disque sur un stockage LVM classique | snapshot impossible, allocation non dynamique | disque déplacé vers `vg-vmstore-1To-thin` |
| NAT inopérant sur l'hôte | `net.ipv4.ip_forward = 0` | `/etc/sysctl.d/99-ip-forward.conf` |
| Installation lancée en partitionnement automatique | destination non ouverte dans Anaconda | retour à l'instantané `avant-installation` |
| Schéma imposé (43 Gio) plus grand que le disque (30 Gio) | calcul des tailles | disque porté à 50 Gio |
| Nouvelle taille non vue par l'installateur | table lue avant l'agrandissement | redémarrage de la VM |
| Volumes répartis dans deux groupes (`rlm` et `vg_sys`) | politique de taille automatique | renommage, « Aussi grand que possible », volumes rattachés à `vg_sys` |
| Nom d'hôte `localhost` puis `rhe19-lab` | non appliqué, puis faute de frappe | `hostnamectl set-hostname rhel9-lab` |
| Pas de résolution DNS | `ping 1.1.1.1` OK, `ping` d'un nom KO | `nmcli connection modify ens18 ipv4.dns …` |
| Image toujours attachée | `sr0` visible dans `lsblk` | lecteur CD/DVD vidé dans Proxmox |
| VM nommée `rhe19-lab` dans Proxmox | faute de frappe | renommée |

### ubuntu24-lab
| Incident | Diagnostic | Correction |
|---|---|---|
| Disque de 25 Gio, une seule interface | vérification de l'onglet Hardware | disque à 50 Gio, interface NAT ajoutée |
| Interfaces dans l'ordre inverse de rhel9-lab | adresses MAC dans l'installateur | ordre corrigé dans Proxmox, instantané refait |
| Partitionnement guidé par défaut | schéma non conforme vu à l'écran de profil | retour à l'instantané, « Custom storage layout » |
| Fuseau horaire en UTC | `timedatectl` | `timedatectl set-timezone Europe/Paris` |

### Partie D
| Incident | Diagnostic | Correction |
|---|---|---|
| Capture de `passwd -S root` vide (Rocky) | sortie non écrite quand elle est redirigée | sortie reprise du terminal, confirmée par `lslogins` |
| Nom complet « Prénom Mbaya » | valeur d'exemple copiée telle quelle | `usermod -c` / `chfn -f` |
| `sudo -n true` échoue | sudo exige un mot de passe | test avec `ssh -t … sudo true` |
| Preuves d'accès distant toutes en échec | VMs éteintes (« No route to host » y compris pour le compte nominatif) | VMs redémarrées, tests relancés |

Détail chronologique : `docs/journal.md`.

## 10. Reconstruction rapide

### 1. Hyperviseur (shell Proxmox)
Prérequis : `vmbr1` (NAT 192.168.2.0/24) et `vmbr2` (OVS, VLAN 42) configurés (voir rubrique 2).
```bash
qm create 202 --name rhel9-lab --pool MASTERE-4 --ostype l26 \
  --cores 2 --memory 2048 --scsihw virtio-scsi-single \
  --scsi0 vg-vmstore-1To-thin:50,discard=on,iothread=1 \
  --ide2 iso-local:iso/NOM-ISO-ROCKY,media=cdrom \
  --net0 virtio,bridge=vmbr1 --net1 virtio,bridge=vmbr2,tag=42 \
  --boot 'order=scsi0;ide2'
qm snapshot 202 avant-installation
qm start 202
```
(même commande avec `201`, `ubuntu24-lab` et l'image `ubuntu-24.04.3-live-server-amd64.iso` pour Ubuntu)

### 2. Installation (manuelle)
Profil minimal, partitionnement de la rubrique 3, réseau de la rubrique 2, nom d'hôte, compte nominatif. Puis retirer l'image du lecteur.

### 3. Après installation — rhel9-lab
```bash
sudo hostnamectl set-hostname rhel9-lab
sudo nmcli connection modify ens18 ipv4.dns "1.1.1.1 9.9.9.9" && sudo nmcli connection up ens18
sudo dnf -y update && sudo dnf -y install epel-release && sudo reboot
sudo passwd -l root
sudo useradd -m -c "Nom complet" compte && sudo passwd compte && sudo usermod -aG wheel compte
echo "PermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/00-tp01-durcissement.conf
sudo sshd -t && sudo systemctl reload sshd
sudo firewall-cmd --permanent --remove-service=cockpit --remove-service=dhcpv6-client
sudo firewall-cmd --permanent --add-service=ssh && sudo firewall-cmd --reload
```

### 4. Après installation — ubuntu24-lab
```bash
sudo timedatectl set-timezone Europe/Paris
sudo apt update && sudo apt -y full-upgrade && sudo apt -y autoremove && sudo reboot
sudo passwd -l root
sudo adduser compte && sudo usermod -aG sudo compte
echo "PermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/00-tp01-durcissement.conf
sudo sshd -t && sudo systemctl reload ssh
sudo ufw allow OpenSSH && sudo ufw enable
```

### 5. Depuis le poste d'administration
```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_lab
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub compte@192.168.2.20
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub compte@192.168.2.21
```
Puis contrôles des rubriques 6 et 7, et instantané `apres-partie-D`.
