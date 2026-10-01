# 🏀 Jour 5 --- Stack Complète : Next.js + FastAPI + PostgreSQL

## 🎯 Objectif du jour

À la fin de cette journée, tu auras assemblé une **architecture 3-tiers complète** prête pour la production :
1. **Frontend** : Dashboard interactif en **Next.js** (React) conteneurisé avec un Dockerfile multi-stage ultra-léger.
2. **Backend** : API REST **FastAPI** connectée à PostgreSQL via l'ORM **SQLAlchemy**.
3. **Database** : Base de données relationnelle **PostgreSQL 16** persistée sur volume.
4. **Réseau & Reverse Proxy** : Nginx routant `mondomaine.com` vers le Frontend et `api.mondomaine.com` vers le Backend.

---

# 1. 🏗️ Architecture complète de la stack

``` text
                               ┌─────────────────────────┐
                               │  UTILISATEUR / INTERNET │
                               └────────────┬────────────┘
                                            │
                        ┌───────────────────┴───────────────────┐
                        │        NGINX (Reverse Proxy)          │
                        │    Certificats SSL / HTTPS Let's Encrypt│
                        └─────────────┬───────────┬─────────────┘
                                      │           │
       https://ton-domaine.com        │           │  https://api.ton-domaine.com
                                      ▼           ▼
                      ┌──────────────────┐     ┌──────────────────┐
                      │ Frontend Next.js │     │ Backend FastAPI  │
                      │   (Port 3000)    │     │   (Port 8000)    │
                      └──────────────────┘     └────────┬─────────┘
                                                        │ Réseau interne Docker
                                                        ▼
                                               ┌──────────────────┐
                                               │    PostgreSQL    │
                                               │   (Port 5432)    │
                                               │ Volume persistant│
                                               └──────────────────┘
```

---

# 2. 🗄️ Étape 1 : Le Backend FastAPI connecté à PostgreSQL

Connecte-toi à ton VPS :

```powershell
# 💻 [SUR TON PC]
ssh devops@IP_DE_TON_VPS
```

Rends-toi dans le dossier de l'API :

```bash
# 🌐 [SUR LE VPS - Session SSH]
cd ~/basket-app/api
```

### 1.1 Mettre à jour `requirements.txt` avec SQLAlchemy et le driver PostgreSQL

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > requirements.txt
fastapi==0.111.0
uvicorn[standard]==0.30.1
sqlalchemy==2.0.30
psycopg2-binary==2.9.9
pydantic==2.7.1
EOF
```

### 1.2 Créer la configuration de la base de données `database.py`

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > database.py
import os
import time
from sqlalchemy import create_engine
from sqlalchemy.orm import declarative_base, sessionmaker

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://basket_user:super_secret_password_123@db:5432/basket_db")

# Mécanisme de reconnexion pour attendre que PostgreSQL soit prêt
engine = None
for _ in range(10):
    try:
        engine = create_engine(DATABASE_URL)
        with engine.connect():
            break
    except Exception:
        time.sleep(2)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
EOF
```

### 1.3 Créer les modèles et schémas `models.py` & `schemas.py`

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > models.py
from sqlalchemy import Column, Integer, String, DateTime
from datetime import datetime
from database import Base

class MatchModel(Base):
    __tablename__ = "matches"

    id = Column(Integer, primary_key=True, index=True)
    home_team = Column(String(50), nullable=False)
    away_team = Column(String(50), nullable=False)
    home_score = Column(Integer, default=0)
    away_score = Column(Integer, default=0)
    status = Column(String(20), default="Live")
    created_at = Column(DateTime, default=datetime.utcnow)
EOF
```

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > schemas.py
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class MatchCreate(BaseModel):
    home_team: str
    away_team: str
    home_score: int = 0
    away_score: int = 0
    status: str = "Live"

class MatchResponse(MatchCreate):
    id: int
    created_at: Optional[datetime] = None

    class Config:
        from_attributes = True
EOF
```

### 1.4 Mettre à jour `main.py` avec les routes CRUD réelles

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > main.py
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.orm import Session
from typing import List

from database import engine, Base, get_db
import models
import schemas

# Créer automatiquement les tables dans PostgreSQL au démarrage
Base.metadata.create_all(bind=engine)

app = FastAPI(title="Basket Live API", version="2.0.0")

# Autoriser les requêtes cross-origin depuis le frontend Next.js
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
def health():
    return {"status": "healthy", "service": "basket-api-v2"}

@app.get("/matches", response_model=List[schemas.MatchResponse])
def get_matches(db: Session = Depends(get_db)):
    return db.query(models.MatchModel).order_by(models.MatchModel.id.desc()).all()

@app.post("/matches", response_model=schemas.MatchResponse, status_code=status.HTTP_201_CREATED)
def create_match(match: schemas.MatchCreate, db: Session = Depends(get_db)):
    db_match = models.MatchModel(**match.model_dump())
    db.add(db_match)
    db.commit()
    db.refresh(db_match)
    return db_match
