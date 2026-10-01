# 🐳 Jour 3 --- Docker et Docker Compose

## 🎯 Objectif du jour

À la fin de cette journée, tu sauras :
1. Comprendre la différence exacte entre une **Machine Virtuelle** et un **Conteneur Docker**.
2. Installer Docker Engine et le plugin Docker Compose officiel sur ton VPS.
3. Écrire un `Dockerfile` propre, sécurisé et optimisé pour une API FastAPI.
4. Construire une image Docker, lancer un conteneur et comprendre le mapping de ports (`-p 8000:8000`).
5. Orchestrer plusieurs conteneurs (FastAPI + Base de données PostgreSQL) avec un seul fichier `docker-compose.yml`.
6. Assurer la persistance des données grâce aux **Volumes Docker**.

---

# 1. 🧠 La théorie : Pourquoi Docker a révolutionné l'informatique ?

### Le problème historique : *"Mais pourtant ça marche sur ma machine !"*
Sans Docker, chaque serveur doit avoir exactement les mêmes versions de Python, de librairies C, et de pilotes que ton PC. Le moindre décalage de version brise l'application.

``` text
[ Machine Virtuelle (Lourd : 2-10 Go) ]       [ Conteneur Docker (Léger : 50-200 Mo) ]
┌──────────────────────────────────────┐       ┌──────────────────────────────────────┐
│  App A       │  App B                │       │  App A       │  App B                │
│  Lib A       │  Lib B                │       │  Lib A       │  Lib B                │
├──────────────┴───────────────────────┤       ├──────────────┴───────────────────────┤
│  OS Invité Complet (ex: Ubuntu 2 Go) │       │  Moteur Docker (Processus isolé)     │
├──────────────────────────────────────┤       ├──────────────────────────────────────┤
│  Hyperviseur (VirtualBox, KVM...)    │       │  Noyau Linux partagé de l'hôte (OS)  │
└──────────────────────────────────────┘       └──────────────────────────────────────┘
```

### Le vocabulaire Docker démystifié :

| Terme | Analogie | Définition technique |
| :--- | :--- | :--- |
| **`Dockerfile`** | La **recette de cuisine** | Fichier texte listant les instructions pas-à-pas pour assembler l'environnement. |
| **Image Docker** | Le **plat surgelé / moule** | Un paquet immuable contenant l'OS minimal, les dépendances et ton code. |
| **Conteneur** | Le **plat chaud prêt à servir** | Une instance vivante et isolée de ton image qui exécute ton application. |
| **Volume** | Le **disque dur externe** | Un dossier sur le serveur hôte qui conserve les données même si le conteneur est détruit. |

---

# 2. 🛠️ Étape 1 : Installer Docker Engine & Docker Compose sur le VPS

Connecte-toi à ton VPS en SSH :

```powershell
# 💻 [SUR TON PC]
ssh devops@IP_DE_TON_VPS
```

### 1.1 Nettoyer les anciens paquets et configurer le dépôt officiel Docker

```bash
# 🌐 [SUR LE VPS - Session SSH]

# Mettre à jour les paquets requis pour HTTPS
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Ajouter la clé GPG officielle de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajouter le dépôt officiel Docker aux sources APT
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installer Docker Engine, le CLI et le plugin Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 1.2 Autoriser ton utilisateur `devops` à utiliser Docker sans `sudo`

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo usermod -aG docker devops

# Appliquer le nouveau groupe immédiatement
newgrp docker

# Vérifier que Docker fonctionne sans sudo
docker --version
docker compose version
```

---

# 3. 🧹 Étape 2 : Libérer le port 8000

Au Jour 2, nous avions lancé un service systemd sur le port 8000. Pour que Docker puisse utiliser ce port, désactivons l'ancien service :

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo systemctl stop basket-api
sudo systemctl disable basket-api
```

---

# 4. 📄 Étape 3 : Créer le Dockerfile de l'API

Rends-toi dans le dossier de ton API :

```bash
# 🌐 [SUR LE VPS - Session SSH]
cd ~/basket-app/api
```

Crée le fichier `Dockerfile` :

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > Dockerfile
# 1. Image de base légère avec Python 3.12
FROM python:3.12-slim

# 2. Définir le répertoire de travail dans le conteneur
WORKDIR /app

# 3. Empêcher Python d'écrire des fichiers .pyc et activer les logs immédiats
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# 4. Copier et installer uniquement les dépendances en premier (pour exploiter le cache Docker)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 5. Créer un utilisateur non-privilégié pour la sécurité
RUN useradd -m appuser && chown -R appuser /app
USER appuser

# 6. Copier le reste du code de l'application
COPY . .

# 7. Documenter le port utilisé
EXPOSE 8000

# 8. Commande exécutée au démarrage du conteneur
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

---

# 5. 🏗️ Étape 4 : Construire l'image et démarrer le conteneur

### 5.1 Construire l'image Docker (`docker build`)

```bash
# 🌐 [SUR LE VPS - Session SSH]
# Le point '.' à la fin signifie : "cherche le Dockerfile dans le dossier actuel"
docker build -t basket-api:1.0 .
```

### 5.2 Lancer le conteneur (`docker run`)

```bash
# 🌐 [SUR LE VPS - Session SSH]
docker run -d \
  --name basket-api-container \
  -p 8000:8000 \
  --restart always \
  basket-api:1.0
