# 🛡️ Jour 7 --- Sécurité Avancée, Sauvegardes et Guide de Dépannage

## 🎯 Objectif du jour

À la fin de cette dernière journée, ton infrastructure sera **durcie, sauvegardée et sous surveillance** :
1. Configurer un pare-feu strict (**UFW**) et bloquer les attaques par force brute avec **Fail2ban**.
2. Verrouiller la configuration SSH (interdiction du login root et des mots de passe bruts).
3. Automatiser les sauvegardes quotidiennes de PostgreSQL avec un script de rotation et **Cron**.
4. Maîtriser la matrice de diagnostic des **10 pannes de production les plus fréquentes**.
5. Valider l'examen final en toute autonomie.

---

# 1. 🛡️ Étape 1 : Durcissement de la sécurité du serveur (Server Hardening)

Connecte-toi à ton VPS en SSH :

```powershell
# 💻 [SUR TON PC]
ssh devops@IP_DE_TON_VPS
```

### 1.1 Configurer le Pare-feu UFW (Uncomplicated Firewall)

Par défaut, nous fermons tout et nous n'ouvrons **QUE** les trois ports vitaux :
- Port `22` : Connexion SSH d'administration.
- Port `80` : Trafic web HTTP (pour la redirection et le challenge Let's Encrypt).
- Port `443` : Trafic web chiffré HTTPS.

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Définir les règles par défaut : bloquer tout ce qui entre, autoriser ce qui sort
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 2. Autoriser expressément SSH, HTTP et HTTPS
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# 3. Activer le pare-feu
sudo ufw enable

# 4. Vérifier le statut
sudo ufw status verbose
```
*Désormais, tous les autres ports (comme 5432 pour PostgreSQL ou 8000 pour FastAPI) sont invisibles et inaccessibles depuis Internet !*

---

### 1.2 Protéger contre les attaques par force brute avec Fail2ban

Des robots scannent constamment Internet pour tenter des attaques par dictionnaire sur le port SSH. **Fail2ban** surveille les logs et bannit automatiquement l'adresse IP de tout attaquant après 3 tentatives infructueuses.

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Installer Fail2ban
sudo apt update && sudo apt install -y fail2ban

# 2. Créer une configuration locale de protection SSH
sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = 22
EOF

# 3. Redémarrer et activer Fail2ban
sudo systemctl enable --now fail2ban

# 4. Voir les adresses IP bannies en temps réel
sudo fail2ban-client status sshd
```

---

### 1.3 Verrouiller l'accès SSH

Nous désactivons la connexion directe par mot de passe et l'accès au compte `root` :

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo nano /etc/ssh/sshd_config
```

Vérifie ou modifie les lignes suivantes :
```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

> [!WARNING]
> **Règle de sécurité vitale :** Ne ferme **PAS** ta session SSH actuelle avant d'avoir testé !
> 1. Redémarre le démon SSH : `sudo systemctl restart ssh`
> 2. Ouvre un **nouvel onglet de terminal sur ton PC** et tente de te connecter : `ssh devops@IP_DE_TON_VPS`.
> 3. Si la connexion fonctionne, ton serveur est parfaitement sécurisé.

---

# 2. 💾 Étape 2 : Sauvegardes automatisées de la base de données

Un crash de disque, une mauvaise manipulation ou un conteneur corrompu peuvent détruire tes données. Nous mettons en place un script de sauvegarde automatique quotidienne.

### 2.1 Créer le script de sauvegarde

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Créer un dossier pour les scripts et les sauvegardes
mkdir -p ~/scripts ~/backups/postgres

# 2. Créer le script de backup
cat << 'EOF' > ~/scripts/backup_db.sh
#!/bin/bash
set -e

BACKUP_DIR="/home/devops/backups/postgres"
DATE=$(date +'%Y-%m-%d_%H%M%S')
FILENAME="$BACKUP_DIR/basket_db_$DATE.sql.gz"

echo "📦 [$(date)] Démarrage de la sauvegarde PostgreSQL..."

# Exporter et compresser la base depuis le conteneur Docker
docker compose -f /home/devops/basket-app/docker-compose.yml exec -T db pg_dump -U basket_user basket_db | gzip > "$FILENAME"

echo "✅ Sauvegarde réussie : $FILENAME (Taille : $(du -h $FILENAME | cut -f1))"

# Supprimer automatiquement les sauvegardes de plus de 7 jours (rotation)
find "$BACKUP_DIR" -type f -name "*.sql.gz" -mtime +7 -delete
echo "🧹 Nettoyage des anciennes sauvegardes effectué."
EOF

# Donner les droits d'exécution au script
chmod +x ~/scripts/backup_db.sh
```

### 2.2 Tester le script manuellement

```bash
# 🌐 [SUR LE VPS - Session SSH]
~/scripts/backup_db.sh
ls -lh ~/backups/postgres/
```

### 2.3 Programmer la sauvegarde chaque nuit avec Cron

Ouvre l'éditeur cron :
```bash
# 🌐 [SUR LE VPS - Session SSH]
crontab -e
```
*(Choisis `nano` si demandé, puis ajoute cette ligne tout en bas pour exécuter le backup tous les jours à 03h00 du matin) :*

```text
0 3 * * * /home/devops/scripts/backup_db.sh >> /home/devops/backups/postgres/backup.log 2>&1
```

---

# 3. 🩺 Étape 3 : La Matrice Ultime de Dépannage (Troubleshooting)

Quand un problème survient en production, **ne panique pas**. Suis cette méthode d'investigation en entonnoir :

``` text
1. Le serveur répond-il au ping / SSH ?
   │
2. Le pare-feu ou le DNS bloque-t-il ? (nslookup, ufw)
   │
3. Nginx est-il vivant ? (sudo systemctl status nginx, sudo nginx -t)
   │
4. Les conteneurs Docker tournent-ils ? (docker compose ps)
   │
5. Que disent les logs de l'application ? (docker compose logs -f api)
```

---

### Guide des 10 pannes courantes et leurs solutions immédiates :

| Problème / Symptôme | Cause probable | Commande de diagnostic | Solution |
| :--- | :--- | :--- | :--- |
| **`502 Bad Gateway`** | Nginx tourne mais le conteneur (FastAPI ou Next.js) est éteint ou ne répond pas. | `docker compose ps`<br>`docker compose logs -f api` | Relancer le conteneur :<br>`docker compose up -d` |
| **`504 Gateway Timeout`** | L'application met trop de temps à répondre (requête SQL bloquée ou boucle infinie). | `docker compose logs api`<br>`top` | Optimiser la requête SQL ou augmenter `proxy_read_timeout` dans Nginx. |
| **`Address already in use` (Port 80 ou 8000)** | Un autre processus écoute déjà sur ce port. | `sudo ss -tulpn \| grep :80` | Tuer le vieux processus :<br>`sudo kill -9 <PID>` ou arrêter le service concurrent. |
| **`CrashLoopBackOff` / Conteneur qui redémarre sans cesse** | Erreur au démarrage de l'application (variable `.env` manquante ou erreur de syntaxe). | `docker compose logs api` | Lire la dernière trace d'erreur et corriger le code ou le fichier `.env`. |
| **`could not connect to server: Connection refused`** | L'API tente de se connecter à PostgreSQL avant que la base ne soit prête. | `docker compose ps db`<br>`docker compose logs db` | Vérifier le bloc `healthcheck` et `depends_on` dans `docker-compose.yml`. |
| **`No space left on device` (Disque plein)** | Anciennes images Docker inutilisées et logs volumineux. | `df -h` | Nettoyer Docker :<br>`docker system prune -a --volumes -f` |
| **Certificat SSL Expiré ou Invalide** | Le renouvellement Certbot n'a pas pu s'exécuter. | `sudo certbot certificates` | Forcer le renouvellement :<br>`sudo certbot renew --force-renewal` |
| **`Permission Denied (publickey)` lors du déploiement GitHub** | La clé privée dans GitHub Secrets ne correspond pas à la clé publique dans `authorized_keys`. | Vérifier `~/.ssh/authorized_keys` sur le VPS. | Régénérer la paire de clés SSH et mettre à jour le secret `VPS_SSH_KEY`. |
| **`DNS_PROBE_FINISHED_NXDOMAIN`** | Mauvaise configuration DNS chez le registrar. | `nslookup ton-domaine.com` | Vérifier l'enregistrement de type `A` vers l'IP du VPS. |
| **Conteneur tué subitement (`Exit Code 137`)** | Le serveur manque de RAM et l'OS a tué le processus (**OOM Killer**). | `free -h`<br>`dmesg \| grep -i oom` | Ajouter un fichier d'échange (Swap) :<br>`sudo fallocate -l 2G /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |

---

# 4. 🎓 Examen Final : Le Grand Chelem DevOps

Pour valider l'ensemble de ta semaine, tente de reproduire l'exercice suivant **en autonomie totale** :

1. Ajoute une nouvelle fonctionnalité sur ton PC : un endpoint `GET /stats` qui renvoie le nombre total de matchs et le score moyen.
2. Écris un test unitaire `test_stats_endpoint` dans `api/tests/test_api.py`.
3. Pousse sur GitHub avec `git push origin main`.
4. Observe GitHub Actions exécuter les tests, se connecter au VPS et mettre à jour l'application en direct.
5. Visite ton site en HTTPS et constate la mise à jour sans avoir ouvert de terminal SSH !

---

# 🏆 Félicitations !

Tu maîtrises désormais toute la chaîne moderne du développement au déploiement en production :
$$\textbf{Linux} \longrightarrow \textbf{SSH} \longrightarrow \textbf{Docker} \longrightarrow \textbf{Nginx (HTTPS)} \longrightarrow \textbf{GitHub Actions (CI/CD)}$$

Garde précieusement ce dépôt, consulte la **[Cheat Sheet](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/08-CHEATSHEET.md)** et le **[Glossaire](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/09-GLOSSAIRE.md)** dès que tu as un doute sur un futur projet !
