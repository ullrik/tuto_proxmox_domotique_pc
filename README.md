# tuto_proxmox_domotique_pc
Mise en place du serveur domotique sur pc

## Backup
### Promox configuration
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
