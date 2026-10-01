# 🐧 Jour 1 --- Linux, SSH et Git

## 🎯 Objectif du jour

À la fin de cette journée, tu sauras :
1. Configurer ton environnement de dev local (Windows/Mac/Linux) avec Git et SSH.
2. Générer une paire de clés SSH et la lier à ton compte GitHub.
3. Maîtriser les commandes essentielles du terminal Linux sans appréhension.
4. Créer, versionner et pousser un projet sur GitHub via la ligne de commande.
5. Comprendre la différence physique entre ta machine locale et un serveur distant.

---

# 1. 🧠 La théorie vulgarisée : L'analogie du restaurant

Pour comprendre l'écosystème DevOps, imagine la gestion d'un restaurant gastronomique :

| Élément | Rôle dans l'analogie | Rôle en informatique |
| :--- | :--- | :--- |
| **Ton PC** | **Le labo secret du chef** | Ton ordinateur personnel où tu écris le code et fais tes tests. |
| **Git** | **Le carnet de recettes** | L'historique qui enregistre chaque modification de code étape par étape. |
| **GitHub** | **La bibliothèque centrale** | Le serveur cloud sécurisé qui stocke et partage ton carnet de recettes. |
| **Le VPS** | **La cuisine du restaurant** | Un serveur Linux loué, allumé 24h/24, accessible au public. |
| **SSH** | **Le badge d'accès sécurisé** | Le tunnel chiffré qui te permet d'entrer dans la cuisine à distance. |

---

# 2. 🛠️ Configuration initiale sur ton PC

### Étape 2.1 : Vérifier / Installer Git

Sur ton PC Windows, ouvre **PowerShell** ou **Git Bash** :

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
git --version
```
*Si la commande n'est pas reconnue, télécharge et installe [Git for Windows](https://git-scm.com/download/win).*

### Étape 2.2 : Configurer ton identité Git

Ces informations seront attachées à chacun de tes commits :

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
git config --global user.name "Ton Prenom Nom"
git config --global user.email "ton.email@example.com"
```

Vérifie que la configuration a bien été enregistrée :
```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
git config --list
```

---

# 3. 🔑 Les clés SSH : Le badge d'accès universel

### Comment ça marche ? (Asymmetric Cryptography)
Quand tu crées une clé SSH, deux fichiers sont générés :
1. **La clé privée (`id_ed25519`)** : Ton mot de passe secret absolu. **Elle ne doit JAMAIS quitter ton PC.**
2. **La clé publique (`id_ed25519.pub`)** : La serrure correspondante. Tu peux la donner à GitHub ou à ton VPS sans danger.

### Étape 3.1 : Générer ta paire de clés SSH

Ouvre ton terminal sur ton PC :

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
ssh-keygen -t ed25519 -C "ton.email@example.com"
```

1. Le terminal te demande : `Enter file in which to save the key` $\rightarrow$ **Appuie simplement sur Entrée** (pour garder le chemin par défaut `~/.ssh/id_ed25519`).
2. Il te demande une passphrase (optionnelle mais recommandée) $\rightarrow$ Tape un mot de passe ou appuie sur **Entrée** deux fois.

### Étape 3.2 : Copier ta clé publique

Affiche le contenu de la clé **publique** (celle qui termine par `.pub`) :

```powershell
# 💻 [SUR TON PC - PowerShell sous Windows]
Get-Content ~\.ssh\id_ed25519.pub

# OU si tu es sous Git Bash / macOS / Linux :
cat ~/.ssh/id_ed25519.pub
```
*Le résultat ressemble à : `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ton.email@example.com`.*
Copie tout ce texte dans ton presse-papiers.

### Étape 3.3 : Ajouter la clé sur ton compte GitHub

1. Va sur [github.com/settings/keys](https://github.com/settings/keys).
2. Clique sur le bouton vert **"New SSH key"**.
3. Dans **Title**, écris par exemple : `Mon PC Portable Windows`.
4. Dans **Key**, colle le contenu complet de ta clé publique copiée à l'étape 3.2.
5. Clique sur **"Add SSH key"**.

### Étape 3.4 : Tester la connexion avec GitHub

Dans ton terminal local, teste si GitHub te reconnaît :

```powershell
# 💻 [SUR TON PC - PowerShell ou Git Bash]
ssh -T git@github.com
```

> **Résultat attendu :**
> `Hi ton-pseudo! You've successfully authenticated, but GitHub does not provide shell access.`
> 🎉 Félicitations, ton PC est maintenant authentifié de manière sécurisée auprès de GitHub sans avoir besoin de retaper ton mot de passe !

