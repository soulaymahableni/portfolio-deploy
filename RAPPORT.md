# Rapport — TP1 : Ansible et Semaphore UI

## 1. Objectif et environnement
Automatiser le déploiement d'un portfolio Next.js sur un serveur Ubuntu 24.04 (`automatisation`, 192.168.190.141)
depuis un poste de contrôle (`admin-ansible`, 192.168.190.155), puis industrialiser l'exécution avec Semaphore UI.
La connexion se fait en SSH par clé ed25519 avec l'utilisateur `deploy` (sudo sans mot de passe).

## 2. Conception du playbook
Le playbook `site.yml` enchaîne quatre rôles : `common` (paquets de base et utilisateur), `nodejs` (dépôt NodeSource
et PM2), `app` (clone, installation, build, lancement via PM2) et `nginx` (reverse proxy vers `127.0.0.1:{{ app_port }}`).
Toutes les valeurs sont centralisées dans `group_vars/all.yml`. Le port de l'application a été fixé à 3001 car le port 3000
était déjà utilisé par un conteneur Docker sur la cible : il a suffi de changer une variable, sans toucher aux rôles.

## 3. Idempotence et handlers
Les modules `command` renvoient toujours « changed » ; ils sont donc conditionnés. `npm ci` et `npm run build`
s'exécutent seulement si la tâche `git` a détecté une modification (`register: repo` / `when: repo.changed`) ou si le
fichier `.next/BUILD_ID` est absent. Le démarrage PM2 n'a lieu que si `pm2 describe` échoue. Le redémarrage de
l'application est un handler notifié uniquement par la tâche `git`, et le rechargement de Nginx un handler notifié par le
template et l'activation du site. Résultat : le second passage affiche `changed=0` (capture `idempotence.png`).

## 4. Industrialisation avec Semaphore
Semaphore centralise la clé SSH (Key Store), le dépôt Git, l'inventaire et les variables (Variable Group). Le Task Template
« Deploy Portfolio » exécute `site.yml` ; la variable de Survey `app_version` permet de choisir la branche ou le tag à
déployer au lancement. Après modification d'une page du portfolio et push, la relance de la tâche montre la tâche `git`
en « changed » puis l'exécution du handler `Redémarrer l'application` ; un nouveau lancement sans commit ne redémarre rien.

## 5. Difficultés rencontrées
- Clé SSH sous Windows : `~` non interprété par PowerShell → utilisation de `$env:USERPROFILE`, puis travail directement
  depuis la VM `admin-ansible` (Ansible ne fonctionne pas nativement sous Windows).
- Erreurs d'IP (`.141` / `.155`) : vérification systématique avec `ip a`.
- Port 3000 occupé sur la cible → `app_port: 3001`.
- `become_user` vers un utilisateur non privilégié : installation du paquet `acl` nécessaire sur Ubuntu.

## 6. Bilan
Le déploiement est reproductible, idempotent et déclenchable en un clic. Améliorations possibles : HTTPS avec Certbot,
utilisation d'Ansible Vault pour les secrets, et déclenchement automatique de Semaphore par webhook à chaque push.
