# 🚀 Jour 2 --- VPS et Premier Déploiement Manuel

## 🎯 Objectif du jour

À la fin de cette journée, tu sauras :
1. Louer et initialiser un serveur Linux VPS (Ubuntu 22.04 / 24.04 LTS).
2. T'y connecter en SSH depuis ton PC.
3. Sécuriser l'accès initial en créant un utilisateur dédié avec droits `sudo`.
4. Installer l'environnement Python et exécuter une API FastAPI en production.
5. Automatiser le démarrage de ton application en tâche de fond avec **systemd**.
6. Comprendre le routage réseau : Adresses IP, Ports (8000, 80, 22) et interfaces (`0.0.0.0` vs `127.0.0.1`).

---

# 1. 🧠 La théorie : Qu'est-ce qu'un VPS ?

Un **VPS (Virtual Private Server)** est une machine virtuelle hébergée dans un centre de données (Datacenter).

``` text
[ Centre de Données Cloud ]
  └─ Serveur Physique Puissant (Hyperviseur)
       ├─ [VPS Client A] (Linux Debian)
       ├─ [VPS Client B] (Windows Server)
       └─ [TON VPS] (Ubuntu Linux 24.04 LTS) ── Adresse IP Publique Fixe (ex: 123.45.67.89)
```

- **Avantage clé** : Il possède sa propre adresse IP publique fixe sur Internet, tourne 24h/24 et ne s'éteint jamais (même quand ton PC personnel est éteint).

---

# 2. 🛒 Étape 1 : Obtenir un VPS

