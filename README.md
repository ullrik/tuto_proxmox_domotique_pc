# tuto_proxmox_domotique_pc
Mise en place du serveur domotique sur pc

## Backup
### Promox configuration
/!\ On part du principe que la sauvegarde est sur un 2ème disque dur.
Sur ce disque qui est monté dans /mnt/data créer le repertoire "backups" et dedans le repertoire "proxmox-config"
'''
cd /mnt/data
mkdir backups
mkdir backups/proxmox-config
'''
Création d'un script dans le répertoire root (de proxmox)
Au préalable créer un répertoire scripts. 
'''
nano /root/scripts/backup-proxmox-config.sh
'''
et le contenu : 
'''
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
'''
