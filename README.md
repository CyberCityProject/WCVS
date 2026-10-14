# WCVS — Web Common Vulnerability Scanner

WCVS est une plateforme de **scan de posture web passif** : elle observe ce qu’un site expose publiquement (HTTPS, en-têtes, CMS, DNS mail, fuites de chemins, etc.), produit un **score**, un **journal**, un **résumé** et des **pistes de correction** — **sans exploitation**, sans injection, sans brute-force.

Pensée pour les **agences** et les **indépendants** qui gèrent plusieurs sites clients : interface multilingue, langue de rapport par site, abonnements Basic / Pro / Expert, et Assistance IA (plan Expert).

---

## Ce que fait WCVS

| Domaine | Exemples de contrôles |
|--------|------------------------|
| Disponibilité | HTTP, erreurs 5xx, host injoignable |
| TLS | HTTPS, expiration cert, protocole, ALPN/HTTP2 |
| En-têtes | HSTS, CSP, CORS, cookies Secure/HttpOnly… |
| Contenu | Mixed content, SRI, e-mails exposés, libs JS à risque |
| CMS | Empreintes WordPress / Prestashop / Joomla / Drupal |
| WordPress | Plugins/thèmes, xmlrpc, REST users, versions EOL |
| DNS / mail | SPF, DMARC, DKIM, CAA, DNSSEC, BIMI, MTA-STS… |
| Surface | robots.txt, security.txt, chemins sensibles, redirects |

Chaque scan produit :

1. **Score de posture** (0–100) — indice WCVS (ce n’est **pas** un score CVE officiel)
2. **Constats** (findings) avec gravité, confiance, preuve, piste de fix
3. **Fix first** — priorités top 5
4. **Diff** vs scan précédent (nouveaux / résolus / ignorés)
5. **Résumé** rédigé via **Ollama** (LLM local)
6. **Export** Markdown + HTML imprimable (PDF navigateur)

---

## Stack technique

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Portal     │────▶│  API FastAPI │────▶│ PostgreSQL  │
│  React/Vite │     │  + Alembic   │     └─────────────┘
│  :80        │     │  :8000       │
└─────────────┘     └──────┬───────┘
                           │ Redis (file de scans)
                           ▼
                    ┌──────────────┐     ┌─────────────┐
                    │   Worker     │────▶│   Ollama    │
                    │  (scanner)   │     │  (host)     │
                    └──────────────┘     └─────────────┘
