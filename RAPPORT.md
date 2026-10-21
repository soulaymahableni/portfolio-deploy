# Rapport — TP1 : Ansible et Semaphore UI

## 1. Objectif et environnement
Automatiser le déploiement d'un portfolio Next.js sur un serveur Ubuntu (`automatisation`, 192.168.190.141)
depuis un poste de contrôle (`admin-ansible`, 192.168.190.155), puis industrialiser l'exécution avec Semaphore UI
lancé en conteneur Docker. La connexion se fait en SSH par clé ed25519 avec l'utilisateur `deploy` (sudo sans mot de passe).

## 2. Conception du playbook
`site.yml` enchaîne quatre rôles : `common` (paquets de base et utilisateur), `nodejs` (dépôt NodeSource et PM2),
`app` (clone, `npm ci`, build, lancement via PM2) et `nginx` (reverse proxy vers `127.0.0.1:{{ app_port }}`).
Toutes les valeurs sont centralisées dans `group_vars/all.yml`. Le port 3000 étant déjà utilisé par un conteneur Docker
sur la cible, l'application écoute sur 3001 : il a suffi de modifier une variable, sans toucher aux rôles.

## 3. Idempotence et handlers
Les modules `command` renvoient toujours « changed » ; ils sont donc conditionnés. `npm ci` et `npm run build` ne
s'exécutent que si la tâche `git` a détecté une modification (`register: repo`, `when: repo.changed`) ou si
`.next/BUILD_ID` est absent. Le démarrage PM2 n'a lieu que si `pm2 describe` échoue. Le redémarrage de l'application
est un handler notifié uniquement par la tâche `git` ; le rechargement de Nginx est un handler notifié par le template
et l'activation du site. Deux exécutions successives donnent `changed=0` à la seconde.

## 4. Industrialisation avec Semaphore
Semaphore centralise la clé SSH (Key Store), le dépôt Git, l'inventaire et les variables (Variable Group `prod`).
Le Task Template « Deploy Portfolio » exécute `site.yml` ; la variable de Survey `app_version` permet de choisir la
branche à déployer. Après le passage de « Version 1.0 » à « Version 2.0 » dans le dépôt de l'application, la relance
montre la tâche `git` en « changed » puis l'exécution du handler `Redémarrer l'application`, et le site affiche la
nouvelle version.

## 5. Difficultés rencontrées et solutions
- **Clé SSH sous Windows** : `~` non interprété par PowerShell → travail directement depuis la VM `admin-ansible`.
- **Horloge de la cible en retard** : apt refusait les dépôts (« Release file is not valid yet ») → mise à l'heure
  (`date -s`) et installation de `chrony`.
- **Dépôt NodeSource déjà déclaré** avec une autre clé (`Signed-By` en conflit) : configuration faite à la main
  auparavant (dérive de configuration) → suppression de l'ancienne déclaration dans `/etc/apt/sources.list.d/`.
- **Port 80 inaccessible** : K3s (Traefik) installé sur la cible interceptait le trafic → arrêt de K3s et ouverture
  du port 80 dans le pare-feu.
- **Semaphore** : conteneur en redémarrage en boucle avec la base `bolt` → passage à `sqlite` ; téléchargement Galaxy
  bloqué → suppression de `requirements.yml` (la collection `community.general` est déjà présente dans l'image) ;
  premier run « Success » mais `no hosts matched` → correction du contenu de l'inventaire statique.
- **`become_user` vers un utilisateur non privilégié** : installation du paquet `acl` nécessaire sur Ubuntu.

## 6. Bilan
Le déploiement est reproductible, idempotent et déclenchable en un clic depuis Semaphore, avec un redémarrage de
l'application uniquement lorsque le code change. Améliorations possibles : HTTPS avec Certbot, Ansible Vault pour les
secrets, et déclenchement automatique de Semaphore par webhook à chaque push.