```

> **Décryptage des arguments :**
> - `-d` (*detached*) : Tourne en arrière-plan (ne bloque pas le terminal).
> - `--name basket-api-container` : Donne un nom lisible au conteneur.
> - `-p 8000:8000` : Relie le port `8000` du serveur VPS au port `8000` interne du conteneur.
> - `--restart always` : Redémarre automatiquement le conteneur si le VPS redémarre ou si le processus crash.

### 5.3 Vérifier le conteneur et inspecter ses logs

```bash
# 🌐 [SUR LE VPS - Session SSH]

# Voir les conteneurs actifs
docker ps

# Consulter les logs en direct (Ctrl+C pour quitter)
docker logs -f basket-api-container

# Tester la réponse avec curl
curl http://localhost:8000/health
```

---

# 6. 🎼 Étape 5 : Multi-conteneurs avec Docker Compose (API + PostgreSQL)

Une application réelle a besoin d'une base de données. Gérer plusieurs `docker run` à la main devient ingérable. **Docker Compose** permet de décrire toute la stack dans un seul fichier `docker-compose.yml`.

### 6.1 Créer l'arborescence du projet

Place-toi à la racine de ton application :

```bash
# 🌐 [SUR LE VPS - Session SSH]
cd ~/basket-app
```

### 6.2 Créer le fichier d'environnement `.env`

Ce fichier contient les mots de passe et configurations secrètes :

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > .env
# Configuration PostgreSQL
POSTGRES_DB=basket_db
POSTGRES_USER=basket_user
POSTGRES_PASSWORD=super_secret_password_123

# URL de connexion transmise à l'API
DATABASE_URL=postgresql://basket_user:super_secret_password_123@db:5432/basket_db
EOF
```

### 6.3 Créer le fichier `docker-compose.yml`

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > docker-compose.yml
services:
  # Service Backend (Notre API FastAPI)
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    container_name: basket_api
    restart: always
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - basket_network

  # Service Base de données (PostgreSQL 16 officiel)
  db:
    image: postgres:16-alpine
    container_name: basket_postgres
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      # Persistance des données sur le disque du VPS
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - basket_network

volumes:
  postgres_data:
    driver: local

networks:
  basket_network:
    driver: bridge
EOF
```

---

# 7. 🚀 Étape 6 : Lancer et piloter la stack complète

Avant de lancer Compose, supprimons notre conteneur isolé de test pour libérer le port :

```bash
# 🌐 [SUR LE VPS - Session SSH]
docker stop basket-api-container && docker rm basket-api-container
```

### Lancer la stack avec Docker Compose :

```bash
# 🌐 [SUR LE VPS - Session SSH]
# Démarrer tous les services en tâche de fond (-d)
docker compose up -d --build
```

### Commandes indispensables pour administrer Compose :

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Voir l'état de santé de tous les services de la stack
docker compose ps

# 2. Voir les logs de l'API en temps réel
docker compose logs -f api

# 3. Voir les logs de PostgreSQL
docker compose logs -f db

# 4. Exécuter une commande SQL directement dans le conteneur PostgreSQL
docker compose exec db psql -U basket_user -d basket_db -c "SELECT current_database(), current_user;"
```

---

# 8. 💥 Exercice "Casser & Réparer" : Tester la persistance des données

**Scénario de test :** Prouver que la destruction des conteneurs n'efface pas les données de la base grâce au volume.

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Créons une table de test dans PostgreSQL
docker compose exec db psql -U basket_user -d basket_db -c "CREATE TABLE test_scores (id SERIAL PRIMARY KEY, team VARCHAR(50)); INSERT INTO test_scores (team) VALUES ('Chicago Bulls');"

# 2. Vérifions que la ligne existe
docker compose exec db psql -U basket_user -d basket_db -c "SELECT * FROM test_scores;"

# 3. Arrêtons et supprimons TOUS les conteneurs de la stack
docker compose down

# 4. Vérifions que plus rien ne tourne
docker compose ps

# 5. Relançons la stack
docker compose up -d

# 6. Vérifions si les données sont toujours là
docker compose exec db psql -U basket_user -d basket_db -c "SELECT * FROM test_scores;"
# 🎉 La ligne 'Chicago Bulls' est toujours présente grâce au volume nommé 'postgres_data' !
```

---

# ✅ Checklist de validation - Jour 3

- [ ] Docker Engine et Docker Compose sont installés et fonctionnels sans `sudo`.
- [ ] J'ai écrit un `Dockerfile` optimisé pour mon API FastAPI.
- [ ] Je comprends la signification de `-p 8000:8000` (Port Hôte : Port Conteneur).
- [ ] J'ai orchestré l'API et PostgreSQL avec `docker-compose.yml`.
- [ ] J'ai testé l'accès aux logs avec `docker compose logs -f`.
- [ ] J'ai validé la persistance des données grâce aux volumes Docker.

👉 **Demain (Jour 4) :** On arrête d'utiliser l'adresse IP brute avec le port 8000 ! Nous allons configurer un **nom de domaine**, un **reverse proxy Nginx** et un certificat **HTTPS (SSL)** sécurisé !