```

| Composant | Techno |
|-----------|--------|
| API / Worker | Python 3, FastAPI, SQLAlchemy, Alembic, Redis |
| Portail | React 18, TypeScript, Vite, Nginx |
| Base | PostgreSQL |
| File | Redis 7 |
| LLM | Ollama (ex. `qwen2.5:1.5b-instruct` résumé, `3b` assist) |
| Déploiement | Docker Compose |

---

## Abonnements

| Plan | Prix indicatif | Sites | Scans | Assistance IA |
|------|----------------|-------|-------|---------------|
| **Basic** | 20 € / mois | jusqu’à **5** | illimités | non |
| **Pro** | 50 € / mois | **illimités** | illimités | non |
| **Expert** | 100 € / mois | **illimités** | illimités | **oui** |

> Les quotas portent sur le **nombre de sites**, pas sur le nombre de scans.  
> Le paiement Stripe n’est pas encore branché : bascule de plan via **console admin**.

---

## Multilingue (FR / EN / ES / DE)

Deux niveaux distincts (cas **agence multi-clients**) :

| Niveau | Champ | Rôle |
|--------|--------|------|
| **Interface** | `users.preferred_locale` | Langue du portail (menus, labels, titres/journal à l’affichage) |
| **Rapport site** | `sites.locale` | Langue client : résumé Ollama, export MD/PDF, Assistance IA |

Les titres / hints des règles sont stockés en français en base et **traduits à la lecture**.  
Le paragraphe Ollama est **figé au moment du scan** : pour le régénérer dans une autre langue, changez la langue du site puis relancez un scan.

---

## Prérequis

- Ubuntu Server (ou Linux récent) recommandé
- Docker + Docker Compose
- PostgreSQL accessible depuis les conteneurs (`host.docker.internal` ou IP)
- [Ollama](https://ollama.com/) sur l’hôte (optionnel mais recommandé pour résumé / assist)

Modèles typiques :

```bash
ollama pull qwen2.5:1.5b-instruct
ollama pull qwen2.5:3b-instruct    # Assistance IA (Expert)
```

---

## Installation rapide

### 1. Cloner

```bash
git clone https://github.com/VOTRE_COMPTE/WCVS.git
cd WCVS
```

### 2. (Optionnel) Scripts d’installation hôte

Sur une machine Ubuntu dédiée :

```bash
export WCVS_DB_PASSWORD='mot_de_passe_fort'
chmod +x scripts/*.sh
./scripts/01-install-base.sh      # Postgres, Docker, Ollama
./scripts/02-deploy-app.sh --yes  # build + up de la stack
```

### 3. Configuration

```bash
cd app
cp .env.example .env
# Éditer .env : POSTGRES_*, JWT_SECRET, ADMIN_EMAIL, ADMIN_PASSWORD
```

Variables importantes (voir `.env.example`) :

| Variable | Description |
|----------|-------------|
| `POSTGRES_*` | Connexion PostgreSQL |
| `JWT_SECRET` | Secret JWT (long, aléatoire) |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | Compte admin créé au seed |
| `OLLAMA_ENABLED` | `true` / `false` |
| `OLLAMA_BASE_URL` | ex. `http://host.docker.internal:11434` |
| `OLLAMA_MODEL` | Modèle résumé de scan |
| `OLLAMA_ASSIST_MODEL` | Modèle Assistance IA |
| `ALERT_WEBHOOK_URL` | Webhook optionnel (nouveaux high) |

### 4. Démarrer

```bash
cd app
docker compose up -d --build
```

Services :

| Service | Rôle |
|---------|------|
| `portal` | UI sur le port **80** |
| `api` | FastAPI + migrations Alembic + seed |
| `worker` | Exécution des scans |
| `redis` | File d’attente |

Contrôle :

```bash
curl http://localhost/api/health
# → {"status":"ok","service":"wcvs-api"}
```

Ouvrir le portail : `http://IP_DU_SERVEUR/`

---

## Parcours utilisateur

1. **Inscription / connexion**
2. Choisir la **langue d’interface** (sélecteur)
3. **Ajouter un site** (URL) → **vérification** de propriété (mécanisme prévu par l’API sites)
4. Régler la **langue du rapport** du site (client final)
5. Lancer un scan **Quick** ou **Deep**
6. Consulter score, Fix first, journal, constats, diff
7. Exporter **Markdown** ou **Print / PDF**
8. (Expert) **Assistance IA** sur un constat → guide de remédiation
9. (Admin) Console `/admin` : stats, utilisateurs, changement de plan

---

## Structure du dépôt

```
WCVS/
├── README.md                 ← ce fichier
├── journal-de-bord.html      ← notes de chantier (dev)
├── scripts/
│   ├── 01-install-base.sh    # Postgres, Docker, Ollama
│   ├── 02-deploy-app.sh      # Déploiement Compose
│   ├── enable-ollama.sh
│   ├── enable-expert-assist.sh
│   └── deploy-i18n.ps1       # Sync rapide Windows → serveur
└── app/
    ├── docker-compose.yml
    ├── .env.example
    ├── api/                  # FastAPI + worker + scanner
    │   ├── Dockerfile
    │   ├── requirements.txt
    │   ├── alembic/
    │   └── wcvs/
    │       ├── main.py
    │       ├── worker.py
    │       ├── routers/      # auth, sites, scans, assist, admin…
    │       └── scanner/      # moteur passif + i18n règles
    ├── portal/               # React (Vite)
    └── worker/               # doc pointeur (code = api/wcvs)
```

---

## API (aperçu)

Base : `/api` — documentation interactive : `http://IP:8000/docs` (si l’API est exposée) ou via le reverse-proxy du portal.

| Zone | Exemples |
|------|----------|
| Auth | login, register, locale utilisateur |
| Sites | CRUD, vérification, locale du site |
| Scans | démarrer, détail, historique, export `.md` / `.html` |
| Alerts | historique, webhook |
| Ignores | baseline (accepter un constat) |
| Assist | guide IA (Expert) |
| Admin | stats, plans, utilisateurs |
| Health | `GET /api/health` |

Authentification : **JWT** Bearer.

---

## Développement local

### API

```bash
cd app/api
python -m venv .venv
source .venv/bin/activate   # Windows : .venv\Scripts\activate
pip install -r requirements.txt
# Configurer DATABASE_URL / Redis / .env
alembic upgrade head
python -m wcvs.seed
uvicorn wcvs.main:app --reload --port 8000
```

Worker (autre terminal) :

```bash
python -m wcvs.worker
```

Tests :

```bash
pytest
```

### Portail

```bash
cd app/portal
npm install
npm run dev
```

---

## Sécurité & éthique

- **Scans passifs uniquement** : pas d’exploitation active, pas de payload offensif.
- Ne scanner que des sites dont vous êtes **propriétaire** ou pour lesquels vous avez une **autorisation**.
- Les rapports peuvent contenir des **preuves techniques** : ne pas les publier sans revue.
- Ne **jamais** committer le fichier `app/.env` (secrets JWT, mots de passe DB).

---

## État du projet

Produit en cours (v0.1) :

- [x] Moteur de scan passif multi-modules  
- [x] Score, diff, baseline, alertes / webhook  
- [x] Résumé Ollama + Assistance IA Expert  
- [x] Portail FR/EN/ES/DE + locale par site  
- [x] Console admin (changement de plan manuel)  
- [ ] Paiement Stripe  
- [ ] E-mails d’alerte  

---

## Licence

À définir (ex. MIT / propriétaire). Ajoutez ici votre licence avant publication publique.

---

## Auteur

Projet **WCVS** — scanner de posture web pour agences et indépendants.

