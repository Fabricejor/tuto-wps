# 🔒 Jour 4 --- Nginx, Nom de Domaine et HTTPS

## 🎯 Objectif du jour

À la fin de cette journée, tu sauras :
1. Comprendre le rôle d'un **Reverse Proxy** et comment fonctionne le système de **DNS**.
2. Faire pointer un nom de domaine ou sous-domaine vers l'adresse IP de ton VPS.
3. Installer et configurer **Nginx** pour rediriger le trafic web vers ton conteneur Docker.
4. Générer un certificat **SSL/TLS gratuit** avec **Let's Encrypt & Certbot**.
5. Obtenir une URL sécurisée de type `https://api.ton-domaine.com/` avec le cadenas vert 🔒.

---

# 1. 🧠 La théorie : Pourquoi Nginx et pas un accès direct ?

### L'analogie du réceptionniste d'hôtel
Imagine un grand hôtel :
- Les clients (navigateurs) arrivent à la réception principale (Port 80 / 443).
- Le réceptionniste (**Nginx**) vérifie leur badge de sécurité (**HTTPS/SSL**).
- Selon ce que demande le client, il l'oriente vers la bonne pièce :
  - `https://mondomaine.com` $\rightarrow$ Le restaurant (Frontend Next.js sur port 3000).
  - `https://api.mondomaine.com` $\rightarrow$ La cuisine (Backend FastAPI sur port 8000).

``` text
Navigateur Web (Client)
         │
         │ Requête HTTPS sécurisée (Port 443)
         ▼
 ┌──────────────────────────────────────────────────────────┐
 │  Serveur Nginx (Reverse Proxy + Terminaison SSL)         │
 └────────────────────────────┬─────────────────────────────┘
                              │
               Routage interne sécurisé sur localhost
                              │
                              ▼
 ┌──────────────────────────────────────────────────────────┐
 │  Conteneur Docker FastAPI (Écoute sur http://127.0.0.1:8000)│
 └──────────────────────────────────────────────────────────┘
```

### Pourquoi c'est indispensable en production ?
1. **Sécurité** : Tes conteneurs Docker ne sont plus exposés directement sur Internet.
2. **Centralisation SSL** : Nginx gère le déchiffrement HTTPS. Tes applications internes n'ont pas à gérer les certificats.
3. **Performance** : Nginx peut compresser les réponses (`gzip`), mettre en cache les fichiers statiques et gérer des milliers de connexions simultanées.

---

# 2. 🌐 Étape 1 : Configurer le nom de domaine (DNS)

Pour que `api.ton-domaine.com` pointe vers ton serveur, il faut ajouter un enregistrement DNS de type **`A`**.

### Configuration chez ton Registrar (Cloudflare, OVH, Namecheap, etc.) :

Rends-toi sur l'interface de gestion de ton nom de domaine dans la section **DNS Records** et ajoute :

| Type | Nom / Host / Sous-domaine | Valeur / Cible | TTL |
| :--- | :--- | :--- | :--- |
| **A** | `api` (ou `api.ton-domaine.com`) | `IP_DE_TON_VPS` (ex: `198.51.100.42`) | Automatique / 300s |
| **A** | `@` (ou `ton-domaine.com`) | `IP_DE_TON_VPS` (pour le futur frontend) | Automatique / 300s |