EOF
```

---

# 3. 💻 Étape 2 : Le Frontend Next.js et son Dockerfile Multi-Stage

Nous allons créer une application Next.js avec un **Dockerfile Multi-Stage** : cela permet de compiler l'application dans un conteneur temporaire et de ne garder dans l'image finale que les fichiers strictement nécessaires.
👉 **Résultat : Une image de 90 Mo au lieu de 1,2 Go !**

### 2.1 Initialiser le dossier frontend

```bash
# 🌐 [SUR LE VPS - Session SSH]
mkdir -p ~/basket-app/frontend/src/app
cd ~/basket-app/frontend
```

### 2.2 Créer le fichier `package.json`

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > package.json
{
  "name": "basket-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "14.2.3",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  }
}
EOF
```

### 2.3 Créer la configuration `next.config.mjs` (Mode Standalone pour Docker)

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > next.config.mjs
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
};

export default nextConfig;
EOF
```

### 2.4 Créer la page d'accueil interactive `src/app/page.jsx`

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > src/app/page.jsx
"use client";
import { useState, useEffect } from "react";

export default function Home() {
  const [matches, setMatches] = useState([]);
  const [homeTeam, setHomeTeam] = useState("");
  const [awayTeam, setAwayTeam] = useState("");
  const [homeScore, setHomeScore] = useState(0);
  const [awayScore, setAwayScore] = useState(0);
  const [loading, setLoading] = useState(true);

  // URL de l'API configurée via variable d'environnement ou fallback
  const API_URL = process.env.NEXT_PUBLIC_API_URL || "https://api.ton-domaine.com";

  const fetchMatches = async () => {
    try {
      const res = await fetch(`${API_URL}/matches`);
      const data = await res.json();
      setMatches(data);
    } catch (err) {
      console.error("Erreur de récupération :", err);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchMatches();
  }, []);

  const handleAddMatch = async (e) => {
    e.preventDefault();
    if (!homeTeam || !awayTeam) return;

    try {
      await fetch(`${API_URL}/matches`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          home_team: homeTeam,
          away_team: awayTeam,
          home_score: parseInt(homeScore),
          away_score: parseInt(awayScore),
          status: "Live"
        }),
      });
      setHomeTeam("");
      setAwayTeam("");
      setHomeScore(0);
      setAwayScore(0);
      fetchMatches();
    } catch (err) {
      alert("Erreur lors de l'ajout du match");
    }
  };

  return (
    <main style={{ maxWidth: "800px", margin: "40px auto", fontFamily: "system-ui, sans-serif", padding: "20px" }}>
      <header style={{ borderBottom: "2px solid #3b82f6", paddingBottom: "15px", marginBottom: "30px" }}>
        <h1 style={{ margin: 0, color: "#1e3a8a" }}>🏀 Basket Live Dashboard</h1>
        <p style={{ color: "#6b7280" }}>Stack : Next.js 14 + FastAPI + PostgreSQL + Nginx</p>
      </header>

      {/* Formulaire d'ajout */}
      <section style={{ background: "#f3f4f6", padding: "20px", borderRadius: "8px", marginBottom: "30px" }}>
        <h3>Ajouter un Match en Direct</h3>
        <form onSubmit={handleAddMatch} style={{ display: "grid", gridTemplateColumns: "1fr 1fr 80px 80px auto", gap: "10px", alignItems: "center" }}>
          <input placeholder="Équipe Domicile" value={homeTeam} onChange={(e) => setHomeTeam(e.target.value)} required style={{ padding: "8px" }} />
          <input placeholder="Équipe Extérieur" value={awayTeam} onChange={(e) => setAwayTeam(e.target.value)} required style={{ padding: "8px" }} />
          <input type="number" value={homeScore} onChange={(e) => setHomeScore(e.target.value)} style={{ padding: "8px" }} />
          <input type="number" value={awayScore} onChange={(e) => setAwayScore(e.target.value)} style={{ padding: "8px" }} />
          <button type="submit" style={{ background: "#3b82f6", color: "white", border: "none", padding: "10px 15px", borderRadius: "4px", cursor: "pointer" }}>Ajouter</button>
        </form>
      </section>

      {/* Liste des matchs */}
      <section>
        <h3>Matchs Enregistrés ({matches.length})</h3>
        {loading ? <p>Chargement des matchs...</p> : (
          <div style={{ display: "flex", flexDirection: "column", gap: "10px" }}>
            {matches.map((m) => (
              <div key={m.id} style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "15px", border: "1px solid #e5e7eb", borderRadius: "8px", background: "white", boxShadow: "0 1px 3px rgba(0,0,0,0.1)" }}>
                <span style={{ fontWeight: "bold", fontSize: "1.1rem" }}>{m.home_team} vs {m.away_team}</span>
                <span style={{ fontSize: "1.3rem", fontWeight: "bold", color: "#2563eb" }}>{m.home_score} - {m.away_score}</span>
                <span style={{ background: "#dbeafe", color: "#1e40af", padding: "4px 8px", borderRadius: "12px", fontSize: "0.85rem" }}>{m.status}</span>
              </div>
            ))}
          </div>
        )}
      </section>
    </main>
  );
}
EOF
```

