# Déploiement Odoo 19 sur un VPS

Ce projet déploie Odoo Community 19 avec PostgreSQL 18 et Traefik comme reverse proxy HTTPS.

Architecture :

```text
Internet
   │
   ├── :80  ──> Traefik ── redirection HTTPS
   └── :443 ─> Traefik ──> Odoo :8069
                              └── Websocket :8072

Odoo ──> PostgreSQL :5432
```

Le port `8069` d’Odoo et le port `5432` de PostgreSQL ne sont pas publiés sur le VPS.

## 1. Préparer le VPS

Ces instructions supposent un VPS Debian ou Ubuntu fraîchement installé.

Mettre le système à jour :

```bash
sudo apt update && sudo apt upgrade -y
```

Installer Docker et le plugin Compose avec la méthode recommandée par la distribution ou la documentation Docker. Vérifier ensuite :

```bash
docker --version
docker compose version
```

Créer un utilisateur d’administration non root si nécessaire, puis l’ajouter au groupe Docker :

```bash
sudo usermod -aG docker $USER
```

Fermer puis rouvrir la session SSH pour que le changement de groupe soit pris en compte.

## 2. Configurer le DNS

Créer un enregistrement DNS de type `A` :

```text
odoo.example.com  ->  IP_PUBLIQUE_DU_VPS
```

Vérifier la résolution avant de demander le certificat :

```bash
dig +short odoo.example.com
```

Le domaine doit pointer vers le VPS et les ports TCP `80` et `443` doivent être accessibles depuis Internet. Traefik utilise le challenge HTTP de Let’s Encrypt sur le port `80`.

## 3. Configurer le pare-feu

Avec UFW, exemple minimal :

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

Ne pas ouvrir les ports `8069`, `8072` ou `5432` sur Internet.

## 4. Installer le projet

Créer le répertoire de déploiement :

```bash
sudo mkdir -p /opt/odoo
sudo chown -R "$USER":"$USER" /opt/odoo
cd /opt/odoo
```

Copier dans ce répertoire :

- `docker-compose.yml`
- `config/odoo.conf`
- `.env.example`

Le fichier `.env` ne doit jamais être ajouté au dépôt Git.

## 5. Créer le fichier `.env`

```bash
cp .env.example .env
chmod 600 .env
nano .env
```

Renseigner des valeurs réelles et uniques :

```dotenv
POSTGRES_USER=odoo
POSTGRES_PASSWORD=mot-de-passe-postgresql-long-et-aleatoire
POSTGRES_DB=postgres
ODOO_DB_NAME=odoo_prod
ODOO_ADMIN_PASSWD=mot-de-passe-maitre-odoo-long-et-aleatoire
DOMAIN=odoo.example.com
LETSENCRYPT_EMAIL=admin@example.com
```

`ODOO_ADMIN_PASSWD` est le mot de passe maître utilisé pour les opérations d’administration des bases. Il est différent du mot de passe des utilisateurs Odoo.

Générer un mot de passe aléatoire, par exemple :

```bash
openssl rand -base64 32
```

## 6. Vérifier la configuration avant démarrage

```bash
docker compose config
```

Cette commande doit se terminer sans erreur. Elle affiche les valeurs interpolées : ne partagez jamais sa sortie si elle contient des secrets.

## 7. Premier démarrage

Démarrer les services :

```bash
docker compose up -d
```

Contrôler l’état :

```bash
docker compose ps
docker compose logs -f traefik
```

Lors du premier démarrage, Traefik demande automatiquement le certificat Let’s Encrypt. Le volume `traefik-letsencrypt` conserve le fichier ACME entre les redémarrages.

Tester ensuite :

```text
https://odoo.example.com
```

## 8. Créer ou restaurer la base Odoo

La configuration utilise :

```ini
list_db = False
```

La base correspondant à `ODOO_DB_NAME` doit donc exister avant l’utilisation normale d’Odoo.

Pour une nouvelle installation, créer la base depuis un poste d’administration ou depuis un accès temporaire contrôlé. Ne laissez pas le gestionnaire de bases exposé publiquement en production.

Pour une restauration, restaurer les deux éléments suivants :

1. le dump PostgreSQL ;
2. le filestore situé dans `/var/lib/odoo/filestore/<nom_de_base>`.

Ces deux éléments doivent provenir d’une sauvegarde cohérente.

## 9. Vérifications post-déploiement

Vérifier les conteneurs :

```bash
docker compose ps
```

Vérifier les journaux :

```bash
docker compose logs --tail=200 db
docker compose logs --tail=200 odoo
docker compose logs --tail=200 traefik
```

Vérifier que seuls les ports attendus sont publiés :

```bash
docker compose port traefik 80
docker compose port traefik 443
```

Tester la redirection HTTPS :

```bash
curl -I http://odoo.example.com
curl -I https://odoo.example.com
```

Le trafic HTTP doit être redirigé vers HTTPS.

## 10. Sauvegardes

Une sauvegarde de production doit contenir :

- un dump PostgreSQL ;
- le filestore Odoo ;
- une copie hors du VPS.

Le dump seul ne suffit pas : les pièces jointes et images sont stockées dans le filestore.

La procédure de sauvegarde doit être automatisée avec un cron ou un outil dédié, puis testée régulièrement sur un environnement de restauration. Ne considérez pas une sauvegarde comme valide tant qu’une restauration complète n’a pas été vérifiée.

## 11. Mise à jour

Avant toute mise à jour :

1. effectuer une sauvegarde complète ;
2. vérifier la compatibilité des modules personnalisés ;
3. tester l’image sur une instance de staging ;
4. prévoir une procédure de retour arrière.

Mettre à jour les images :

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Ne pas utiliser une mise à jour majeure PostgreSQL comme une simple mise à jour d’image. Une migration PostgreSQL entre versions majeures doit être planifiée et testée séparément.

## 12. Commandes d’exploitation utiles

Arrêter la stack :

```bash
docker compose down
```

Redémarrer uniquement Odoo :

```bash
docker compose restart odoo
```

Suivre les logs Odoo :

```bash
docker compose logs -f odoo
```

Ouvrir un shell PostgreSQL :

```bash
docker compose exec db psql -U odoo -d postgres
```

Si `POSTGRES_USER` a été modifié dans `.env`, remplacer `odoo` par la valeur configurée.

## 13. Points de sécurité à conserver

- Ne pas publier directement `8069`, `8072` ou `5432`.
- Ne pas commiter `.env`.
- Utiliser HTTPS uniquement pour les utilisateurs.
- Conserver `proxy_mode = True` derrière Traefik.
- Conserver `list_db = False` en production.
- Utiliser un mot de passe maître Odoo unique.
- Sauvegarder PostgreSQL et le filestore ensemble.
- Protéger l’accès SSH avec des clés et, si possible, désactiver l’authentification SSH par mot de passe.
- Surveiller régulièrement l’espace disque, la mémoire et les logs.
