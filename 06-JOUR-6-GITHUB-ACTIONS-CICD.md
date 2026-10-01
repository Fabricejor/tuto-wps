# 🔄 Jour 6 --- CI/CD Automatique avec GitHub Actions

## 🎯 Objectif du jour

À la fin de cette journée, tu auras automatisé **100% de la chaîne de déploiement** :
1. Comprendre l'anatomie d'un pipeline **CI/CD** (Intégration Continue & Déploiement Continu).
2. Configurer des **GitHub Secrets** pour autoriser GitHub à piloter ton VPS en toute sécurité.
3. Écrire un fichier de workflow `.github/workflows/deploy.yml`.
4. Automatiser les tests unitaires et le déploiement sans coupure (*zero-downtime*) à chaque `git push origin main`.
5. Bloquer automatiquement les déploiements si les tests échouent.

---

# 1. 🧠 La théorie : Le saut qualitatif du CI/CD

### Le calvaire du déploiement manuel :
``` text
[💻 Ton PC]                      [🌐 Ton VPS en SSH]
  git push   ───▶ (Attente) ───▶ Connexion manuelle SSH
                                 cd ~/basket-app
                                 git pull
                                 docker compose build
                                 docker compose up -d
```
*Inconvénients : Lent, source d'erreurs humaines, risque de déployer du code cassé directement en production.*

### La sérénité du pipeline CI/CD automatisé :
``` text
[💻 Ton PC]                                [🐙 GitHub Actions Runner]                                [🌐 Ton VPS]
  git push main ──▶ Déclencheur (Trigger) ──▶ ÉTAPE 1 : CI (Linting + Tests unitaires)
                                              └── SI SUCCÈS ──▶ ÉTAPE 2 : CD (Connexion SSH) ──▶ Déploiement automatique
                                                                                                 docker compose up -d --build
                                                                                                 Healthcheck de confirmation
```

---

# 2. 🔑 Étape 1 : Créer une clé SSH dédiée pour GitHub Actions

Pour permettre à GitHub de se connecter à ton VPS sans lui donner ta clé personnelle, nous générons une clé de déploiement dédiée.

### 2.1 Générer la clé sur ton PC ou sur le VPS

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_vps
```
Cela crée deux fichiers :
- `github_actions_vps` (Clé PRIVÉE $\rightarrow$ ira dans GitHub Secrets)
- `github_actions_vps.pub` (Clé PUBLIQUE $\rightarrow$ ira sur le VPS)

### 2.2 Ajouter la clé publique sur ton VPS

Affiche ta clé publique :
```powershell
# 💻 [SUR TON PC]
cat ~/.ssh/github_actions_vps.pub
```

Connecte-toi à ton VPS et ajoute cette clé au fichier `authorized_keys` de l'utilisateur `devops` :

```bash
# 🌐 [SUR LE VPS - Session SSH]
echo "COLLE_ICI_LE_CONTENU_DE_GITHUB_ACTIONS_VPS_PUB" >> ~/.ssh/authorized_keys
```

---

# 3. 🐙 Étape 2 : Configurer les Secrets sur GitHub

1. Rends-toi sur ton dépôt GitHub dans ton navigateur.
2. Clique sur **Settings** $\rightarrow$ **Secrets and variables** $\rightarrow$ **Actions**.
3. Clique sur le bouton vert **"New repository secret"** et ajoute ces 4 secrets :

| Nom du Secret | Valeur à renseigner |
| :--- | :--- |
| `VPS_HOST` | L'adresse IP publique de ton VPS (ex: `198.51.100.42`) |
| `VPS_USERNAME` | `devops` |
| `VPS_SSH_KEY` | Le contenu **complet** de ta clé privée `github_actions_vps` (commence par `-----BEGIN OPENSSH PRIVATE KEY-----`) |
| `VPS_PORT` | `22` |

> [!CAUTION]
> Ne commite **JAMAIS** ta clé privée dans ton code source Git ! Les GitHub Secrets sont chiffrés et masqués dans les logs de build.

---

# 4. 📁 Étape 3 : Écrire des tests unitaires pour l'API

Sur ton PC, dans le dossier local de ton projet :

```bash
# 💻 [SUR TON PC - Dans le dossier du projet]
mkdir -p api/tests
```

Crée le fichier de test `api/tests/test_api.py` :

```python
# 📁 [DANS TON ÉDITEUR - api/tests/test_api.py]
import pytest
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_health_check():
    """Vérifie que l'endpoint de santé répond correctement."""
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "healthy", "service": "basket-api-v2"}

def test_get_matches_endpoint():
    """Vérifie que la route matches renvoie une liste JSON."""
    response = client.get("/matches")
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

---

# 5. ⚙️ Étape 4 : Créer le Workflow GitHub Actions

