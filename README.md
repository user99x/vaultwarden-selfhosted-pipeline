# Vaultwarden Local Hosting + GitLab CI/CD 🔐
Hébergez facilement votre propre instance Vaultwarden en local et automatisez votre déploiement avec GitLab Runner !

![License](https://img.shields.io/badge/license-Free-green.svg)  
![Security](https://img.shields.io/badge/security-High-red.svg)  

## 🛡️ Fonctionnalités de Sécurité & Déploiement
✔️ **Hébergement local sécurisé de Vaultwarden (HTTPS, sauvegardes)**
✔️ **Automatisation CI/CD avec GitLab Runner local**
✔️ **Déploiement rapide via Docker et Docker Compose**
✔️ **Gestion simple des mises à jour et des backups**
✔️ **Projet 100% auto-hébergé, sans dépendance à des services externes**

## 📋 Prérequis  
- Docker & Docker compose
- GitLab RUnner installé en local
- Gitlab (auto-hébérgé ou GitLab.com) 

## 🛠️ Guide complet : De 0 -> Coffre-fort Vaultwarden + Gitlab runner local + extraction de secret 
### 1. Installer Docker et Docker compose :  
```sh
apt update && apt install -y docker.io docker-compose
systemctl enable docker
systemctl start docker
docker --version
docker compose version 
```
### 2. Déployer Vaultwarden avec Docker compose
1. Crée le dossier :
```sh
mkdir -p /opt/vaultwarden
cd /opt/vaultwarden
```
2. Crée le fichier docker-compose.yml :
```sh
nano docker-compose.yml
````
3. Copie et colle dans le fichier docker-compose.yml :
```sh
version: '3'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./data:/data
    environment:
      WEBSOCKET_ENABLED: 'true'
```
4. Lance Vaultwarden :
```sh
docker-compose up -d
```
5. Va sur ton navigateur :
http://localhost:8080

### 3. Sécuriser l'accès Admin de Vaultwarden
1. Génère un token sécurisé :
```bash
docker exec -it vaultwarden /vaultwarden hash
```
Entre un mot de passe que tu veux, une fois effectuée tu auras un 'ADMIN_TOKEN=' tu le copies.
2. Modifier ton docker-compose.yml 
Ajoute : 
```yaml
environment:
  WEBSOCKET_ENABLED: 'true'
  ADMIN_TOKEN: '$TON_ARGON2ID'
```
3. Redémarre Vaultwarden
```bash
docker-compose down
docker-compose up -d
```
### 4. Configurer Vaultwarden (Web)
- Créer ton premier utilisateru admin
- Créer une organisation (ex : Entreprise)
- Créer une collection (ex : CI & CD)
- Ajoute un élement
- Remplis comme suit :
Nom : Clé SSH Déploiement CI
Type : Secure Note (note sécurisée pour ceux qui ont 5/20 de moyenne en anglais)
Note : Clé privée pour déploiement
-Maintenant ajoute un champ personnalisé :
Nom : clé_ssh
Valeur : Tu mets ta clé privée

Tu n'as pas de clé privée ? Panique pas trql
```bash
ssh-keygen -t ed25519 -C "clé déploiement GitLab CI"
```
Quand il te demande : 
"Enter file in wich to save the key" tu réponds avec : /root/.ssh/id_gitlab_ci
"Enter passphrase" : Laisse vide (appuie 2 fois sur entrée)
Pour lire ta clé : 
```bash
cat /root/.ssh/id_gitlab_ci
```
Ca t'afficheras : 
```bash
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```
TU COPIES TOUT LE CONTENU

### 5. Installer GitLab Runner en local 
```bash
curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
chmod +x /usr/local/bin/gitlab-runner
gitlab-runner --version
````
### 6. Créer ton projet CI local 
# 1. Reviens dans /root : 
```bash
cd ..
```
# 2. Crée le dossier propre et rentre dedans et initalise ton dépôt Git : 
```bash
mkdir -p vaultwarden/vault-ci-local
cd vaultwarden/vault-ci-local
git init
```
Retourne sur ton 'localhost:8080'
Dans l'admin console en bas à gauche, clique sur 'membres' puis 'inviter un membre' en haut a droite, dans courriel : ci-bot@entreprise.local (en vrai tu mets ce que tu veux)
Tu sauvegardes, tu te déconnectes, puis tu crées un compte avec l'email que tu viens de rentrer, une fois le compte créer, tu retournes sur ton compte admin et dans sa console. Tu confirmes l'invitation de l'invité, puis tu lui accordes les accès à la collection "CI & CD"
# 3. Créer ton .gitlab-ci.yml
```bash
nano .gitlab-ci.yml
```
Colle dedans : 
```yaml
stages:
  - test

test_vaultwarden:
  stage: test
  script:
    - apt update && apt install -y curl jq unzip
    - curl -sL 'https://vault.bitwarden.com/download/?app=cli&platform=linux' -o bw.zip
    - unzip bw.zip
    - mv bw /usr/local/bin/bw
    - chmod +x /usr/local/bin/bw
    - bw logout || true
    - bw config server http://localhost:8080
    - echo "🔓 Unlocking vault directly..."
    - SESSION=$(bw login "TON_EMAIL_DE_LUSER" "TON_MOT_DE_PASSE" --raw)
    - echo "-----🔑 Clé récupérée depuis Vaultwarden :-----"
    - bw get item "LID_DE_LA_CLE_SSH" --session "$SESSION" | jq -r '.fields[] | select(.name=="clé_ssh") | .value'
```
# 4 Installation de 'jq' & 'bw'
```bash
apt update
apt install -y jq
```
```
apt update
apt install -y unzip
```
```
curl -sL 'https://vault.bitwarden.com/download/?app=cli&platform=linux' -o bw.zip
unzip bw.zip
mv bw /usr/local/bin/bw
chmod +x /usr/local/bin/bw
```
```
unzip bw.zip
```
```
mv bw /usr/local/bin/bw
```
```
chmod +x /usr/local/bin/bw
```
# 5.Configurer le serveur + récuperer l'id de sa clée SSH de Vaultwarden
1. Configuration du serveur au local :
```
bw config server http://localhost:8080
```
2. Configuration de ton email + mdp :
```
bw login "TON_EMAIL_DE_LUSER" "TON_MOT_DE_PASSE"
```
3. Récuperer l'id de la clé SSH :
```
export BW_SESSION=$(bw unlock --raw)
bw list items --session "$BW_SESSION" | jq -r '.[] | "\(.name) => \(.id)"'
```
Tu devrais avoir en résultat :
```
? Master password: [hidden]
Clé SSH Déploiement CI => L_ID_DE_TA_CLE_SSH
```
Dès à présent tu copies l'id et tu modifies ton fichier .gitlab-ci.yml

Malheuresement il te sera impossible d'éxecuter le script car tu es sur la version 17.x GitLab runner ! 
# 6. Installer Gitlab Runner 15.x
1. Supprimer l'acuel Gitlab runner 17.x et installe gitlab runner 15.11.0
```bash
apt remove --purge -y gitlab-runner
curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/v15.11.0/binaries/gitlab-runner-linux-amd64
chmod +x /usr/local/bin/gitlab-runner
gitlab-runner --version
```
# 7. Ajoute ton .gitlab-ci.yml dans Git
```bash
git add .gitlab-ci.yml
git commit -m "Initial commit - Setup Vaultwarden CI"
```
#8. Exécute ton job et admire le résultat : 
```bash
gitlab-runner exec shell test_vaultwarden
``` 

## 📢 Assistance

N'hésite pas a me contacter via discord ou telegram **(check mon read.me)** pour toute demande d'assistance concernant l'utilisation de l'outil !

---

**by user99x** ⚡

<p align="center">
  <img src="user99x.jpeg" alt="Logo github" width="200" align="right">
</p>

