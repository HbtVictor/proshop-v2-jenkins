# ProShop v2 — Pipeline CI/CD Jenkins

> Documentation du pipeline Jenkins ajouté au repo ProShop v2 dans le cadre du TP CI/CD
> M1 DEVFLSTK Caplogy / YNOV. Pour le README applicatif d'origine, voir [`readme.md`](./readme.md).

---

## Architecture de la stack

```
        ┌─────────┐
        │ DEV     │ git push origin main
        └────┬────┘
             │
             ▼
   ┌────────────────────────────────────┐
   │             JENKINS                │
   │  Poll SCM toutes les 2 min         │
   │  ┌──────────────────────────────┐  │
   │  │  Pipeline (8 stages)         │  │
   │  │   Checkout → Install → Lint  │  │
   │  │     → Test → Build Docker    │  │
   │  │     → Push GHCR → Deploy     │  │
   │  │     → Health Check           │  │
   │  └──────────────────────────────┘  │
   └────────────────────────────────────┘
                  │
   ┌──────────────┴──────────────┐
   │                             │
   ▼ ghcr.io/hbtvictor/        ▼ docker compose up -d
   ┌──────────────┐         ┌──────────────────────────┐
   │ GHCR         │         │ Stack ProShop locale     │
   │              │         │  proxy (Nginx, port 80)  │
   │ proshop-     │         │  frontend (React)        │
   │  backend     │         │  backend  (Node/Express) │
   │  :v1.0       │         │  mongo    (volume)       │
   │  :latest     │         └──────────────────────────┘
   │ proshop-     │
   │  frontend    │
   │  :v1.0       │
   │  :latest     │
   └──────────────┘
```

## Stratégie de versioning des images Docker (Partie 8)

Le pipeline produit **trois tags** pour chaque image à chaque build sur `main` :

| Tag                        | Type      | Mutabilité | Cas d'usage                              |
| -------------------------- | --------- | ---------- | ---------------------------------------- |
| `ci-${BUILD_NUMBER}`       | technique | immuable   | Debug interne — chaque build a son tag   |
| `v1.0` (ou autre version)  | release   | immuable   | Déploiement production reproductible     |
| `latest`                   | rolling   | mouvante   | Démo / dev / tutoriels                   |

**Pourquoi 3 tags ?**

