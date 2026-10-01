# 🚀 Roadmap --- VPS, Docker & GitHub Actions en 7 Jours

Bienvenue dans la roadmap pratique pour apprendre à déployer et automatiser vos applications web comme un pro !

---

## 🎯 Objectif du parcours

En 7 jours, tu vas construire, conteneuriser, sécuriser et déployer une application web complète sur un **VPS Linux** avec un pipeline **CI/CD GitHub Actions**.

### Ce que tu vas maîtriser

- 🐧 **Linux & Terminal** : Naviguer, gérer les processus et les permissions en ligne de commande.
- 🔑 **SSH** : Se connecter à distance et sécuriser l'accès avec des paires de clés asymétriques.
- 📦 **Docker & Docker Compose** : Conteneuriser une stack complète (Frontend Next.js + Backend FastAPI + Base PostgreSQL).
- 🌐 **Nginx & Reverse Proxy** : Router le trafic, gérer les noms de domaine et activer le chiffrement **HTTPS (SSL/TLS)** avec Let's Encrypt.
- 🔄 **GitHub Actions (CI/CD)** : Automatiser les tests et le déploiement continu à chaque `git push`.
- 🛡️ **Sécurité & Monitoring** : Pare-feu (UFW), gestion des secrets, logs en temps réel et dépannage d'incidents de production.

> [!NOTE]
> Le but n'est **pas** de mémoriser 500 commandes par cœur. Le but est de **comprendre l'architecture**, savoir **où taper quoi**, et être capable de diagnostiquer et réparer un problème en autonomie.

---

## 🛠️ Boîte à outils : Que dois-je installer pour commencer ?

Avant de commencer le Jour 1, installe les outils suivants sur ton **ordinateur personnel (PC Windows / macOS / Linux)**.

### 1. Sur ton PC (Environnement de développement local)

