# 08 - Tout en code : Git et Ansible

## Objectif
Décrire la plateforme dans des fichiers **versionnés sous Git** et la déployer avec un playbook **Ansible idempotent**, capable de **reconstruire une machine vierge**.

## Machines
- Serveur cloud : création du dépôt et du playbook.
- Seconde VM cloud vierge : test de reconstruction.

## 1. Versionner la configuration
```bash
cd /srv/plateforme
sudo tee .gitignore > /dev/null <<'EOF2'
data/
.env
*.bak*
EOF2
sudo cp /etc/nftables.conf /srv/plateforme/nftables.conf
sudo git init -b main
sudo git config user.name "<NOM_PRENOM>"
sudo git config user.email "<EMAIL_GITHUB>"
sudo git add .
sudo git commit -m "plateforme: kuma, ntfy et caddy en https"
```
Créer un dépôt vide sur GitHub, ajouter une clé SSH du serveur dans **GitHub > Settings > SSH and GPG keys**, puis :
```bash
sudo git remote add origin git@github.com:<COMPTE_GITHUB>/plateforme.git
sudo git push -u origin main
```
> Avant chaque push, vérifier qu'aucun secret n'est suivi :
```bash
sudo git ls-files
sudo git grep -n -i -E "password|secret|token|api_key"
```

## 2. Installer Ansible
```bash
sudo apt install -y ansible
ansible --version | head -3
```

## 3. Écrire le playbook
```bash
sudo mkdir -p /srv/plateforme/ansible && cd /srv/plateforme/ansible

sudo tee ansible.cfg > /dev/null <<'EOF2'
[defaults]
inventory = inventory.ini
retry_files_enabled = False
interpreter_python = auto_silent
EOF2

sudo tee inventory.ini > /dev/null <<'EOF2'
[serveurs]
localhost ansible_connection=local
EOF2
```

`site.yml` :
```yaml
---
- name: Plateforme (SSH, pare-feu, Docker, pile)
  hosts: serveurs
  become: true
  vars:
    plateforme_dir: /srv/plateforme
    repo_dir: "{{ playbook_dir }}/.."
    status_host: "status.<DOMAINE>"
    notif_host: "notif.<DOMAINE>"

  tasks:
    - name: SSH - pas de root, pas de mot de passe
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/10-durcissement.conf
        mode: "0644"
        content: |
          PermitRootLogin no
          PasswordAuthentication no
        validate: /usr/sbin/sshd -t -f %s
      notify: Recharger ssh

    - name: Pare-feu nftables (22, 80, 443)
      ansible.builtin.copy:
        src: "{{ repo_dir }}/nftables.conf"
        dest: /etc/nftables.conf
        mode: "0755"
      notify: Recharger nftables

    - name: nftables actif
      ansible.builtin.systemd:
        name: nftables
        enabled: true
        state: started

    - name: Docker installé
      ansible.builtin.apt:
        name: [docker.io, docker-compose-v2]
        state: present
        update_cache: true

    - name: Docker démarre après nftables
      ansible.builtin.copy:
        dest: /etc/systemd/system/docker.service.d/10-after-nftables.conf
        content: |
          [Unit]
          After=nftables.service
          Wants=nftables.service
      notify: Redémarrer docker

    - name: Appliquer les redémarrages avant la pile
      ansible.builtin.meta: flush_handlers

    - name: Dossiers de la plateforme
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
      loop: ["{{ plateforme_dir }}", "{{ plateforme_dir }}/data"]

    - name: Config de la pile (noms paramétrés)
      ansible.builtin.template:
        src: "{{ repo_dir }}/{{ item }}"
        dest: "{{ plateforme_dir }}/{{ item }}"
      loop: [compose.yaml, Caddyfile]

    - name: Pile docker compose démarrée
      ansible.builtin.command: docker compose up -d
      args:
        chdir: "{{ plateforme_dir }}"
      register: compose_up
      changed_when: compose_up.stderr is search('Created|Started|Recreated')

  handlers:
    - name: Recharger ssh
      ansible.builtin.service: { name: ssh, state: reloaded }
    - name: Recharger nftables
      ansible.builtin.service: { name: nftables, state: restarted }
    - name: Redémarrer docker
      ansible.builtin.systemd: { name: docker, state: restarted, daemon_reload: true }
```
> Dans `compose.yaml` et `Caddyfile`, remplacer les noms en dur par `{{ status_host }}` et `{{ notif_host }}` pour réutiliser le playbook sur une autre machine.

## 4. Tester l'idempotence
```bash
cd /srv/plateforme/ansible
ansible-playbook site.yml --check --diff
ansible-playbook site.yml
ansible-playbook site.yml
```
Attendu : au second passage, `changed=0` dans le `PLAY RECAP`.

## 5. Reconstruire une machine vierge
Sur la seconde VM :
```bash
sudo apt update && sudo apt install -y git ansible
git clone https://github.com/<COMPTE_GITHUB>/plateforme.git ~/plateforme-repo
cd ~/plateforme-repo/ansible
ansible-playbook site.yml -K -e status_host=status2.<DOMAINE> -e notif_host=notif2.<DOMAINE>
ansible-playbook site.yml -K -e status_host=status2.<DOMAINE> -e notif_host=notif2.<DOMAINE>
```
> Créer d'abord les enregistrements DNS `status2` et `notif2` vers l'IP de la seconde VM, et ouvrir 22/80/443 dans son pare-feu cloud.

## Vérification
```bash
git -C ~/plateforme-repo log --oneline | head -5
sudo ss -tlnp | grep -E ":22 |:80 |:443 "
sudo docker compose -f /srv/plateforme/compose.yaml logs caddy --tail 40 | grep -i obtained
```
Attendu : 1er passage avec des `changed`, 2e passage `changed=0`, seuls 22/80/443 en écoute, certificats obtenus.

## Pièges
- Le module `community.docker.docker_compose_v2` peut signaler un faux `changed` : utiliser `command` + `changed_when`.
- Créer le compte admin de Kuma juste après le déploiement : la page de création est publique.