```bash
# 🌐 [SUR LE VPS - Session SSH]
# Layout global Next.js
cat << 'EOF' > src/app/layout.jsx
export const metadata = {
  title: "Basket Live Dashboard",
  description: "Roadmap DevOps VPS",
};

export default function RootLayout({ children }) {
  return (
    <html lang="fr">
      <body style={{ margin: 0, backgroundColor: "#f9fafb" }}>{children}</body>
    </html>
  );
}
EOF
```

### 2.5 Créer le `Dockerfile` Multi-Stage de Next.js

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > Dockerfile
# Étape 1 : Installation des dépendances
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json ./
RUN npm install

# Étape 2 : Construction du projet
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
RUN npm run build

# Étape 3 : Image finale de production ultra-légère
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000

CMD ["node", "server.js"]
EOF
```

---

# 4. 🎼 Étape 3 : Unifier avec `docker-compose.yml`

Reviens à la racine du projet :

```bash
# 🌐 [SUR LE VPS - Session SSH]
cd ~/basket-app
```

Crée le fichier `docker-compose.yml` complet :

```bash
# 🌐 [SUR LE VPS - Session SSH]
cat << 'EOF' > docker-compose.yml
services:
  # 1. Frontend Next.js
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        NEXT_PUBLIC_API_URL: https://api.ton-domaine.com
    container_name: basket_frontend
    restart: always
    ports:
      - "3000:3000"
    depends_on:
      - api
    networks:
      - basket_net

  # 2. Backend FastAPI
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    container_name: basket_api
    restart: always
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://basket_user:super_secret_password_123@db:5432/basket_db
    depends_on:
      db:
        condition: service_healthy
    networks:
      - basket_net

  # 3. Base de données PostgreSQL
  db:
    image: postgres:16-alpine
    container_name: basket_postgres
    restart: always
    environment:
      POSTGRES_DB: basket_db
      POSTGRES_USER: basket_user
      POSTGRES_PASSWORD: super_secret_password_123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U basket_user -d basket_db"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - basket_net

volumes:
  postgres_data:

networks:
  basket_net:
    driver: bridge
EOF
```

### Lancer la construction et le démarrage de la stack :

```bash
# 🌐 [SUR LE VPS - Session SSH]
docker compose up -d --build
```
Vérifie avec `docker compose ps` que les 3 conteneurs sont au statut `Up` !

---

# 5. 🌐 Étape 4 : Configurer Nginx pour Frontend + Backend

Nous configurons Nginx pour router :
- `ton-domaine.com` $\rightarrow$ Next.js (`http://127.0.0.1:3000`)
- `api.ton-domaine.com` $\rightarrow$ FastAPI (`http://127.0.0.1:8000`)

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo nano /etc/nginx/sites-available/basket-app
```

Colle la configuration unifiée :

```nginx
# 1. Configuration du FRONTEND
server {
    listen 80;
    server_name ton-domaine.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

# 2. Configuration de l'API BACKEND
server {
    listen 80;
    server_name api.ton-domaine.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Active la configuration et sécurise les deux domaines avec Certbot :

```bash
# 🌐 [SUR LE VPS - Session SSH]
sudo ln -sf /etc/nginx/sites-available/basket-app /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/basket-api
sudo nginx -t
sudo systemctl reload nginx

# Obtenir les certificats SSL pour les deux domaines en une seule commande
sudo certbot --nginx -d ton-domaine.com -d api.ton-domaine.com
```

---

# 6. 🧪 Étape 5 : Test de bout en bout

1. Ouvre `https://ton-domaine.com` dans ton navigateur.
2. Remplis le formulaire pour ajouter un match (ex: *Lakers vs Warriors : 110 - 105*).
3. Actualise la page : le match persiste car il a été enregistré dans PostgreSQL ! 🎉

---

# ✅ Checklist de validation - Jour 5

- [ ] L'API FastAPI communique avec PostgreSQL via SQLAlchemy.
- [ ] Le frontend Next.js est conteneurisé avec une image optimisée (multi-stage build).
- [ ] Docker Compose orchestre les 3 conteneurs (`frontend`, `api`, `db`) sur un réseau isolé.
- [ ] Nginx gère le routage et le SSL pour le domaine principal et le sous-domaine API.
- [ ] Les données créées depuis l'interface web sont conservées après actualisation.

👉 **Demain (Jour 6) :** On automatise tout ! Plus besoin de se connecter en SSH pour mettre à jour l'application : nous mettons en place le pipeline **CI/CD avec GitHub Actions** !
