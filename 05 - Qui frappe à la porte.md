# 05 - Qui frappe à la porte ? Journaux, pare-feu, CrowdSec

## Objectif
Constater les tentatives d'intrusion, **réduire la surface exposée** à 22/80/443 et **bannir automatiquement** les attaquants.

## Machine
Serveur cloud Ubuntu 24.04 (+ poste client pour le scan).

## 1. Lire les journaux SSH
```bash
sudo journalctl -u ssh --since today | grep -c "Invalid user"
sudo journalctl -u ssh --since today | grep "Invalid user" | awk '{print $(NF-2)}' | sort | uniq -c | sort -rn | head -5
```

## 2. Pare-feu nftables
```bash
sudo apt install -y nftables
sudo tee /etc/nftables.conf > /dev/null <<'NFT'
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif "lo" accept
        meta l4proto { icmp, ipv6-icmp } accept
        tcp dport { 22, 80, 443 } accept
    }
}
NFT
sudo nft -c -f /etc/nftables.conf
sudo systemctl enable --now nftables
sudo systemctl restart nftables
```
> `flush ruleset` efface aussi les règles créées par **Docker** et **CrowdSec**. Après chaque rechargement :
```bash
sudo systemctl restart docker
sudo systemctl restart crowdsec-firewall-bouncer
```
Pour que Docker démarre toujours après le pare-feu au boot :
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/10-after-nftables.conf > /dev/null <<'CONF'
[Unit]
After=nftables.service
Wants=nftables.service
CONF
sudo systemctl daemon-reload
```

## 3. CrowdSec et son bouncer
Suivre l'installation officielle du dépôt : https://docs.crowdsec.net/u/getting_started/installation/linux/
```bash
sudo apt install -y crowdsec crowdsec-firewall-bouncer-nftables
sudo cscli bouncers list
```
Si le bouncer n'apparaît pas ou ne tourne pas, l'enregistrer et l'activer :
```bash
KEY=$(sudo cscli bouncers add firewall-bouncer -o raw)
sudo sed -i "s/^api_key:.*/api_key: $KEY/" /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
sudo systemctl enable --now crowdsec-firewall-bouncer
```

## Vérification
Sur le serveur :
```bash
sudo cscli bouncers list && systemctl is-active crowdsec-firewall-bouncer
sudo cscli decisions list | head -15
sudo nft list table ip crowdsec | grep -E "chain|hook|drop" | head -8
```
Depuis le poste client (PowerShell) :
```powershell
foreach ($p in 22,80,443,3001,3306,8080,8081) { Test-NetConnection <IP_SERVEUR> -Port $p -WarningAction SilentlyContinue | Select-Object RemotePort, TcpTestSucceeded }
```
Attendu : `True` pour 22, 80, 443 et `False` pour les autres ; bouncer `active` ; des IP dans `decisions list`.

## Pièges
- Un bouncer installé n'est pas forcément actif : vérifier `systemctl is-active`.
- Garder une session SSH ouverte pendant les tests de pare-feu pour ne pas s'enfermer dehors.
