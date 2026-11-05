# 07 - Sauvegardes avec restic

## Objectif
Sauvegarder automatiquement chaque nuit la plateforme, **vérifier** les sauvegardes, savoir **restaurer**, être **alerté** en cas d'échec et garder une **copie hors serveur**.

## Machine
Serveur cloud Ubuntu 24.04 (+ poste client pour la copie).

## 1. Initialiser le dépôt
```bash
sudo apt install -y restic
sudo sh -c 'openssl rand -base64 24 > /root/.restic-pass && chmod 600 /root/.restic-pass'
sudo restic -r /var/backups/restic --password-file /root/.restic-pass init
sudo restic -r /var/backups/restic --password-file /root/.restic-pass backup /srv/plateforme /etc --tag manuel
```
> Sans le mot de passe, la sauvegarde est **illisible** : le conserver en lieu sûr, ailleurs que sur le serveur.

## 2. Script de sauvegarde
```bash
sudo tee /usr/local/sbin/backup-plateforme.sh > /dev/null <<'EOS'
#!/bin/bash
set -euo pipefail
export RESTIC_REPOSITORY=/var/backups/restic
export RESTIC_PASSWORD_FILE=/root/.restic-pass
cd /srv/plateforme
trap 'curl -s -H "Priority: high" -H "Title: Sauvegarde en echec" -d "backup-plateforme a echoue sur $(hostname)" https://notif.<DOMAINE>/<SUJET_NTFY> >/dev/null || true' ERR
trap 'docker compose start kuma >/dev/null 2>&1 || true' EXIT
docker compose stop kuma
restic backup /srv/plateforme /etc --tag auto
docker compose start kuma
restic forget --keep-daily 7 --keep-weekly 4 --prune
restic check
EOS
sudo chmod 750 /usr/local/sbin/backup-plateforme.sh
```
> Kuma est arrêté pendant la copie pour que sa base SQLite soit cohérente. Le `trap EXIT` le relance même en cas d'erreur.

## 3. Service et timer systemd
```bash
sudo tee /etc/systemd/system/backup-plateforme.service > /dev/null <<'EOS'
[Unit]
Description=Sauvegarde restic de la plateforme
After=docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup-plateforme.sh
EOS

sudo tee /etc/systemd/system/backup-plateforme.timer > /dev/null <<'EOS'
[Unit]
Description=Sauvegarde quotidienne de la plateforme

[Timer]
OnCalendar=*-*-* 03:30:00
Persistent=true
RandomizedDelaySec=5m

[Install]
WantedBy=timers.target
EOS

sudo systemd-analyze verify /etc/systemd/system/backup-plateforme.service /etc/systemd/system/backup-plateforme.timer
sudo systemctl daemon-reload
sudo systemctl enable --now backup-plateforme.timer
sudo systemctl start backup-plateforme.service
sudo journalctl -u backup-plateforme.service --no-pager -n 20
```

## 4. Tester une restauration
```bash
sudo restic -r /var/backups/restic --password-file /root/.restic-pass restore latest --target /tmp/restore-test --include /srv/plateforme/data/kuma
sudo ls -l /tmp/restore-test/srv/plateforme/data/kuma
sudo rm -rf /tmp/restore-test
```

## 5. Tester l'alerte (échec volontaire)
```bash
sudo cp /root/.restic-pass /root/.restic-pass.ok
echo "faux" | sudo tee /root/.restic-pass > /dev/null
sudo systemctl start backup-plateforme.service
sudo mv /root/.restic-pass.ok /root/.restic-pass
sudo docker compose -f /srv/plateforme/compose.yaml ps kuma
```
Attendu : le service échoue (`wrong password`), la notification « Sauvegarde en echec » arrive sur le téléphone et Kuma est de nouveau `Up`.

## 6. Copie hors serveur
Sur le serveur :
```bash
sudo tar -czf /tmp/restic-copy.tgz -C /var/backups restic
sudo chown <UTILISATEUR> /tmp/restic-copy.tgz
sha256sum /tmp/restic-copy.tgz
```
Sur le poste client (PowerShell) :
```powershell
mkdir "$env:USERPROFILE\vps-backup" -Force
scp <UTILISATEUR>@<IP_SERVEUR>:/tmp/restic-copy.tgz "$env:USERPROFILE\vps-backup"
Get-FileHash "$env:USERPROFILE\vps-backup\restic-copy.tgz" -Algorithm SHA256
```
Les deux empreintes doivent être identiques. Puis supprimer la copie temporaire :
```bash
sudo rm -f /tmp/restic-copy.tgz
```

## Vérification
```bash
sudo restic -r /var/backups/restic --password-file /root/.restic-pass snapshots
sudo restic -r /var/backups/restic --password-file /root/.restic-pass check
systemctl list-timers backup-plateforme.timer --no-pager
```
Attendu : des snapshots `auto` quotidiens, `no errors were found`, prochaine exécution planifiée.

## Pièges
- Un dépôt sur le même disque que les données ne protège pas d'une perte du serveur : d'où la copie hors serveur.
- Ne jamais versionner `/root/.restic-pass`.
