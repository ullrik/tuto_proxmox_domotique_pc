# tuto_proxmox_domotique_pc
Mise en place du serveur domotique sur pc

## Configuration 

### Backup Externe
#### Montage
Brancher le disque externe en USB 
1. Identification du disque USB
```txt
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,LABEL,UUID,MOUNTPOINTS
blkid
ls -l /dev/disk/by-id/ | grep -E 'usb|USB'
```
2. Monter temporairement le disque
```txt
# Créer le point de montage :
mkdir -p /mnt/backup-usb
# Monter le disque USB (UUID via lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,LABEL,UUID,MOUNTPOINTS) :
mount -t exfat UUID=XXXX-XXXX /mnt/backup-usb
# Vérifier le montage :
df -h /mnt/backup-usb
# Vérifier le contenu du disque :
ls -lah /mnt/backup-usb
```
3. Configurer le montage automatique
```txt
# Sauvegarder le fichier fstab :
cp /etc/fstab /etc/fstab.bak-$(date +%F)
# Ajouter le disque USB au démarrage (UUID via lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,LABEL,UUID,MOUNTPOINTS) :
echo 'UUID=XXXX-XXXX /mnt/backup-usb exfat defaults,nofail,x-systemd.device-timeout=10 0 0' >> /etc/fstab
# /!\ Le paramètre nofail est important : Proxmox pourra démarrer normalement même si le disque USB est débranché.
# Vérifier la configuration
umount /mnt/backup-usb
systemctl daemon-reload
mount -a
# Vu du disque et confirme le bon montage automatique
df -h /mnt/backup-usb
```

#### Immich 
Prérequis : 
Etape Montage
```txt
mkdir -p /mnt/backup-usb/Immich
```
1. Préparer Immich :
Arrêter temporairement les deux services Immich, sans arrêter PostgreSQL ni Redis :
```txt
pct exec 100 -- systemctl stop immich-web.service immich-ml.service
# Vérifie leur état :
pct exec 100 -- systemctl status immich-web.service immich-ml.service --no-pager
# Les deux services doivent apparaître comme inactive (dead).
```
2. Copie des données :
```txt
rsync -rltvh --info=progress2 \
  --no-owner --no-group --no-perms \
  /mnt/data/immich/ \
  /mnt/backup-usb/Immich/
```
3. Controler :
```txt
# Taille réel des backups :
du -sh --apparent-size /mnt/data/immich
du -sh --apparent-size /mnt/backup-usb/Immich
```
Contrôler l'epsace utilisé et restant sur le disque backup : 
```txt
df -h /mnt/backup-usb
```
Vérifier l'intégrité de la copie : 
```txt
rsync -rclnvi \
  --no-owner --no-group --no-perms \
  /mnt/data/immich/ \
  /mnt/backup-usb/Immich/
```

### Autres
Désactivation du Bluetooth. Depuis le BIOS (F10) :
1.Dans Security > Built-In Device Options (en bas).
2.Cherche une ligne Bluetooth, Wireless LAN ou Embedded WLAN/Bluetooth. Et le mettre sur Disable.
(Optionnel) Coté Linux/Proxmox :
```txt
rfkill block bluetooth
# Pour controler (Soft blocked: yes)
rfkill list bluetooth
```

Vérifier que les sauvegardes PostgreSQL sont exploitables :
```txt
for f in /mnt/data/immich/backups/*.sql.gz; do
  echo "Vérification : $(basename "$f")"
  gzip -t "$f" && echo "OK" || echo "ERREUR"
done
```

## Backup
### Proxmox
| Planification| Sélection| Rétention|
|:----------|:----:|------:|
| Mensuel (1er du mois, 00:00) | ex : 110 – AdGuard Home | keep-last=2 | 
| Quotidien 21:00 | LXC 100 – Immich, VM 103 – Home Assistant et LXC 101 | keep-last=3 | 

ex : https://domo.rem81.com/index.php/2026/08/01/proxmox-backup-server-ma-strategie-de-sauvegarde-jobs-retention-stockage/
### Proxmox configuration (système)
/!\ On part du principe que la sauvegarde est sur un 2ème disque dur.
Sur ce disque qui est monté dans /mnt/data créer le repertoire "backups" et dedans le repertoire "proxmox-config"
```txt
cd /mnt/data
mkdir backups
mkdir backups/proxmox-config
```
Création d'un script dans le répertoire root (de proxmox)
Au préalable créer un répertoire scripts. 
```txt
nano /root/scripts/backup-proxmox-config.sh
```
et le contenu : 
```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/mnt/data/backups/proxmox-config"
DATE=$(date +%F)

mkdir -p "$BACKUP_DIR"

tar -czf "$BACKUP_DIR/proxmox-config-$DATE.tar.gz" \
  /etc/pve \
  /etc/network/interfaces \
  /etc/hosts \
  /etc/hostname \
  /etc/resolv.conf \
  /etc/fstab

# Supprime les sauvegardes de plus de 30 jours
find "$BACKUP_DIR" -type f -name "proxmox-config-*.tar.gz" -mtime +30 -delete
```
et pour l'executer : 
```txt
chmod 700 /root/scripts/backup-proxmox-config.sh
```

#### Mise en place d'un service systemd
Partie 1 : le service 
```txt
nano /etc/systemd/system/backup-proxmox-config.service
```
avec : 
```bash
[Unit]
Description=Sauvegarde de la configuration Proxmox
RequiresMountsFor=/mnt/data/backups
After=pve-cluster.service

[Service]
Type=oneshot
ExecStart=/root/scripts/backup-proxmox-config.sh
```
Partie 2 : le timer
```txt
nano /etc/systemd/system/backup-proxmox-config.timer
```
avec : 
```bash
[Unit]
Description=Sauvegarde quotidienne de la configuration Proxmox

[Timer]
OnCalendar=*-*-* 05:30:00
Persistent=true
RandomizedDelaySec=5m
Unit=backup-proxmox-config.service

[Install]
WantedBy=timers.target
```
Partie 2 : l'activation
```txt
systemctl daemon-reload
systemctl enable --now backup-proxmox-config.timer
systemctl start backup-proxmox-config.service
```
Commande utile : 
```txt
# Le status du service
systemctl status backup-proxmox-config.service
# Information sur la prochaine exécution
list-timers backup-proxmox-config.timer
# Log de l'exécution
journalctl -u backup-proxmox-config.service
# Controler si un nouveau fichier est bien présent
ls -lh /mnt/data/backups/proxmox-config/
```