- `ci-NN` permet de retrouver précisément quel build CI a produit quelle image (utile pour
  reproduire un bug à partir de l'historique Jenkins).
- `v1.0` est le tag **immuable** qu'on déploie en production. Si je `docker pull
  ghcr.io/hbtvictor/proshop-backend:v1.0` aujourd'hui ou dans 6 mois, je récupère exactement
  la même image (même SHA256, même comportement).
- `latest` est un alias **mouvant** qui pointe toujours sur la dernière version publiée. C'est
  utile pour la doc ("essayez `docker run … :latest`") mais **dangereux en production** : si
  on fait un `docker compose pull` la nuit, on récupère la nouvelle version sans s'en rendre
  compte. **Ne jamais déployer prod avec `:latest`.**

## Endpoints exposés

| URL                                  | Service        | Description                          |
| ------------------------------------ | -------------- | ------------------------------------ |
| `http://127.0.0.1/`                  | frontend       | UI React (catalogue, panier, etc.)   |
| `http://127.0.0.1/api/products`      | backend        | Liste des produits (catalogue)       |
| `http://127.0.0.1/api/health`        | backend        | Health check (status + timestamp)    |
| `http://127.0.0.1/uploads/<file>`    | backend        | Images produits uploadées            |
| `http://127.0.0.1:8080`              | jenkins        | UI Jenkins                           |

## Démarrer la stack en local

```bash
# 1. Clone et entrer dans le repo
git clone https://github.com/HbtVictor/proshop-v2-jenkins.git
cd proshop-v2-jenkins

# 2. Créer le fichier .env (non commité)
cp .env.example .env
# → remplir MONGO_URI, JWT_SECRET, PayPal sandbox creds

# 3. Lancer la stack complète (Mongo + backend + frontend + proxy + Jenkins)
docker compose up -d --build

# 4. Seed la base de données (6 produits + 2 users)
docker compose exec backend node backend/seeder.js

# 5. Accès
# - App ProShop : http://127.0.0.1/
# - API health  : http://127.0.0.1/api/health
# - Jenkins     : http://127.0.0.1:8080/
```

## Configuration Jenkins post-démarrage

1. Récupérer le mot de passe admin initial :
   ```bash
   docker exec proshop-jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
2. Aller sur http://127.0.0.1:8080 → suivre l'assistant.
3. Installer les plugins suggérés + vérifier la présence de **Docker Pipeline** et **Git**.
4. Créer un credential Jenkins de type **Username with password** :
   - ID : `ghcr-credentials`
   - Username : ton login GitHub (ex: `HbtVictor`)
   - Password : un Personal Access Token (scope `write:packages, read:packages`)
5. Créer le job `proshop-pipeline` (type Pipeline) → Pipeline script from SCM → Git → URL du
   repo → Script Path `Jenkinsfile`.

## Procédure de rollback

### Cas 1 — Rollback du déploiement local (urgent — < 2 min)

**Symptôme** : `v1.1` vient d'être déployée, l'app crashe ou `/api/health` répond 500.

```bash
# 1. Stopper la stack courante (ne touche pas à MongoDB ni à Jenkins)
docker compose stop backend frontend proxy

# 2. Re-taguer la version stable précédente comme :local
docker tag ghcr.io/hbtvictor/proshop-backend:v1.0 proshop-backend:local
docker tag ghcr.io/hbtvictor/proshop-frontend:v1.0 proshop-frontend:local

# 3. Redémarrer la stack avec ces images
docker compose up -d --no-deps backend frontend proxy

# 4. Vérifier
curl http://127.0.0.1/api/health
```

Si les tags `:v1.0` n'ont pas été pull localement, faire d'abord :
```bash
docker pull ghcr.io/hbtvictor/proshop-backend:v1.0
docker pull ghcr.io/hbtvictor/proshop-frontend:v1.0
```

### Cas 2 — Rollback définitif via Git

**Symptôme** : le commit qui a introduit le bug doit être annulé pour que les déploiements
futurs n'en souffrent plus.

```bash
# 1. Identifier le commit fautif (ex: avec git log --oneline -5)
git log --oneline -5

# 2. Annuler ce commit (crée un commit "Revert ...")
git revert <SHA_du_commit_fautif>

# 3. Push : déclenche le polling SCM Jenkins (< 2 min) → nouveau build
git push origin main

# 4. Le pipeline rebuilde, taggue et redéploie la version corrigée
# 5. Vérifier le health check sur http://127.0.0.1/api/health
```

### Cas 3 — Rollback complet vers un build CI spécifique

**Symptôme** : on veut déployer exactement le build CI #42 (par exemple parce qu'on a identifié
qu'il était stable).

```bash
# 1. Pull l'image taguee ci-42 depuis GHCR (si elle existe encore)
docker pull ghcr.io/hbtvictor/proshop-backend:ci-42
# (Note : ces tags sont uniquement créés en local par le pipeline — ils ne sont pas pushés
#  vers GHCR par défaut. Pour les pousser, modifier le stage Push GHCR.)

# 2. Tag :local et redeploy
docker tag ghcr.io/hbtvictor/proshop-backend:ci-42 proshop-backend:local
docker compose up -d --no-deps backend
```

## Désactiver le polling temporairement

Si tu veux arrêter le polling SCM (ex: pendant un weekend), commente la section `triggers` du
`Jenkinsfile`, commit, push. Ou désactive le job dans l'UI Jenkins (Disable Project).

## Limitations du setup actuel

- **Pas de webhook GitHub réel** : Jenkins tourne en `127.0.0.1`, GitHub ne peut pas
  l'atteindre. Pour une démo live, exposer Jenkins via ngrok / Cloudflare Tunnel et
  configurer le webhook (voir Q24 dans `REPONSES_TP_Jenkins.md`).
- **`MONGO_URI` sans authentification** : la base est isolée dans le réseau Docker mais
  utilise une connexion non-authentifiée. En production réelle, activer
  `MONGO_INITDB_ROOT_USERNAME/PASSWORD` et l'inclure dans l'URI.
- **PAT GHCR à expiration** : si le token expire, le stage Push GHCR échouera. Renouveler le
  PAT et remettre à jour le credential `ghcr-credentials` dans Jenkins.
