# 📑 Aide-Mémoire (Cheat Sheet) --- VPS, Docker & CI/CD

Retrouve ici toutes les commandes indispensables, classées par domaine et avec l'indication exacte de l'environnement d'exécution.

---

## 💻 1. Commandes sur ton PC (Windows / PowerShell / Git Bash)

### Git & Gestion de Versions
```bash
# État et différences
git status                       # Voir les fichiers modifiés et en attente
git diff                         # Voir le détail des lignes modifiées

# Cycle standard d'enregistrement
git add .                        # Ajouter tous les fichiers modifiés
git commit -m "feat: mon message" # Enregistrer le commit
git push origin main             # Pousser sur la branche distante
git pull origin main             # Récupérer les nouveautés distantes

# Branches
git checkout -b feature/nom      # Créer et basculer sur une nouvelle branche
git branch -a                    # Lister toutes les branches
git merge feature/nom            # Fusionner une branche dans la branche actuelle
```

### SSH & Connexion
```bash
# Générer une clé SSH ultra-sécurisée Ed25519
ssh-keygen -t ed25519 -C "ton.email@example.com"

# Afficher la clé publique (à copier)
cat ~/.ssh/id_ed25519.pub

# Tester la liaison avec GitHub
ssh -T git@github.com

# Se connecter à ton serveur VPS
ssh devops@IP_DE_TON_VPS
```

---

## 🌐 2. Commandes sur le Serveur VPS (Session SSH)

### Navigation & Fichiers Linux
```bash
pwd                              # Afficher le dossier courant
ls -la                           # Lister tous les fichiers avec permissions et tailles
cd /chemin/vers/dossier          # Changer de dossier (cd .. pour remonter)
mkdir -p dossier/sous-dossier    # Créer un dossier récursivement
touch fichier.txt                # Créer un fichier vide
cat fichier.txt                  # Afficher tout le fichier
tail -n 50 -f fichier.log        # Suivre les 50 dernières lignes d'un fichier de log en direct
nano fichier.txt                 # Éditer un fichier (Ctrl+O pour sauver, Ctrl+X pour quitter)
rm -rf dossier                   # Supprimer un dossier et son contenu (ATTENTION)
```

### Surveillance Système & Ressources
```bash
df -h                            # Voir l'espace disque disponible
free -h                          # Voir la mémoire RAM libre et utilisée
top  # ou htop                   # Gestionnaire des processus en temps réel (q pour quitter)
sudo ss -tulpn                   # Lister tous les ports ouverts et les processus associés
```

### Gestion des Services Système (systemd)
```bash
sudo systemctl status nom_service   # Voir l'état d'un service
sudo systemctl start nom_service    # Démarrer un service
sudo systemctl stop nom_service     # Arrêter un service
sudo systemctl restart nom_service  # Redémarrer un service
sudo systemctl enable --now service # Démarrer et activer au démarrage de la machine
journalctl -u nom_service -f        # Voir les logs du service en direct
```

---

## 🐳 3. Docker & Docker Compose

### Gestion des Conteneurs & Images
```bash
docker ps                        # Lister les conteneurs en cours d'exécution
docker ps -a                     # Lister TOUS les conteneurs (même arrêtés)
docker images                    # Lister les images locales
docker logs -f nom_conteneur     # Suivre les logs d'un conteneur
docker exec -it nom_conteneur sh # Entrer dans le conteneur en ligne de commande
docker stop nom_conteneur        # Arrêter un conteneur
docker rm nom_conteneur          # Supprimer un conteneur arrêté
docker rmi nom_image             # Supprimer une image
docker system prune -a --volumes # NETTOYAGE COMPLET (images, conteneurs et volumes orphelins)
```

### Docker Compose (Stack multi-conteneurs)
```bash
docker compose up -d             # Démarrer toute la stack en arrière-plan
docker compose up -d --build     # Reconstruire les images modifiées et relancer
docker compose down              # Arrêter et supprimer tous les conteneurs de la stack
docker compose down -v           # Arrêter et supprimer aussi les volumes (ATTENTION aux données)
docker compose ps                # Voir l'état des services de la stack
docker compose logs -f api       # Voir les logs d'un service spécifique (ex: api)
docker compose restart api       # Redémarrer un seul service sans toucher aux autres
docker compose exec db psql -U user -d db # Exécuter une commande dans un service
```

---

## 🌐 4. Nginx, Certbot & Sécurité

### Nginx
```bash
sudo nginx -t                    # TESTER LA SYNTAXE DE LA CONFIGURATION (INDISPENSABLE)
sudo systemctl reload nginx      # Recharger la configuration sans couper le trafic
sudo systemctl restart nginx     # Redémarrer Nginx
sudo tail -f /var/log/nginx/error.log  # Voir les logs d'erreurs Nginx
```

### Certbot (Certificats HTTPS)
```bash
sudo certbot --nginx -d domaine.com -d api.domaine.com # Générer et installer les certificats
sudo certbot renew --dry-run                           # Tester la simulation de renouvellement
sudo certbot certificates                              # Voir la liste des certificats actifs
```

### Pare-feu UFW & Fail2ban
```bash
sudo ufw status verbose          # Voir les règles du pare-feu
sudo ufw allow 80/tcp            # Ouvrir le port HTTP
sudo ufw allow 443/tcp           # Ouvrir le port HTTPS
sudo fail2ban-client status sshd # Voir les IPs bannies pour tentatives d'intrusion
sudo fail2ban-client set sshd unbanip IP_VICTIME # Débannir une adresse IP
```