> [!TIP]
> **Option 100% Gratuite pour s'entraîner (DuckDNS) :**
> Si tu n'as pas de nom de domaine payant :
> 1. Va sur [duckdns.org](https://www.duckdns.org/) et connecte-toi avec GitHub.
> 2. Crée un sous-domaine (ex: `mon-basket-app`).
> 3. Renseigne l'adresse IP de ton VPS dans le champ IP.
> 4. Tu disposes immédiatement d'un domaine fonctionnel : `mon-basket-app.duckdns.org` !

### Vérifier la propagation DNS depuis ton PC :

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
nslookup api.ton-domaine.com
```
*Le résultat doit afficher l'adresse IP exacte de ton VPS.*

---

# 3. 📦 Étape 2 : Installer Nginx sur le VPS

Connecte-toi à ton VPS :

```powershell
# 💻 [SUR TON PC]
ssh devops@IP_DE_TON_VPS
```

Installe Nginx avec le gestionnaire de paquets APT :

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Installer Nginx
sudo apt update
sudo apt install -y nginx

# 2. Démarrer et activer Nginx au démarrage du système
sudo systemctl enable --now nginx

# 3. Vérifier que Nginx tourne
sudo systemctl status nginx
```

Si tu ouvres `http://IP_DE_TON_VPS` dans ton navigateur, tu vois la page d'accueil par défaut : *"Welcome to nginx!"*.

---

# 4. ⚙️ Étape 3 : Configurer le Reverse Proxy pour FastAPI

Nous allons créer un fichier de configuration dédié à notre API.

### 4.1 Créer le fichier de configuration

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo nano /etc/nginx/sites-available/basket-api
```

Colle la configuration suivante en remplaçant `api.ton-domaine.com` par ton domaine réel :

```nginx
server {
    listen 80;
    server_name api.ton-domaine.com;

    # En-têtes pour transmettre les informations réelles du client à FastAPI
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts pour éviter que Nginx ne coupe les requêtes longues
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```
*Sauvegarde avec `Ctrl + O` puis Entrée, et quitte avec `Ctrl + X`.*

### 4.2 Activer la configuration et supprimer le site par défaut

Sous Debian/Ubuntu, Nginx utilise deux dossiers :
- `sites-available/` : Contient toutes les configurations disponibles.
- `sites-enabled/` : Contient les liens symboliques des configurations actives.

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Créer le lien symbolique pour activer notre site
sudo ln -s /etc/nginx/sites-available/basket-api /etc/nginx/sites-enabled/

# 2. Supprimer la configuration par défaut de Nginx
sudo rm -f /etc/nginx/sites-enabled/default

# 3. Tester la syntaxe de la configuration (IMPORTANT !)
sudo nginx -t
```
> **Résultat attendu :**
> `nginx: the configuration file /etc/nginx/nginx.conf syntax is ok`
> `nginx: configuration file /etc/nginx/nginx.conf test is successful`

### 4.3 Recharger Nginx

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo systemctl reload nginx
```

Si tu visites `http://api.ton-domaine.com` (en HTTP), tu accèdes désormais à ton API FastAPI !

---

# 5. 🔐 Étape 4 : Activer le chiffrement HTTPS avec Certbot (Let's Encrypt)

Let's Encrypt fournit des certificats SSL/TLS valides et reconnus par tous les navigateurs gratuitement.

### 5.1 Installer Certbot et son plugin Nginx

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo apt install -y certbot python3-certbot-nginx
```

### 5.2 Générer le certificat SSL automatiquement

Lance la commande Certbot en remplaçant par ton domaine :

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo certbot --nginx -d api.ton-domaine.com
```

1. Renseigne une adresse email de contact (pour les alertes d'expiration).
2. Accepte les conditions d'utilisation (`Y`).
3. Certbot valide automatiquement le domaine, génère les clés de chiffrement et modifie la configuration Nginx pour rediriger automatiquement tout le trafic HTTP (port 80) vers HTTPS (port 443) !

### 5.3 Tester le renouvellement automatique des certificats

Les certificats Let's Encrypt sont valables 90 jours. Certbot installe une tâche cron / timer systemd pour les renouveler automatiquement. Testons une simulation de renouvellement :

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo certbot renew --dry-run
```
> `Congratulations, all simulated renewals succeeded!`

---

# 6. 🧪 Étape 5 : Tester la sécurisation de bout en bout

Depuis ton PC, ouvre ton navigateur ou ton terminal :

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
curl -I https://api.ton-domaine.com/health
```

> **Résultat attendu :**
> `HTTP/2 200`
> `content-type: application/json`
> `{"status":"healthy","service":"basket-api"}`

Ouvre également l'URL dans ton navigateur : le cadenas de sécurité s'affiche fièrement à gauche de l'adresse ! 🔒

---

# 7. 💥 Exercice "Casser & Réparer"

**Scénario d'incident 1 : L'erreur de syntaxe Nginx**
Que se passe-t-il si tu fais une erreur dans la configuration Nginx ?

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Ajoutons volontairement une erreur de syntaxe
sudo sed -i 's/proxy_pass/mauvaise_directive/' /etc/nginx/sites-available/basket-api

# 2. Testons la configuration avec nginx -t AVANT de recharger le service
sudo nginx -t
# Nginx t'indique précisément le numéro de la ligne en erreur !

# 3. Réparons le fichier
sudo sed -i 's/mauvaise_directive/proxy_pass/' /etc/nginx/sites-available/basket-api
sudo nginx -t
sudo systemctl reload nginx
```

**Scénario d'incident 2 : L'erreur "502 Bad Gateway"**
Que signifie une erreur 502 ? Cela signifie que Nginx fonctionne parfaitement, mais que l'application derrière (FastAPI) ne répond pas !

```bash
# 🌐 [SUR LE VPS - Session SSH]

# 1. Arrêtons l'API Docker
cd ~/basket-app && docker compose stop api

# 2. Testons l'URL depuis le navigateur ou curl
curl -I https://api.ton-domaine.com/health
# Résultat : HTTP/1.1 502 Bad Gateway

# 3. Redémarrons l'API
docker compose start api
curl -I https://api.ton-domaine.com/health
# Résultat : HTTP/2 200 OK !
```

---

# ✅ Checklist de validation - Jour 4

- [ ] Mon sous-domaine `api.ton-domaine.com` pointe vers l'adresse IP de mon VPS.
- [ ] Nginx est installé et configuré en Reverse Proxy vers le port 8000.
- [ ] J'ai supprimé la configuration Nginx par défaut et validé avec `sudo nginx -t`.
- [ ] Le certificat SSL Let's Encrypt est installé avec succès via Certbot.
- [ ] Toute requête `http://` est automatiquement redirigée vers `https://`.
- [ ] Je sais ce que signifie une erreur `502 Bad Gateway` et comment la diagnostiquer.

👉 **Demain (Jour 5) :** Nous construisons la stack complète en ajoutant le **Frontend Next.js** et en reliant le tout à **PostgreSQL** !