---

# 4. 🐧 Le Terminal Linux en pratique (Tutoriel complet)

Pour t'entraîner aux commandes Linux, ouvre **Git Bash** (sur Windows) ou un terminal Linux/macOS.

### Les commandes indispensables décryptées :

| Commande | Signification | Ce qu'elle fait |
| :--- | :--- | :--- |
| `pwd` | *Print Working Directory* | Affiche le dossier dans lequel tu te trouves actuellement. |
| `ls -la` | *List (All + Long)* | Liste **tous** les fichiers et dossiers (y compris les cachés `.`) avec les droits et tailles. |
| `cd <nom>` | *Change Directory* | Entre dans un dossier (`cd ..` pour remonter d'un niveau). |
| `mkdir <nom>` | *Make Directory* | Crée un nouveau dossier. |
| `touch <fichier>`| *Touch* | Crée un fichier vide. |
| `cat <fichier>` | *Concatenate* | Affiche le contenu d'un fichier dans le terminal. |
| `nano <fichier>`| *Nano Editor* | Ouvre un éditeur de texte direct dans le terminal (Ctrl+O pour enregistrer, Ctrl+X pour quitter). |
| `cp <source> <cible>` | *Copy* | Copie un fichier ou dossier (`cp -r` pour un dossier). |
| `mv <source> <cible>` | *Move / Rename* | Déplace ou renomme un fichier/dossier. |
| `rm <fichier>` | *Remove* | Supprime un fichier (`rm -rf <dossier>` pour supprimer un dossier et son contenu). |
| `clear` | *Clear* | Nettoie l'écran du terminal. |

---

### 🧪 TP Pratique 1 : Manipulation du système de fichiers

Tape ces commandes une par une dans ton terminal :

```bash
# 💻 [SUR TON PC - Dans Git Bash ou WSL]

# 1. Vérifier où nous sommes
pwd

# 2. Créer un dossier d'entraînement et y entrer
mkdir lab-linux
cd lab-linux

# 3. Créer une arborescence complète en une seule commande
mkdir -p projet/src projet/logs

# 4. Créer des fichiers
touch projet/src/index.js projet/README.md

# 5. Écrire du texte dans un fichier grâce aux redirections :
# Le symbole '>' ÉCRASE ou crée le fichier
echo "console.log('Hello DevOps');" > projet/src/index.js
echo "# Mon Projet Lab" > projet/README.md

# Le symbole '>>' AJOUTE à la fin du fichier sans effacer l'existant
echo "Documentation en cours..." >> projet/README.md

# 6. Vérifier le contenu des fichiers
cat projet/README.md
cat projet/src/index.js

# 7. Voir toute l'arborescence
ls -la projet/
```

---

# 5. 📊 Surveiller les ressources et processus

Quand une machine ou un serveur tourne, il est crucial de savoir ce qui consomme de la mémoire ou du processeur.

| Commande | Ce qu'elle affiche | Pourquoi l'utiliser ? |
| :--- | :--- | :--- |
| `whoami` | Nom de l'utilisateur actuel | Savoir si on est en `root` (admin) ou utilisateur standard. |
| `hostname` | Nom de la machine | Savoir sur quel serveur on se trouve. |
| `df -h` | *Disk Free (Human readable)* | Vérifier l'espace disque restant (ex: 80% utilisé). |
| `free -h` | *Memory Free* | Vérifier la RAM disponible et utilisée. |
| `top` ou `htop` | Gestionnaire de tâches interactif | Voir les processus en direct qui consomment du CPU/RAM (`q` pour quitter). |
| `ps aux` | *Process Status* | Liste instantanée de tous les processus en cours d'exécution. |

---

# 6. 🐙 Git & GitHub : Le cycle de vie complet

### Le flux de travail universel :

``` text
+------------------+       git add       +-----------------+      git commit     +-------------------+       git push       +------------------+
| Espace de travail| ------------------> | Zone de transit | ------------------> | Dépôt Local (.git)| -------------------> |  Dépôt Distant   |
| (Fichiers réels) |                     |    (Staging)    |                     |    (Historique)   |                      | (ex: sur GitHub) |
+------------------+                     +-----------------+                     +-------------------+                      +------------------+
```

---

### 🧪 TP Pratique 2 : Créer et pousser ton premier dépôt

#### Étape 1 : Créer le dépôt localement

```bash
# 💻 [SUR TON PC - PowerShell ou Git Bash]

# Aller dans ton dossier de projets personnels
cd ~
mkdir basket-devops
cd basket-devops

# Initialiser Git
git init

# Créer un fichier README
echo "# Basket DevOps - Mini Application" > README.md

# Voir l'état du dépôt
git status
```
*Le terminal affiche `README.md` en rouge (fichier non suivi / untracked).*

#### Étape 2 : Indexer et commiter

```bash
# 💻 [SUR TON PC - PowerShell ou Git Bash]

# Ajouter le fichier au panier (staging)
git add README.md

# Vérifier l'état (le fichier doit être vert)
git status

# Enregistrer l'état dans l'historique
git commit -m "feat: initial commit with README"
```

#### Étape 3 : Créer le dépôt sur GitHub et le relier

1. Ouvre ton navigateur sur [github.com/new](https://github.com/new).
2. Nom du dépôt : `basket-devops`.
3. Laisse le dépôt en **Public** (ou Private selon ton choix).
4. Ne coche **AUCUNE** case (ni README, ni .gitignore, ni licence, car nous l'avons déjà créé en local).
5. Clique sur **"Create repository"**.
6. Choisis l'option **SSH** (et non HTTPS) sur la page qui s'affiche, puis copie les commandes :

```bash
# 💻 [SUR TON PC - PowerShell ou Git Bash]

# Renommer la branche principale en 'main'
git branch -M main

# Relier ton dossier local au dépôt GitHub (remplace par ton URL SSH)
git remote add origin git@github.com:TON_PSEUDO/basket-devops.git

# Pousser ton code vers GitHub
git push -u origin main
```

Actualise ta page GitHub dans ton navigateur : **ton README.md est en ligne ! 🚀**

---

# 7. 💥 Exercice "Casser & Réparer"

**Scénario de panne :** Tu fais une mauvaise modification par erreur et tu souhaites revenir en arrière avant d'avoir commité.

```bash
# 💻 [SUR TON PC - PowerShell ou Git Bash]

# 1. Tu écris une bêtise dans le fichier
echo "UNE ERREUR CRITIQUE QUI CASSE TOUT" >> README.md

# 2. Tu vérifies ce qui a changé
git diff

# 3. Comment annuler cette modification proprement ?
git restore README.md

# 4. Vérifie le fichier : l'erreur a disparu !
cat README.md
```

---

# ✅ Checklist de validation - Jour 1

Coche ces cases pour valider ton premier module :

- [ ] J'ai vérifié les installations de Git, VS Code et SSH sur mon PC.
- [ ] J'ai généré une paire de clés SSH (`id_ed25519` et `id_ed25519.pub`).
- [ ] J'ai ajouté ma clé publique sur mon compte GitHub et validé la connexion avec `ssh -T git@github.com`.
- [ ] Je sais utiliser `pwd`, `ls -la`, `cd`, `mkdir`, `cat`, `rm` dans le terminal.
- [ ] Je comprends la différence entre `>` (écraser) et `>>` (ajouter à la fin).
- [ ] J'ai créé un dépôt Git local et je l'ai poussé sur GitHub avec `git add`, `git commit`, `git push`.

👉 **Demain (Jour 2) :** Nous prenons possession de notre propre serveur Linux VPS et nous déployons notre première API FastAPI !
