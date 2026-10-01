# 📖 Glossaire DevOps --- Termes & Concepts Clés

Ce glossaire regroupe toutes les notions fondamentales rencontrées lors du déploiement d'applications web, expliquées simplement avec des analogies concrètes.

---

### 🌐 1. Infrastructure & Réseau

- **VPS (Virtual Private Server)** : Un serveur virtuel loué dans un centre de données qui tourne en permanence avec sa propre adresse IP publique.
- **Adresse IP (Internet Protocol)** : L'identifiant numérique unique d'une machine sur un réseau (ex: `198.51.100.42`). *(Analogie : L'adresse postale d'un immeuble).*
- **Port Réseau** : Un numéro (de 1 à 65535) qui identifie une application spécifique sur un serveur. *(Analogie : Le numéro d'appartement ou la porte d'entrée).*
  - `22` : SSH (Accès terminal sécurisé)
  - `80` : HTTP (Web non chiffré)
  - `443` : HTTPS (Web chiffré SSL)
  - `3000` : Port habituel de Next.js
  - `5432` : Port par défaut de PostgreSQL
  - `8000` : Port habituel de FastAPI / Uvicorn
- **DNS (Domain Name System)** : Le carnet d'adresses d'Internet qui traduit un nom de domaine lisible (`api.mondomaine.com`) en une adresse IP (`198.51.100.42`).
  - **Enregistrement A (Address Record)** : Associe directement un nom de domaine à une adresse IPv4.
  - **CNAME (Canonical Name)** : Crée un alias qui pointe vers un autre nom de domaine.
- **Reverse Proxy** : Un serveur intermédiaire (ex: Nginx) placé devant les applications internes pour recevoir les requêtes des clients, gérer le chiffrement HTTPS et router vers le bon service. *(Analogie : Le réceptionniste d'un grand hôtel).*
- **SSL / TLS & HTTPS** : Protocole cryptographique qui chiffre les échanges entre le navigateur du client et le serveur pour empêcher toute interception de mots de passe ou données privées.
- **Pare-feu (Firewall / UFW)** : Logiciel de sécurité qui filtre et bloque les connexions réseau non autorisées.

---

### 🔑 2. Sécurité & Accès

- **SSH (Secure Shell)** : Protocole de communication chiffré permettant d'ouvrir un terminal à distance sur un serveur Linux.
- **Clé Asymétrique (Paire de clés SSH)** :
  - **Clé Publique (`.pub`)** : La serrure. Peut être transmise publiquement à GitHub ou copiée sur le serveur dans `authorized_keys`.
  - **Clé Privée** : La clé physique secrète. Ne doit **jamais** quitter votre ordinateur ni être partagée sur Git.
- **Root** : Le super-administrateur tout-puissant sous Linux (équivalent de `SYSTEM` ou `Administrateur` sous Windows).
- **Sudo (SuperUser DO)** : Commande permettant à un utilisateur standard autorisé d'exécuter une action ponctuelle avec les privilèges administrateur.
- **Secrets d'environnement (`.env`)** : Variables contenant des données sensibles (clés API, mots de passe de base de données) qui ne doivent jamais être commitées dans Git.

---

### 🐳 3. Conteneurisation & Docker

- **Conteneur** : Environnement d'exécution isolé et léger qui embarque une application et toutes ses dépendances sans émuler un système d'exploitation complet.
- **Image Docker** : Le modèle en lecture seule (blueprint) utilisé pour créer un conteneur.
- **`Dockerfile`** : Le fichier texte contenant la suite d'instructions pour fabriquer une image Docker personnalisée.
- **Docker Compose** : Outil permettant de définir et d'exécuter des applications multi-conteneurs à l'aide d'un simple fichier YAML (`docker-compose.yml`).
- **Volume Docker** : Espace de stockage persistant géré par Docker sur le disque de la machine hôte, indépendant du cycle de vie des conteneurs.
- **Mapping de Ports (`-p HOTE:CONTENEUR`)** : Directive reliant un port accessible sur la machine hôte à un port interne du conteneur (ex: `-p 8000:8000`).

---

### 🔄 4. Intégration & Déploiement Continus (CI/CD)

- **CI (Continuous Integration)** : Pratique consistant à automatiser le test et la validation du code à chaque fois qu'un développeur pousse une modification.
- **CD (Continuous Delivery / Continuous Deployment)** : Automatisation de la mise en production du code validé directement sur le serveur sans intervention humaine.
- **GitHub Actions** : Plateforme d'automatisation intégrée à GitHub permettant de créer des pipelines (Workflows) déclenchés par des événements Git (`push`, `pull_request`).
- **Workflow YAML** : Fichier de configuration décrivant les étapes automatisées à exécuter.
- **GitHub Runner** : Machine virtuelle temporaire hébergée par GitHub qui exécute les scripts de vos workflows.
- **GitHub Secrets** : Coffre-fort chiffré sur GitHub permettant de stocker des clés SSH ou des tokens d'accès sans les afficher dans les logs.
- **Health Check** : Requête automatique (ex: `GET /health`) envoyée après un déploiement pour s'assurer que le service fonctionne correctement.