Tu peux choisir n'importe quel hébergeur cloud :
- **Hetzner Cloud** : [hetzner.com](https://www.hetzner.com/cloud) (Recommandé : Serveur CX22 / CPX11, ~4€/mois).
- **OVHcloud** : [ovhcloud.com](https://www.ovhcloud.com/fr/vps/) (VPS Starter, ~4€/mois).
- **DigitalOcean** : [digitalocean.com](https://www.digitalocean.com/) (Droplet Basic, 4-6$/mois).

> [!IMPORTANT]
> **Configuration à choisir lors de la création :**
> - **Système d'exploitation (OS)** : **Ubuntu 22.04 LTS** ou **Ubuntu 24.04 LTS** (64 bits).
> - **Authentification** : Choisis ta clé publique SSH (`id_ed25519.pub`) si l'interface le propose, sinon choisis un mot de passe temporaire fourni par email.
> - Note l'**Adresse IP publique** de ton VPS (ex: `198.51.100.42`).

---

# 3. 🔑 Étape 2 : Première connexion SSH depuis ton PC

Ouvre **PowerShell** ou **Git Bash** sur ton ordinateur :

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
ssh root@IP_DE_TON_VPS
```
*(Remplace `IP_DE_TON_VPS` par l'adresse IP réelle de ton serveur, ex: `ssh root@198.51.100.42`)*

Lors de la toute première connexion, SSH te demande confirmation de l'empreinte de la machine :
`Are you sure you want to continue connecting (yes/no/[fingerprint])?` $\rightarrow$ Tape **`yes`** puis appuie sur **Entrée**.

> 🎉 **Tu es maintenant connecté au cœur de ton serveur Linux distant !**
> Remarque comme l'invite de commande a changé : `root@ubuntu-server:~#`.

---

# 4. 🛡️ Étape 3 : Sécurisation initiale (Créer un utilisateur non-root)

Travailler en permanence avec le compte `root` est dangereux : une seule commande erronée peut détruire le système. Nous allons créer un utilisateur administrateur nommé `devops`.

```bash
# 🌐 [SUR LE VPS - En tant que root]

# 1. Créer le nouvel utilisateur 'devops' (choisis un mot de passe fort quand demandé)
adduser devops

# 2. Donner les droits d'administration (sudo) à cet utilisateur
usermod -aG sudo devops

# 3. Copier la clé SSH du compte root vers le nouvel utilisateur devops
mkdir -p /home/devops/.ssh
cp /root/.ssh/authorized_keys /home/devops/.ssh/
chown -R devops:devops /home/devops/.ssh
chmod 700 /home/devops/.ssh
chmod 600 /home/devops/.ssh/authorized_keys
```

### Tester la connexion avec le nouvel utilisateur :

1. Déconnecte-toi du VPS :
```bash
# 🌐 [SUR LE VPS]
exit
```

2. Reconnecte-toi depuis ton PC en tant que `devops` :
```powershell
# 💻 [SUR TON PC]
ssh devops@IP_DE_TON_VPS
```

> Tu es désormais connecté en tant que `devops@ubuntu-server:~$` ! Pour exécuter une commande administrateur, il suffira de préfixer par `sudo`.

---

# 5. 📦 Étape 4 : Mettre à jour et préparer l'environnement

Exécute les mises à jour et installe Python et Git :

```bash
# 🌐 [SUR LE VPS - En tant que devops]

# 1. Mettre à jour la liste des paquets et le système
sudo apt update && sudo apt upgrade -y

# 2. Installer Python, l'outil d'environnement virtuel, Git et curl
sudo apt install -y python3 python3-pip python3-venv git curl ufw

# 3. Vérifier les versions installées
python3 --version
git --version
```

---

# 6. 🏀 Étape 5 : Créer l'API FastAPI sur le VPS

Nous allons créer un dossier pour notre application, un environnement virtuel Python (`venv`), et le code de notre API.

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Créer le dossier du projet et entrer dedans
mkdir -p ~/basket-app/api
cd ~/basket-app/api

# 2. Créer l'environnement virtuel Python
python3 -m venv .venv

# 3. Activer l'environnement virtuel
source .venv/bin/activate
```
*Remarque le préfixe `(.venv)` qui apparaît devant ta ligne de commande.*

### Étape 5.1 : Créer le fichier des dépendances `requirements.txt`

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > requirements.txt
fastapi==0.111.0
uvicorn[standard]==0.30.1
EOF

# Installer les dépendances
pip install -r requirements.txt
```

### Étape 5.2 : Créer le code source `main.py`

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > main.py
from fastapi import FastAPI
from datetime import datetime

app = FastAPI(title="Basket Live API", version="1.0.0")

# Base de données temporaire en mémoire
MATCHES = [
    {"id": 1, "home_team": "Lakers", "away_team": "Celtics", "home_score": 102, "away_score": 98, "status": "Finished"},
    {"id": 2, "home_team": "Warriors", "away_team": "Bulls", "home_score": 88, "away_score": 91, "status": "Live"},
    {"id": 3, "home_team": "Spurs", "away_team": "Heat", "home_score": 75, "away_score": 70, "status": "Scheduled"},
]

@app.get("/")
def read_root():
    return {
        "message": "Bienvenue sur l'API Basket Live !",
        "server_time": datetime.utcnow().isoformat(),
        "docs_url": "/docs"
    }

@app.get("/health")
def health_check():
    return {"status": "healthy", "service": "basket-api"}

@app.get("/matches")
def get_matches():
    return {"count": len(MATCHES), "data": MATCHES}
EOF
```

### Étape 5.3 : Tester l'API en direct

Lance le serveur Uvicorn :

```bash
# 🌐 [SUR LE VPS - Session SSH]
uvicorn main:app --host 0.0.0.0 --port 8000
```

> **Comprendre `--host 0.0.0.0` vs `127.0.0.1` :**
> - `127.0.0.1` (localhost) n'écoute que les requêtes venant de l'intérieur de la machine elle-même.
> - `0.0.0.0` écoute sur **toutes les cartes réseau**, rendant l'API accessible depuis l'extérieur (ton PC).

### Tester depuis ton PC :

1. Ouvre le navigateur sur ton PC : `http://IP_DE_TON_VPS:8000/`
2. Ouvre la documentation automatique Swagger : `http://IP_DE_TON_VPS:8000/docs`
3. Dans ton terminal VPS, appuie sur **`Ctrl + C`** pour arrêter le serveur de test.

---

# 7. ⚙️ Étape 6 : Gérer l'API en tâche de fond avec systemd

Quand tu fermes ta session SSH, Uvicorn s'arrête si tu l'as lancé manuellement. En production, on crée un **service systemd** qui :
- Lance l'API automatiquement au démarrage du serveur.
- Redémarre l'API automatiquement en cas de crash.

### Créer le fichier de service systemd :

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo nano /etc/systemd/system/basket-api.service
```

Colle la configuration suivante (adapte le nom d'utilisateur si tu n'as pas utilisé `devops`) :

```ini
[Unit]
Description=Service FastAPI Basket Live API
After=network.target

[Service]
User=devops
WorkingDirectory=/home/devops/basket-app/api
ExecStart=/home/devops/basket-app/api/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=3
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```
*Sauvegarde avec `Ctrl + O` puis Entrée, et quitte avec `Ctrl + X`.*

### Activer et démarrer le service :

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Recharger systemd pour détecter le nouveau service
sudo systemctl daemon-reload

# 2. Activer le démarrage automatique au boot du serveur
sudo systemctl enable basket-api

# 3. Démarrer le service immédiatement
sudo systemctl start basket-api

# 4. Vérifier son statut (doit être en vert : active (running))
sudo systemctl status basket-api
```

---

# 8. 💥 Exercice "Casser & Réparer"

**Scénario d'incident :** L'API plante ou quelqu'un a modifié le code avec une erreur de syntaxe.

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Introduisons une erreur de syntaxe dans main.py
echo "UNE LIGNE QUI FAIT CRASHER PYTHON" >> ~/basket-app/api/main.py

# 2. Redémarrons le service
sudo systemctl restart basket-api

# 3. Constatons le crash
sudo systemctl status basket-api
# Le statut affiche 'failed' ou 'activating (auto-restart)'

# 4. Comment voir les logs d'erreur précis ?
journalctl -u basket-api -n 20 --no-pager

# 5. Réparons le fichier en supprimant la mauvaise ligne
sed -i '/UNE LIGNE QUI FAIT CRASHER PYTHON/d' ~/basket-app/api/main.py

# 6. Redémarrons et vérifions
sudo systemctl restart basket-api
sudo systemctl status basket-api
```

---

# ✅ Checklist de validation - Jour 2

- [ ] Mon VPS Linux (Ubuntu) est actif et possède une IP publique.
- [ ] Je sais me connecter en SSH avec mon utilisateur non-root `devops`.
- [ ] J'ai créé un environnement virtuel Python et installé FastAPI et Uvicorn.
- [ ] Mon API répond correctement sur `http://IP_DE_TON_VPS:8000/health`.
- [ ] J'ai configuré un service `systemd` pour que l'API tourne en tâche de fond 24h/24.
- [ ] Je sais consulter les logs en temps réel avec `journalctl -u basket-api -f`.

👉 **Demain (Jour 3) :** Fini les installations manuelles et les conflits de versions ! Nous allons conteneuriser toute notre application avec **Docker & Docker Compose** !
