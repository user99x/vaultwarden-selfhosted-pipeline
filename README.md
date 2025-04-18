# Vaultwarden Local Hosting + GitLab CI/CD 🔐
Hébergez facilement votre propre instance Vaultwarden en local et automatisez votre déploiement avec GitLab Runner !

![License](https://img.shields.io/badge/license-Free-green.svg)  
![Security](https://img.shields.io/badge/security-High-red.svg)  

## 🛡️ Fonctionnalités de Sécurité & Déploiement
✔️ **Hébergement local sécurisé de Vaultwarden** (HTTPS, sauvegardes)
✔️ **Automatisation CI/CD** avec GitLab Runner local
✔️ **Déploiement rapide** via Docker et Docker Compose
✔️ **Gestion simple** des mises à jour et des backups
✔️ **Projet 100% auto-hébergé**, sans dépendance à des services externes

## 📋 Prérequis  
- Docker & Docker compose
- GitLab RUnner installé en local
- Un serveur GitLab (auto-hébergé ou GitLab.com)

---

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
Saisis le mot de passe souhaité. Une fois généré, tu obtiens un ADMIN_TOKEN=. Copie-le.
2. Modifie ton docker-compose.yml 
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
- Crée ton premier utilisateru admin
- Crée une organisation (ex : Entreprise)
- Crée une collection (ex : CI & CD)
- Ajoute un élement :
- Nom : Clé SSH Déploiement CI
- Type : Secure Note (note sécurisée pour ceux qui ont 5/20 de moyenne en anglais)
- Note : Clé privée pour déploiement
Puis ajoute un champ personnalisé :
- Nom : clé_ssh
- Valeur : Tu mets ta clé privée

### 5. Générer une clé SSH pour GitLab CI
Pas encore de clé privée ? Pas d'inquiétude, voici comment la créer : 
```bash
ssh-keygen -t ed25519 -C "clé déploiement GitLab CI"
```
Lorsqu'il te demande : 
- "Enter file in wich to save the key" tapes : /root/.ssh/id_gitlab_ci
- "Enter passphrase" : Laisse vide (appuie 2 fois sur entrée)
Pour afficher ta clé : 
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

### 6. Installer GitLab Runner en local 
```bash
curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
chmod +x /usr/local/bin/gitlab-runner
gitlab-runner --version
````
### 7. Créer ton projet CI local 
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
# 3. Configure Vaultwarden pour CI :
- Sur http://localhost:8080, connecte-toi à l'admin console.
- Invite un nouveau membre :
-   Email : ci-bot@entreprise.local (ou ce que tu veux)
- Crée le compte pour cet utilisateur invité.
- En tant qu'admin, accepte l'invitation et donne-lui accès à la collection "CI & CD".
# 4. Crée ton .gitlab-ci.yml
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
    - echo "Unlocking vault directly..."
    - SESSION=$(bw login "TON_EMAIL_DE_LUSER" "TON_MOT_DE_PASSE" --raw)
    - echo "-----Clé récupérée depuis Vaultwarden :-----"
    - bw get item "LID_DE_LA_CLE_SSH" --session "$SESSION" | jq -r '.fields[] | select(.name=="clé_ssh") | .value'
```
### 8. Installation de 'jq' & 'bw'
```bash
apt update
apt install -y jq unzip
curl -sL 'https://vault.bitwarden.com/download/?app=cli&platform=linux' -o bw.zip
unzip bw.zip
mv bw /usr/local/bin/bw
chmod +x /usr/local/bin/bw
```
```
apt update
apt install -y unzip
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
### 9.Configurer le serveur + récuperer l'id de sa clée SSH de Vaultwarden
# 1. Définir l'URL du serveur : 
```
bw config server http://localhost:8080
```
# 2. Connexion au compte : 
```
bw login "TON_EMAIL_DE_LUSER" "TON_MOT_DE_PASSE"
```
# 3. Récuperer l'id de l'élément :
```
export BW_SESSION=$(bw unlock --raw)
bw list items --session "$BW_SESSION" | jq -r '.[] | "\(.name) => \(.id)"'
```
Tu obtiens :
```
? Master password: [hidden]
Clé SSH Déploiement CI => L_ID_DE_TA_CLE_SSH
```
Copie cet ID et mets-le dans ton .gitlab-ci.yml

### 10. Installer Gitlab Runner 15.x (si problème)
Si tu es sur GitLab Runner 17.x, il peut y avoir des incompatibilités. Installe GitLab Runner 15.11.0 :
```bash
apt remove --purge -y gitlab-runner
curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/v15.11.0/binaries/gitlab-runner-linux-amd64
chmod +x /usr/local/bin/gitlab-runner
gitlab-runner --version
```
### 11. Lancer ton job GitLab Runner
Ajoute ton fichier .gitlab-ci.yml :
```bash
git add .gitlab-ci.yml
git commit -m "Initial commit - Setup Vaultwarden CI"
```
Puis exécute : 
```bash
gitlab-runner exec shell test_vaultwarden
``` 
Admire le résultat !

---

## 📢 Assistance

N'hésite pas a me contacter via discord ou telegram **(check mon read.me)** pour toute demande d'assistance concernant l'utilisation de l'outil !

---

**by user99x** ⚡

<p align="center">
  <img src="user99x.jpeg" alt="Logo github" width="200" align="right">
</p>