| Outil | Rôle | Lien de téléchargement | Vérification dans le terminal |
| :--- | :--- | :--- | :--- |
| **VS Code** | Éditeur de code complet et léger | [Télécharger VS Code](https://code.visualstudio.com/) | `code --version` |
| **Git for Windows** | Gestionnaire de versions + Terminal Git Bash | [Télécharger Git](https://git-scm.com/download/win) | `git --version` |
| **Windows Terminal** | Terminal moderne multi-onglets (recommandé) | [Microsoft Store](https://aka.ms/terminal) | Ouvrir via le menu Démarrer |
| **Node.js (v20 LTS)** | Exécution de JavaScript/Next.js & npm | [Télécharger Node.js](https://nodejs.org/) | `node -v` et `npm -v` |
| **Python (3.11+)** | Langage pour FastAPI & uvicorn | [Télécharger Python](https://www.python.org/downloads/) | `python --version` |
| **Client SSH** | Inclus par défaut dans Windows 10/11 & macOS | Intégré à PowerShell / Terminal | `ssh -V` |
| **Docker Desktop** *(Optionnel en local)* | Pour tester les conteneurs sur ton PC | [Télécharger Docker Desktop](https://www.docker.com/products/docker-desktop/) | `docker --version` |

> [!TIP]
> **Conseil Windows** : Lors de l'installation de Git, coche bien l'option pour installer **Git Bash**. C'est un terminal Linux-like très pratique sous Windows.

---

### 2. Services Cloud & Hébergement (Pour la mise en production)

Tu auras besoin de ces services à partir du **Jour 2** :

1. **Un compte GitHub** (Gratuit) : [github.com](https://github.com/) pour héberger ton code et exécuter GitHub Actions.
2. **Un serveur VPS Linux (Ubuntu 22.04 ou 24.04 LTS)** :
   - *Fournisseurs recommandés (environ 3€ à 6€/mois, résiliable à tout moment)* :
     - [Hetzner Cloud](https://www.hetzner.com/cloud) (Excellent rapport perf/prix, serveur CX22 ~4€/mois)
     - [OVHcloud](https://www.ovhcloud.com/fr/vps/) (Hébergeur français, VPS Starter ~4€/mois)
     - [DigitalOcean](https://www.digitalocean.com/) (Droplet Basic 4-6$/mois)
     - [Scaleway](https://www.scaleway.com/) (Serveurs Stardust / DEV)
3. **Un nom de domaine** (Recommandé au Jour 4, ~5€/an ou gratuit via sous-domaine/DuckDNS) :
   - [Cloudflare Registrar](https://www.cloudflare.com/) ou [Namecheap](https://www.namecheap.com/) ou [OVHcloud](https://www.ovhcloud.com/).
   - *Option 100% gratuite pour s'entraîner* : [DuckDNS](https://www.duckdns.org/) (fournit un sous-domaine gratuit pointant vers l'IP de ton VPS).

---

## 🧭 Comment lire ce guide : La légende "Où taper les commandes ?"

L'une des plus grandes difficultés quand on débute en DevOps est de savoir **sur quelle machine** on se trouve. Pour éviter toute confusion, chaque bloc de code dans ce tutoriel commence par un badge explicite :

- 💻 **`[SUR TON PC - PowerShell ou Git Bash]`** : Commande à taper dans ton terminal local sur ton propre ordinateur.
- 🌐 **`[SUR LE VPS - Session SSH]`** : Commande à taper dans le terminal du serveur distant Linux après t'être connecté en SSH.
- 📁 **`[DANS TON ÉDITEUR - Fichier local ou distant]`** : Code source d'un fichier à créer ou modifier dans VS Code.
- 🐙 **`[SUR GITHUB - Interface Web]`** : Action à faire dans les paramètres de ton dépôt sur GitHub (ex: ajouter un secret).

---

## 🏗️ Projet fil rouge : "Mini Basket Live API + Dashboard"

Tout au long des 7 jours, nous construisons une application concrète de suivi de matchs de basketball :

``` text
                         INTERNET (Utilisateur)
                                  │
                                  ▼
                     ┌─────────────────────────┐
                     │   NGINX (Port 80 / 443) │
                     │  SSL / HTTPS Let's Encrypt
                     └────────────┬────────────┘
                                  │
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
    ┌──────────────────┐                    ┌──────────────────┐
    │ Frontend Next.js │                    │ Backend FastAPI  │
    │ (Dashboard web)  │ ─── requêtes HTTP ─▶ │ (API REST JSON)  │
    │   Port interne   │                    │   Port interne   │
    └──────────────────┘                    └─────────┬────────┘
                                                      │
                                                      ▼
                                            ┌──────────────────┐
                                            │    PostgreSQL    │
                                            │ (Base de données)│
                                            └──────────────────┘
```

Et nous mettons en place le pipeline de déploiement continu automatisé :

``` text
[💻 Ton PC]                      [🐙 GitHub]                    [🌐 Ton VPS Linux]
  git push main  ───────────▶  GitHub Actions CI/CD  ────────▶  Pull du code
                               (Lint, Tests, Build)             Docker Compose rebuild
                                                                Restart sans coupure
```

---

## 📅 Planning de la semaine

| Jour | Thème | Compétences pratiques | Résultat tangible |
| :---: | :--- | :--- | :--- |
| **[01](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/01-JOUR-1-LINUX-SSH-GIT.md)** | **Linux, SSH & Git** | Terminal, commandes bash, clés SSH, GitHub | Savoir administrer un système et versionner son code |
| **[02](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/02-JOUR-2-VPS-DEPLOIEMENT-MANUEL.md)** | **VPS & Déploiement Manuel** | Créer un VPS, sécuriser un user sudo, FastAPI, systemd | API accessible en ligne via `http://IP:8000` |
| **[03](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/03-JOUR-3-DOCKER.md)** | **Docker & Conteneurs** | Dockerfile, Docker Compose, volumes, networking | API + Base de données isolées dans Docker |
| **[04](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/04-JOUR-4-NGINX-DOMAINE-HTTPS.md)** | **Nginx, Domaine & HTTPS** | Reverse proxy, DNS A Record, Certbot / SSL | `https://api.ton-domaine.com` sécurisé par cadenas vert |
| **[05](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/05-JOUR-5-STACK-COMPLETE.md)** | **Stack Complète (Next+FastAPI+PG)** | Frontend Next.js, Docker multi-stage, variables `.env` | Application web complète connectée de bout en bout |
| **[06](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/06-JOUR-6-GITHUB-ACTIONS-CICD.md)** | **GitHub Actions (CI/CD)** | Workflows YAML, GitHub Secrets, déploiement SSH auto | Déploiement automatique à chaque `git push` |
| **[07](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/07-JOUR-7-PROJET-FINAL-SECURITE.md)** | **Sécurité & Dépannage** | Pare-feu UFW, Fail2ban, backups auto, diagnostic de pannes | Serveur durci et maîtrise totale des pannes |

---

## 💡 Règle d'or pour réussir

Pour chaque concept :
$$\textbf{Comprendre} \longrightarrow \textbf{Pratiquer} \longrightarrow \textbf{Casser volontairement} \longrightarrow \textbf{Réparer} \longrightarrow \textbf{Expliquer}$$

Prends ton temps, tape les commandes une à une au lieu de copier-coller des blocs entiers, et observe ce qui s'affiche à l'écran !

Prêt ? Ouvre le **[Jour 1 : Linux, SSH et Git](file:///c:/Users/HP/Documents/Projet%20Perso/roadmap_vps_github_actions/01-JOUR-1-LINUX-SSH-GIT.md)** pour démarrer !