Sur ton PC, crée l'arborescence `.github/workflows/` :

```bash
# 💻 [SUR TON PC]
mkdir -p .github/workflows
```

Crée le fichier `.github/workflows/deploy.yml` :

```yaml
# 📁 [DANS TON ÉDITEUR - .github/workflows/deploy.yml]
name: CI/CD Pipeline - Test & Deploy

# Déclencher à chaque push sur la branche main ou lors d'une Pull Request
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  # ==========================================
  # JOB 1 : Continuous Integration (Tests)
  # ==========================================
  test:
    name: 🧪 Run Automated Tests
    runs-on: ubuntu-latest

    steps:
      - name: 📥 Checkout du code
        uses: actions/checkout@v4

      - name: 🐍 Configurer Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: 📦 Installer les dépendances
        run: |
          python -m pip install --upgrade pip
          pip install -r api/requirements.txt pytest httpx

      - name: 🚀 Exécuter les tests unitaires
        run: |
          pytest api/tests/

  # ==========================================
  # JOB 2 : Continuous Deployment (Sur le VPS)
  # ==========================================
  deploy:
    name: 🚀 Deploy to VPS
    needs: test # Ce job ne s'exécute QUE si les tests ci-dessus réussissent !
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest

    steps:
      - name: 🔑 Connexion SSH et Déploiement sur le VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          port: ${{ secrets.VPS_PORT }}
          script: |
            echo "👉 1. Navigation dans le dossier du projet..."
            cd ~/basket-app

            echo "👉 2. Récupération des dernières modifications depuis GitHub..."
            git pull origin main

            echo "👉 3. Reconstruction et redémarrage des conteneurs Docker..."
            docker compose up -d --build

            echo "👉 4. Nettoyage des anciennes images Docker inutilisées..."
            docker image prune -f

            echo "✅ Déploiement terminé avec succès sur le VPS !"

      - name: 🩺 Health Check Post-Déploiement
        run: |
          echo "Vérification de la disponibilité de l'API..."
          curl --fail --retry 5 --retry-delay 5 https://api.ton-domaine.com/health || exit 1
          echo "🎉 L'application est en ligne et saine !"
```

---

# 6. 🚀 Étape 5 : Pousser et observer le premier déploiement automatique

Depuis ton PC :

```bash
# 💻 [SUR TON PC - PowerShell ou Git Bash]
git add .
git commit -m "ci: configure automated CI/CD pipeline with GitHub Actions"
git push origin main
```

### Observer la magie en direct :
1. Ouvre ton dépôt sur **GitHub**.
2. Clique sur l'onglet **"Actions"**.
3. Clique sur le workflow en cours : tu vois les étapes s'allumer en vert une par une (Checkout $\rightarrow$ Tests $\rightarrow$ SSH Deploy $\rightarrow$ Health Check) ! 🟢

---

# 7. 💥 Exercice "Casser & Réparer" : La protection anti-régression

**Scénario d'incident :** Un développeur introduit un bug dans le code. Le pipeline doit bloquer le déploiement pour protéger la production.

```python
# 📁 [SUR TON PC - Dans api/main.py]
# Modifie volontairement la réponse de /health pour casser le test :
@app.get("/health")
def health():
    return {"status": "broken"} # Le test attend 'healthy'
```

Pousse la modification :

```bash
# 💻 [SUR TON PC]
git add api/main.py
git commit -m "test: simulate broken code"
git push origin main
```

### Constat dans GitHub Actions :
- ❌ L'étape **"Run Automated Tests"** échoue en rouge.
- 🛡️ Le job **"Deploy to VPS"** est **automatiquement annulé** !
- **Ton serveur en production reste intact et continue de fonctionner sans interruption.**

### Réparons :
Remets `"status": "healthy"` dans `api/main.py`, commite et repousse : le pipeline redevient vert et déploie la correction !

---

# ✅ Checklist de validation - Jour 6

- [ ] J'ai généré une clé SSH dédiée et renseigné les 4 secrets dans GitHub (`VPS_HOST`, `VPS_USERNAME`, `VPS_SSH_KEY`, `VPS_PORT`).
- [ ] J'ai écrit des tests unitaires automatisés avec `pytest`.
- [ ] J'ai créé le fichier `.github/workflows/deploy.yml`.
- [ ] Un simple `git push origin main` déclenche les tests puis déploie automatiquement sur le VPS.
- [ ] J'ai vérifié qu'un test qui échoue bloque immédiatement le déploiement en production.

👉 **Demain (Jour 7) :** Le grand final ! Nous allons durcir la **sécurité du serveur** (UFW, Fail2ban), automatiser les **sauvegardes de base de données**, et maîtriser le guide ultime de **dépannage d'incidents** !
