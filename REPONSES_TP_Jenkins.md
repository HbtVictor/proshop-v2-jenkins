# Rapport TP Jenkins — ProShop v2 — Pipeline CI/CD

> 9 parties × 31 questions. Ce document accompagne le code (Dockerfile, docker-compose.yml,
> nginx.conf, Jenkinsfile) versionné dans ce repo.

---

## P.1 — Audit de l'application existante

### Architecture observée

ProShop v2 est une application e-commerce MERN (MongoDB + Express + React + Node.js) avec
trois composants techniques distincts :

| Composant       | Tech                                                 | Port  | Rôle                                                                                |
| --------------- | ---------------------------------------------------- | ----- | ----------------------------------------------------------------------------------- |
| **Frontend**    | React 18, Redux Toolkit, React Router, react-scripts | 3000  | UI client : catalogue, panier, checkout, profil utilisateur                         |
| **Backend**     | Node.js + Express 4 (ES Modules)                     | 5000  | API REST `/api/products`, `/api/users`, `/api/orders`, `/api/upload`                |
| **Base**        | MongoDB (driver Mongoose 7)                          | 27017 | Persistance produits, utilisateurs, commandes                                       |

**Auth** : JWT signés (jsonwebtoken) + cookies (cookie-parser).  
**Paiement** : SDK PayPal côté front et back.  
**Upload images** : middleware Multer écrit sur le filesystem (`/uploads/` en dev,
`/var/data/uploads` en prod).

Variables d'environnement requises (vu dans `.env.example`) :
`PORT`, `MONGO_URI`, `JWT_SECRET`, `PAYPAL_CLIENT_ID`, `PAYPAL_APP_SECRET`, `PAYPAL_API_URL`.

### Limites identifiées (4 catégories)

| Catégorie             | Limites                                                                                                                                                                                                                                                                |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Déploiement**       | Pas de Dockerfile. Pas de docker-compose. Lancement local = installer Node + MongoDB + 2 `npm install`. Aucune commande "one-liner" pour démarrer. Le backend sert lui-même le build React en `NODE_ENV=production` (couplage monolithique).                            |
| **Sécurité**          | Variables sensibles (`JWT_SECRET`, clés PayPal) chargées via `dotenv` sans validation de schéma : si une variable manque, l'app démarre quand même avec un secret vide. Pas de healthcheck endpoint = aucun moyen d'auto-détecter un crash. MongoDB exposé sans authentification par défaut. |
| **Scalabilité**       | Architecture monolithique : un seul process Node sert l'API et les fichiers statiques. Impossible de scaler front et back indépendamment. Pas de file d'attente, pas de cache. Le filesystem local pour les uploads empêche tout scaling horizontal (chaque instance aurait ses propres images). |
| **Observabilité**     | Aucun logger structuré (juste `console.log` natif). Aucune métrique exposée (Prometheus, etc.). Pas de tracing. En cas d'incident, le seul outil est `docker logs` ou les logs Render/Heroku.                                                                          |

### Architecture cible (avec Jenkins au centre)

```
                  ┌──────────────────────┐
                  │     Développeur      │
                  │   git push origin    │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │       GitHub         │
                  │  (proshop-v2-jenkins)│
                  └──────────┬───────────┘
                             │ webhook
                             ▼
                  ┌──────────────────────┐
                  │       JENKINS        │
                  │  ┌────────────────┐  │
                  │  │ Checkout       │  │
                  │  │ Install        │  │
                  │  │ Lint           │  │
                  │  │ Test           │  │
                  │  │ Build Docker   │  │
                  │  │ Deploy Compose │  │
                  │  │ Health Check   │  │
                  │  └────────────────┘  │
                  └──────────┬───────────┘
                             │ docker compose up -d
                             ▼
        ┌─────────────────────────────────────────────┐
        │           Docker Host (Stack)               │
        │                                             │
        │   ┌─────────┐ :80    seul port exposé       │
        │   │  NGINX  │ ◀──── reverse proxy ─┐        │
        │   └────┬────┘                      │        │
        │        │                           │        │
        │   ┌────▼──────┐   ┌──────────────┐ │        │
        │   │ frontend  │   │   backend    │◀┘        │
        │   │  React    │   │  Express     │          │
        │   │  :3000    │   │  :5000       │          │
        │   └───────────┘   └──────┬───────┘          │
        │                          │                  │
        │                   ┌──────▼──────┐           │
        │                   │  MongoDB    │           │
        │                   │  (volume)   │           │
        │                   └─────────────┘           │
        │                                             │
        │   ─── proshop-net : réseau Docker dédié ──  │
        └─────────────────────────────────────────────┘
```

### Réponses Q1-Q4

**Q1 — Pourquoi une app qui fonctionne en local peut-elle échouer en production ?**

Trois causes concrètes (parmi de nombreuses possibles) :

1. **Versions différentes des dépendances système.** En local, on a Node 20.4 ; sur le serveur,
   Node 16. Certaines syntaxes ES2022 (optional chaining côté assignment, `Array.findLast`,
   etc.) plantent. Idem pour Python, OpenSSL, glibc — tout ce qui n'est pas géré par
   `package.json` peut diverger silencieusement.

2. **Variables d'environnement absentes ou différentes.** En local, on a un `.env` complet ; sur
   le serveur, l'admin a oublié `JWT_SECRET`. L'app démarre quand même mais signe les tokens
   avec une valeur vide → tous les utilisateurs sont déconnectés au premier redémarrage. Ou
   pire : `NODE_ENV=production` change le comportement (le backend sert le build React, certaines
   librairies désactivent les warnings, etc.) — un comportement non testé.

3. **Chemins et permissions filesystem.** L'app fait `app.use('/uploads', express.static('/var/data/uploads'))`
   en prod. Si ce dossier n'existe pas, ou si l'utilisateur du process n'a pas le droit
   d'écrire dedans, les uploads plantent. En local, le dev a souvent les droits root sans s'en
   rendre compte.

Autres causes classiques : différences de timezone, base de données vide ou avec un schéma
non-migré, charge réseau, proxy d'entreprise qui filtre certaines URL, certificats SSL absents,
DNS interne différent.

**Q2 — Risques d'un déploiement manuel**

- **Erreurs humaines** : oubli d'un `npm run build`, oubli de redémarrer un service, mauvais
  fichier `.env` copié, push sur le mauvais serveur (staging au lieu de prod). Chaque étape
  manuelle = une chance d'erreur. Une procédure de 10 étapes a en pratique une probabilité
  d'erreur élevée si elle est exécutée à la main, surtout à 23h un vendredi soir.

- **Traçabilité nulle** : "qui a déployé quoi et quand ?" → seul le sysadmin qui a fait le
  déploiement le sait. Impossible de répondre à "depuis quand cette config est en prod ?" sans
  croiser les logs SSH, l'historique bash, et la mémoire de l'équipe. En cas d'audit (PCI-DSS,
  RGPD, ISO 27001), c'est rédhibitoire.

- **Temps de réaction en cas d'incident** : si la prod tombe à 18h, il faut joindre la personne
  qui a fait le dernier déploiement, lui demander ce qu'elle a fait, et espérer qu'elle se
  souvienne. Un rollback manuel prend 15-30 minutes. Avec un pipeline CI/CD versionné, un
  rollback est `git revert + push` (= 2 minutes).

- **Bus factor** : si la seule personne qui sait déployer est en vacances, on ne peut pas
  patcher un bug critique. Le manual deploy est un point de défaillance organisationnel.

**Q3 — « Ça marche sur ma machine » et le mécanisme Docker**

L'expression désigne le phénomène où une application fonctionne dans l'environnement du
développeur (sa machine, son OS, ses dépendances installées) mais échoue ailleurs. C'est le
symptôme d'un **environnement d'exécution implicite** : le dev tient pour acquis des choses qui
ne sont pas documentées (version de Node, libs système, fichiers présents, variables d'env,
ports libres, droits filesystem, etc.).

Docker résout ce problème via le concept d'**image** : une image Docker contient **tout** ce
qu'il faut pour exécuter l'application — le système de fichiers (noyau Linux exclu), les
binaires, les libs, les variables d'env définies, les fichiers source, le user, la commande de
démarrage. Cette image est **un artefact binaire reproductible** : à partir du même
`Dockerfile`, on obtient la même image, qui produit le même comportement sur n'importe quelle
machine ayant un Docker Engine.

Concrètement, on transforme un environnement implicite (la machine du dev) en une description
explicite (le Dockerfile). Ce qui n'est pas dans l'image n'est pas dans l'application. Le
problème "ça marche sur ma machine" devient "ça marche dans cette image, et cette image marche
partout".

**Q4 — Impact business d'une panne d'1h un vendredi soir**

Sur une plateforme e-commerce moyennement fréquentée (disons 1000 commandes/jour, panier moyen
50€) :

- **Pertes directes** : 1 heure de pic vendredi soir = environ 5% du chiffre quotidien.
  ~2 500€ de chiffre d'affaires perdu en commandes non passées. Pour ShopNow à plus grande
  échelle, ça peut représenter des dizaines de milliers d'euros.

- **Pertes indirectes** : les clients qui voient un site cassé partent sur la concurrence et
  ne reviennent souvent pas. Le coût d'acquisition client (~30-80€) est gaspillé sur ces
  visites converties à 0. Le "churn par incident" est mesurable et significatif.

- **Réputation et confiance** : un client qui n'arrive pas à finaliser son panier le vendredi
  soir poste sur Twitter / un avis Trustpilot. Le SEO du site peut être impacté à moyen terme
  (le crawler Google détecte le 500 = signal de qualité dégradée).

- **Coûts opérationnels** : équipe d'astreinte mobilisée (souvent à tarif majoré). Support
  client surchargé le lendemain pour répondre aux questions ("ma commande est-elle passée ?").
  Possibles dédommagements à offrir (codes promo).

- **Risque réglementaire** : si la panne affecte la facturation, des transactions peuvent être
  débitées sans qu'une commande soit créée → litiges, rétrocession.

Sur un vendredi soir en particulier, c'est doublement coûteux : c'est un pic de trafic ET la
résolution doit attendre lundi si l'équipe n'a pas d'astreinte (= 60h de pannes).

---

## P.2 — Conteneurisation avec Docker

### Fichiers livrés

- `backend/Dockerfile` — image Node.js 20-alpine, multi-stage (deps install séparée du runtime),
  port 5000, healthcheck via `wget`, exécution en user non-root `node`.
- `frontend/Dockerfile` — multi-stage React : stage builder Node 20-alpine qui produit le bundle,
  stage runtime Nginx 1.27-alpine qui sert les fichiers statiques. **L'image finale ne
  contient pas Node.js du tout.**
- `frontend/nginx.conf` — configuration interne Nginx pour le SPA React (fallback `index.html`).
- `.dockerignore` à la racine — exclut `node_modules`, `.env`, `.git`, etc. du contexte de build.

### Choix techniques justifiés

1. **Base `alpine`** : 5-10 MB pour l'image de base contre 200+ MB pour `node:20`. Surface
   d'attaque réduite (moins de packages = moins de CVE).
2. **Multi-stage** : seules les dépendances de production atterrissent dans l'image finale
   (option `--omit=dev` au `npm ci`). Côté frontend, on va plus loin : l'image finale ne contient
   ni Node, ni `node_modules`, juste un Nginx avec le bundle statique compilé.
3. **`COPY package*.json ./` AVANT `COPY . .`** : profite du cache layer Docker. Tant que les
   manifests ne changent pas, le layer `npm ci` est réutilisé entre builds.
4. **`USER node` / `USER nginx`** : exécution en user non-root (cf. Q7).
5. **`HEALTHCHECK` intégré** : Docker peut détecter qu'un conteneur est en panne fonctionnelle
   (et pas seulement crashé) — utile pour Compose `depends_on: condition: service_healthy`.
6. **`EXPOSE` documentaire** : on documente le port ouvert dans l'image, sans publier
   automatiquement vers l'extérieur (c'est Compose qui décide quoi publier).

### Réponses Q5-Q8

**Q5 — Image vs Conteneur (analogie)**

Une **image** Docker est une **recette** figée : la liste d'instructions et le résultat
binaire qui décrivent un état initial complet (filesystem, libs, variables d'env, commande de
démarrage). Elle est en lecture seule, immutable, reproductible. On peut la stocker, la
versionner, la pousser sur une registry.

Un **conteneur** est une **instance vivante** d'une image : un processus en cours d'exécution
sur le noyau hôte, isolé par des namespaces Linux. Il a son propre cycle de vie (created →
running → stopped → removed), il peut accumuler des changements sur son filesystem (mais
ceux-ci sont perdus à la suppression sauf si on a monté un volume).

Analogie : l'image est le **plan d'un appartement** (architecte, immuable, dupliquable). Le
conteneur est l'**appartement habité** (avec des meubles, des odeurs, des locataires qui
entrent et sortent). À partir d'un même plan, on peut construire 1, 10 ou 1000 appartements
identiques au départ qui divergeront par l'usage.

**Q6 — Pourquoi séparer frontend / backend / DB en conteneurs distincts ?**

Plusieurs raisons techniques et opérationnelles :

1. **Scaling indépendant.** En cas de charge, on veut souvent 5 instances du backend et 1
   instance du frontend (ou l'inverse). Si tout est dans un conteneur, on ne peut scaler que le
   bloc entier — gâchis de RAM/CPU.

2. **Mises à jour indépendantes.** On peut déployer un nouveau backend sans toucher au
   frontend ni à la DB (rolling update, blue-green). Si tout est mélangé, chaque déploiement
   redémarre tout, y compris la DB.

3. **Sécurité par isolation.** Un compromis du frontend (XSS, dépendance npm vérolée) ne donne
   pas accès au système de fichiers du backend ni à la DB. Avec un seul conteneur, c'est game
   over à la première brèche.

4. **Images officielles maintenues.** MongoDB a son image officielle, maintenue, patchée
   contre les CVE. Si on installe MongoDB dans un conteneur Node, on doit gérer nous-mêmes
   l'image — perte de temps et risque.

5. **Cycle de vie différent.** La DB doit persister (volume monté). Le backend est
   stateless et peut être recréé à volonté. Le frontend est complètement stateless. Mélanger
   ces 3 cycles de vie est une mauvaise abstraction.

6. **12-Factor App** : règle "treat backing services as attached resources" — la DB est une
   ressource à laquelle l'app se connecte par URL, pas un sous-composant interne.

**Q7 — Pourquoi ne pas exécuter un conteneur en root ?**

Par défaut, le `USER` d'un Dockerfile est `root` (UID 0). Si on ne le change pas, le process
dans le conteneur tourne en root sur le noyau hôte.

Risque concret : **escalade de privilèges en cas d'évasion du conteneur**. Bien que Docker
isole les conteneurs via des namespaces Linux, cette isolation a régulièrement des failles
(historiquement : CVE-2019-5736 `runc breakout`, CVE-2022-0185, etc.). Si une faille permet à
un attaquant de sortir du conteneur, et qu'il était root dedans, il est **root sur la machine
hôte** — il peut lire `/etc/shadow`, modifier le kernel, déployer un rootkit, accéder aux
autres conteneurs.

Autre angle : même sans évasion, certaines opérations sont autorisées en root dans le
conteneur (modifier les capabilities, monter des fichiers du host si `--privileged` est
activé, etc.). Le principle of least privilege impose qu'un process n'ait que les permissions
strictement nécessaires.

Solution : `USER <non-root-user>` dans le Dockerfile. Les images officielles Node, Nginx, etc.
fournissent un utilisateur dédié (`node`, `nginx`, UID 1000-101). Si on a besoin d'écrire dans
un dossier, on chown ce dossier vers cet utilisateur dans le Dockerfile (`RUN chown -R
node:node /app`).

**Q8 — Pourquoi optimiser la taille des images Docker ?**

- **Sécurité** : moins de packages installés = moins de surface d'attaque. Une image alpine
  (5 MB de base) a 50× moins de paquets qu'une image Ubuntu (200+ MB). Statistiquement, moins
  de CVE applicables. C'est l'argument numéro 1.

- **Vitesse de déploiement** : pour mettre à jour un service, chaque nœud du cluster doit
  pull la nouvelle image. Sur 10 nœuds avec une image de 500 MB sur une connexion 100 Mbps, on
  parle de plusieurs dizaines de secondes de cumul. Avec une image de 80 MB, c'est instantané.
  Sur un déploiement multi-régions ou edge (CDN à plusieurs PoPs), l'impact est multiplié.

- **Coûts d'infrastructure** : les registries (Docker Hub, GHCR, ECR) facturent au volume
  stocké et au transfert. Une image divisée par 5, c'est aussi 5× moins de coût de bande
  passante quand 1000 nœuds la pull en parallèle.

- **Démarrage à froid plus rapide** : sur Kubernetes ou en serverless containers (AWS Fargate,
  Google Cloud Run), le temps de "cold start" inclut le download de l'image. Une image de 80
  MB démarre 5× plus vite qu'une de 500 MB.

- **Lisibilité et maintenabilité** : un Dockerfile court et optimisé est plus simple à
  auditer. On voit immédiatement ce qui est dans l'image. Une image énorme cache souvent une
  accumulation historique non nettoyée.

Techniques d'optimisation appliquées dans ce TP : base `alpine`, multi-stage, `--omit=dev`,
`.dockerignore`, ordre des `COPY` pour le cache layer.

---

## P.3 — Orchestration avec Docker Compose

### Fichiers livrés

- `docker-compose.yml` — 3 services (`mongo`, `backend`, `frontend`) + 1 volume nommé
  `proshop-mongo-data` + 1 réseau dédié `proshop-net`.
- `.env.example` — template documenté des variables d'env.
- `.env` (LOCAL, non commité) — valeurs réelles.

### Choix d'architecture

1. **Service `mongo` SANS `ports:` publiés** (Q11). MongoDB n'expose son port 27017 que dans
   le réseau Docker interne `proshop-net`. Le backend l'atteint via `mongodb://mongo:27017` (DNS
   automatique fourni par Docker pour le nom de service). Aucun client extérieur ne peut s'y
   connecter — ni par accident, ni par scan.

2. **`depends_on: condition: service_healthy`** sur le backend. Compose démarre d'abord mongo,
   attend que son healthcheck (`db.adminCommand('ping')`) renvoie OK, **puis** démarre le
   backend. Sans cette condition, le backend tenterait de se connecter alors que mongo est
   encore en train d'initialiser ses fichiers de données → crash + restart loop.

3. **Healthcheck mongo** via `mongosh --eval "db.adminCommand('ping').ok"`. C'est la commande
   officielle recommandée par MongoDB Inc. pour valider qu'un nœud est prêt.

4. **Volume nommé `proshop-mongo-data`** monté sur `/data/db` (le path par défaut du dataset
   MongoDB dans l'image officielle). Le `name:` explicite préfixe pas le projet, donc le volume
   survit à un `docker compose down` standard.

5. **Réseau `proshop-net` en `driver: bridge`** — explicite plutôt que d'utiliser le bridge par
   défaut. Cela permet d'avoir un DNS interne propre et d'isoler clairement du reste de Docker
   sur la machine hôte.

6. **`env_file: .env`** pour le backend, avec **surcharge `environment:`** pour les valeurs
   non-secrètes (`NODE_ENV`, `PORT`, `MONGO_URI`) qui ont des valeurs par défaut sûres. Si le
   `.env` est absent, le backend démarre quand même avec les bonnes valeurs (mais sans clés
   PayPal donc paiement KO).

### Validation locale

```bash
docker compose up -d
docker compose exec backend node backend/seeder.js   # injecte 6 produits + 2 users
curl http://localhost:5000/api/products | head       # 6 produits retournés
```

À l'issue du `docker compose up -d` :

| Conteneur          | État              | Port hôte publié |
| ------------------ | ----------------- | ---------------- |
| `proshop-mongo`    | Up (healthy)      | (aucun — interne)|
| `proshop-backend`  | Up (healthy)      | 5000             |
| `proshop-frontend` | Up                | 3000             |

### Réponses Q9-Q11

**Q9 — Pourquoi Docker Compose plutôt que des `docker run` manuels ?**

Lancer 3 conteneurs avec des `docker run` séparés implique :

- 3 commandes longues à taper sans se tromper (image, ports, env, volumes, network…).
- Gérer manuellement l'ordre de démarrage (attendre que mongo soit prêt avant de lancer le
  backend).
- Créer manuellement le réseau (`docker network create proshop-net`) et chaque volume.
- Synchroniser ces commandes entre développeurs (un dev a `--name proshop-back`, un autre
  `--name proshop-backend` → bordel).

Docker Compose résout ça en transformant cette configuration en un **fichier déclaratif
versionné** (`docker-compose.yml`) :

- **Une seule commande** lance toute la stack : `docker compose up -d`.
- **Une seule commande** arrête tout : `docker compose down`.
- **Reproductibilité** : le fichier est dans le repo, tous les devs ont la même config.
- **Ordre de démarrage géré** déclarativement par `depends_on` avec healthcheck conditions.
- **Réseaux et volumes créés automatiquement** au premier `up`.
- **Variables d'environnement gérées** via `env_file` et `environment`.
- **Logs centralisés** : `docker compose logs -f` montre les logs de tous les services en
  parallèle, par ordre chronologique.

C'est aussi la base de Docker Swarm et de Kubernetes (qui utilisent des fichiers déclaratifs
similaires) : apprendre Compose, c'est un pas vers l'orchestration plus complexe.

**Q10 — À quoi sert un volume Docker ?**

Un volume Docker est un **espace de stockage persistant** géré par Docker, indépendant du
cycle de vie des conteneurs. Il vit sur le filesystem hôte (typiquement sous
`/var/lib/docker/volumes/<name>`) mais Docker s'occupe de tout : création, montage, snapshots,
backup, etc.

**Sans volume persistant pour MongoDB** : le conteneur Mongo écrit ses fichiers de données
dans son propre filesystem éphémère (la "writable layer" du conteneur). À chaque `docker
compose down` puis `up`, le conteneur est recréé from scratch — **toutes les données sont
perdues**. Pour une DB, c'est catastrophique : les produits, les utilisateurs, les commandes,
tout disparaît.

**Avec le volume `proshop-mongo-data`** monté sur `/data/db` : Mongo écrit dans le volume, qui
est stocké en dehors du conteneur. Le conteneur peut être détruit et recréé sans perte. C'est
exactement ce qu'on a observé après le `docker compose down && up` : les 6 produits seedés
étaient toujours là.

Autres usages des volumes : partager des fichiers entre conteneurs (logs, uploads), conserver
les caches entre builds, monter de la config en lecture seule, partager un répertoire entre
l'hôte et un conteneur (bind mount, `./local-dir:/app/data`).

**Q11 — Pourquoi un réseau Docker dédié plutôt que le bridge par défaut ?**

Sur le bridge par défaut de Docker :

- **Pas de DNS automatique entre conteneurs.** Sur le bridge default, les conteneurs ne
  peuvent s'atteindre que par adresse IP (qui change à chaque redémarrage), ou avec l'option
  `--link` (deprecated). Sur un réseau dédié custom, chaque conteneur est résolvable par son
  nom de service (DNS interne automatique) — c'est exactement comment le backend trouve
  `mongo:27017`.

- **Tous les conteneurs partagent le même bridge default.** Si on lance d'autres stacks Docker
  sur la même machine, elles voient nos conteneurs et peuvent les pinger / scanner. Avec un
  réseau dédié, **seuls nos services se voient**. C'est une vraie isolation logique.

- **Pas de contrôle des règles iptables.** Le bridge default a une config NAT fixe que Docker
  gère sans qu'on puisse intervenir. Avec un réseau custom, on peut ajouter des labels, des
  drivers IPAM, des sous-réseaux dédiés, etc.

En pratique : créer un réseau dédié coûte 0 (un seul `networks:` block dans le compose) et
apporte une isolation, un DNS propre et une lisibilité réelles. C'est la pratique standard
pour toute stack multi-conteneurs.

---

## P.4 — Reverse Proxy avec Nginx

### Fichiers livrés

- `nginx/nginx.conf` — configuration du reverse proxy : 3 `location` (`/api/`, `/uploads/`,
  `/`), 3 `upstream` (frontend, backend), headers `Host`, `X-Real-IP`, `X-Forwarded-For`,
  `X-Forwarded-Proto`.
- `docker-compose.yml` mis à jour — service `proxy` (image `nginx:1.27-alpine`) qui monte
  `./nginx/nginx.conf` en read-only. **Les `ports:` des services frontend et backend ont été
  supprimés** : ils ne sont plus joignables que depuis le réseau Docker interne.

### Architecture après P.4

```
   internet / localhost
            │
            ▼ port 80 (seul port public)
   ┌─────────────────────────┐
   │   proshop-proxy         │
   │   (nginx:1.27-alpine)   │
   └────┬─────────┬──────────┘
        │         │
   /    │         │  /api/* + /uploads/*
        ▼         ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │frontend│  │backend │──▶ mongo  │
   └────────┘  └────────┘  └────────┘
   (3000)      (5000)        (27017)
   non publié  non publié    non publié
```

Plus rien n'est exposé à l'extérieur sauf le port 80 du proxy.

### Vérification

Depuis l'extérieur de la stack Docker :

| URL                                 | Statut | Comportement attendu                       |
| ----------------------------------- | ------ | ------------------------------------------ |
| `http://127.0.0.1/`                 | 200    | Bundle React (frontend) servi via Nginx    |
| `http://127.0.0.1/api/products`     | 200    | JSON des 6 produits seedés (backend)       |
| `http://127.0.0.1/images/airpods.jpg` | 200    | Image produit (servie par frontend public/) |
| `http://127.0.0.1:3000`             | refus  | Frontend non publié                        |
| `http://127.0.0.1:5000`             | refus  | Backend non publié                         |
| `http://127.0.0.1:27017`            | refus  | Mongo non publié                           |

### Réponses Q12-Q13

**Q12 — Pourquoi utiliser un reverse proxy ?**

Quatre bénéfices distincts :

1. **Point d'entrée unique** = surface d'attaque réduite. Un seul port (80) à exposer au monde
   extérieur, donc une seule porte à protéger (firewall, WAF, rate limiting). Sans reverse
   proxy, il faudrait exposer 3 ports différents (frontend 3000, backend 5000, plus tard
   potentiellement metrics 9090, etc.), chacun étant une surface d'attaque indépendante.

2. **Abstraction de l'architecture interne**. Côté client (navigateur), il n'y a qu'**une**
   origine : `https://shop.mondomaine.com`. Le client ignore qu'en interne, l'API est sur un
   conteneur Node et le frontend sur Nginx avec un bundle React. Si demain on remplace le
   backend Node par du Go, ou si on splitte le frontend en micro-front-ends, **le client ne
   change pas une ligne**.

3. **Terminaison TLS centralisée**. C'est sur le reverse proxy qu'on installe le certificat
   HTTPS (Let's Encrypt, etc.). Les conteneurs internes peuvent rester en HTTP simple (réseau
   Docker isolé). Sans reverse proxy, il faudrait gérer le TLS dans chaque service — duplication
   de config, certificats à renouveler partout, et certains services (Node natif) gèrent mal
   TLS.

4. **Cross-cutting concerns** centralisés : logs d'accès unifiés, compression gzip, cache HTTP,
   rate limiting, blocage géo-IP, redirection HTTP→HTTPS, headers de sécurité (CSP, HSTS,
   X-Frame-Options). Ces fonctions sont implémentées **une fois** dans Nginx et s'appliquent à
   tous les services en aval.

Bénéfices supplémentaires : load balancing (si on a 3 instances backend, Nginx round-robin),
maintenance window (on peut router vers une page de maintenance le temps d'un déploiement),
CORS (gérer les en-têtes Origin proprement).

**Q13 — Proxy vs Reverse Proxy**

Les deux relaient des requêtes HTTP, mais dans des directions opposées et pour des publics
différents.

**Proxy (forward proxy)** : se place **devant les clients**. Le client envoie sa requête au
proxy, qui la transmet au serveur web. Le serveur ne voit que l'IP du proxy.
- Cas d'usage : un proxy d'entreprise sur le réseau interne (Squid) qui filtre les requêtes
  sortantes des employés vers Internet (bloque les sites adultes, journalise les accès,
  applique le filtrage selon des règles RH/sécurité). Autre exemple : un VPN qui agit comme
  proxy pour anonymiser le trafic d'un utilisateur.

**Reverse proxy** : se place **devant les serveurs**. Le client envoie sa requête au reverse
proxy, qui décide à quel serveur interne la transmettre. Le client ignore que des serveurs
existent derrière.
- Cas d'usage : notre Nginx ProShop. Autre exemple : Cloudflare, qui agit comme reverse proxy
  géant devant des millions de sites web — il termine le TLS, applique un WAF, sert un cache
  CDN, puis transmet aux serveurs d'origine.

Schéma résumé :

```
FORWARD PROXY :     [Client] → [Proxy] → [Internet] → [Serveur web]
                    (Le proxy protège/contrôle le client)

REVERSE PROXY :    [Internet] → [Reverse Proxy] → [Serveur(s) internes]
                                (Le proxy protège/oriente les serveurs)
```

Dans les deux cas, c'est exactement le **même logiciel** qui peut faire l'un ou l'autre
(Nginx, HAProxy, Traefik...). C'est la position dans l'architecture qui définit le rôle.

---

## P.5 — Jenkins : Installation & Premier job

### Mise en place

Jenkins est ajouté à `docker-compose.yml` comme un service supplémentaire :

```yaml
jenkins:
  image: jenkins/jenkins:lts-jdk17
  container_name: proshop-jenkins
  user: root
  networks:
    - proshop-net
  ports:
    - '8080:8080'
  volumes:
    - jenkins-home:/var/jenkins_home
    - /var/run/docker.sock:/var/run/docker.sock
```

Choix techniques justifiés :

- **`lts-jdk17`** : version LTS (Long Term Support) recommandée pour la production,
  avec OpenJDK 17 (LTS Java).
- **Volume nommé `jenkins-home`** monté sur `/var/jenkins_home` : persiste **toute** la
  configuration Jenkins (utilisateurs, jobs, credentials, historique des builds, plugins).
- **Socket Docker monté** (`/var/run/docker.sock`) : approche **Docker-out-of-Docker (DooD)**.
  Jenkins n'a pas son propre démon Docker imbriqué — il utilise le démon de l'hôte. Cela
  permet à Jenkins de `docker build` et `docker compose up` sans complexité supplémentaire
  (cf. Q22 du TP).
- **`user: root`** : nécessaire pour lire le socket Docker monté (qui appartient à root sur la
  plupart des hôtes).
- **Port 8080 publié** : seule porte d'accès à l'UI Jenkins. Le port 50000 (agents distants)
  n'est pas publié car on travaille avec l'agent intégré.
- **Réseau partagé `proshop-net`** : Jenkins peut atteindre les autres services par leur nom
  (`mongo`, `backend`, etc.) — utile pour les health checks dans le pipeline P.7.

### Workflow de setup réalisé

1. `docker compose up -d jenkins` → conteneur démarre.
2. Récupération du mot de passe initial :
   `docker exec proshop-jenkins cat /var/jenkins_home/secrets/initialAdminPassword`.
3. Ouverture de http://127.0.0.1:8080 → saisie du mot de passe.
4. Choix **"Install suggested plugins"** → Jenkins installe automatiquement Git, Pipeline,
   Workflow Aggregator, Credentials, etc.
5. Création de l'utilisateur admin avec un mot de passe personnel.
6. Vérification de la présence des plugins **Docker Pipeline** et **Git** dans
   *Manage Jenkins → Plugins → Installed*.
7. Création d'un premier job **Freestyle** :
   - Nom : `proshop-freestyle-test`
   - Source Code Management : Git, URL `https://github.com/HbtVictor/proshop-v2-jenkins.git`,
     branch `*/main`.
   - Build step : *Execute shell* avec `ls -la && cat package.json | head -20`.
8. Clic **Build Now** → boule bleue verte (success). Lecture des logs via *Console Output*.

### Réponses Q14-Q17

**Q14 — Qu'est-ce que Jenkins, et différence avec GitHub Actions ?**

Jenkins est un **serveur d'automatisation open-source** créé en 2011 (forké du projet
Hudson). Il automatise des tâches répétitives liées au développement : build, tests, packaging,
déploiement, notifications, etc. C'est l'outil historique du CI/CD on-premise.

Problème qu'il résout : avant Jenkins, déployer une appli en équipe demandait à un humain
d'exécuter manuellement une séquence de commandes (npm install, tests, build, scp vers le
serveur, redémarrer le service...). C'est lent, erratique, non traçable. Jenkins automatise
cette séquence et la déclenche sur événement (commit Git, cron, webhook).

Différences avec GitHub Actions :

| Critère                    | Jenkins                              | GitHub Actions                       |
| -------------------------- | ------------------------------------ | ------------------------------------ |
| **Hébergement**            | Auto-hébergé (on-premise / cloud privé) | Cloud GitHub par défaut (runners gérés) |
| **Couplage avec le SCM**   | Indépendant (Git, SVN, Mercurial...) | Couplé à GitHub                      |
| **Config**                 | UI + Jenkinsfile (Groovy DSL)        | Fichiers YAML                        |
| **Plugins**                | 1800+ plugins, écosystème mature     | Marketplace d'actions réutilisables  |
| **Coût**                   | Gratuit (mais coût d'infra à payer)  | Gratuit jusqu'à un quota, puis payant |
| **Contrôle**               | Total (réseau interne, secrets, OS)  | Limité (sandbox GitHub)              |
| **Maintenance**            | À ta charge (mises à jour, sécu...)  | GitHub gère tout                     |

Le choix dépend du contexte : Jenkins quand on a besoin de contrôle (banques, gov,
data-sensitive), GitHub Actions quand on veut zéro maintenance et que GitHub est déjà le SCM.

**Q15 — Qu'est-ce qu'un "job" Jenkins ? Freestyle vs Pipeline ?**

Un **job** Jenkins (renommé "Project" puis "Item" dans les versions récentes) est une unité
d'automatisation : un ensemble d'étapes que Jenkins exécute en séquence, déclenché par un
événement ou manuellement. Chaque job a sa propre config, son historique de builds, ses
artefacts, ses logs.

Deux familles principales :

**Freestyle Project** : configuration **uniquement via l'UI web** Jenkins. On clique pour
ajouter le SCM, des build steps (Execute shell, Invoke Gradle...), des post-actions, etc.
Avantages : courbe d'apprentissage faible, idéal pour démarrer. Inconvénients : la config est
stockée uniquement dans Jenkins (un dump XML), donc **non versionnée avec le code**. Si on
perd le serveur Jenkins, on perd la définition du job. Difficile de faire de l'audit ou du
rollback de la config.

**Pipeline Project** : configuration via un **fichier `Jenkinsfile` versionné** dans le repo
Git du projet (en Groovy DSL). C'est l'approche **Pipeline as Code**. Avantages : la config
suit le code (chaque branche peut avoir son propre pipeline), revues en PR possibles,
historique git, reproductibilité, possibilité de définir des stages en parallèle, de gérer des
conditions complexes, etc. C'est la pratique recommandée en production.

Pour un débutant, on commence avec un Freestyle pour comprendre le mécanisme, puis on passe
rapidement à Pipeline pour les vrais projets.

**Q16 — Pourquoi Jenkins est-il préféré en grande entreprise ?**

Trois raisons principales :

1. **Auto-hébergement et contrôle des données.** Les grandes entreprises (banques, défense,
   santé, énergie) ont des contraintes réglementaires fortes : RGPD, PCI-DSS, HIPAA, ANSSI, etc.
   Le code source et les credentials de production ne peuvent pas transiter par les serveurs
   d'un tiers (GitHub, GitLab.com). Jenkins se déploie dans le datacenter interne, sur des VM
   internes, derrière le firewall corporate.

2. **Maturité et extensibilité.** L'écosystème de plugins Jenkins (1800+) couvre des cas
   ultra-spécifiques : intégration avec mainframes IBM, outils de tests ServiceNow, bases
   Oracle, déploiements sur F5 BIG-IP, etc. Sur les solutions cloud (GitHub Actions, GitLab
   CI), ces intégrations exotiques manquent ou nécessitent du code custom.

3. **Coût à l'échelle.** GitHub Actions facture par minute-runner. Pour une entreprise qui
   fait 10 000 builds/mois × 30 min chacun, la facture grimpe vite (plusieurs k€/mois).
   Jenkins sur des machines internes a un coût marginal nul après l'investissement initial. On
   peut faire tourner 50 builds en parallèle sans surcoût.

Autres raisons : interopérabilité avec n'importe quel SCM (pas verrouillé à GitHub),
historique long (certaines entreprises ont 10+ ans de jobs Jenkins, migration vers autre chose
= coût énorme), expertise interne disponible (anciens admins Jenkins formés depuis 2012),
écosystème de support commercial (CloudBees vend du support Jenkins).

**Q17 — Pourquoi monter un volume sur `/var/jenkins_home` ?**

`/var/jenkins_home` est le **répertoire racine de toute la configuration Jenkins** :

- Définitions des jobs (XML serialisé)
- Comptes utilisateurs et permissions
- Credentials (mots de passe, tokens API, clés SSH, etc., chiffrés via le master key Jenkins)
- Historique des builds (logs, artefacts, durées, qui a déclenché quoi)
- Plugins installés et leur config
- Master key de chiffrement de Jenkins
- Configuration globale (URL, SMTP, agents distants...)

Sans volume persistant, **tout cela est dans le filesystem éphémère du conteneur**. À la
moindre recreation du conteneur (`docker compose down && up`, mise à jour de l'image, crash et
redémarrage), Jenkins repart **from scratch** : aucun job, aucun utilisateur, aucun
credentials, aucun historique. Il faudrait tout reconfigurer manuellement. C'est inacceptable
en production.

Avec le volume nommé `jenkins-home` monté sur ce path, tout l'état Jenkins est stocké en
dehors du conteneur (dans `/var/lib/docker/volumes/proshop-jenkins-home/_data` sur l'hôte).
On peut détruire le conteneur, mettre à jour l'image vers une nouvelle version de Jenkins, le
recréer avec le même volume — toute la config est conservée intacte.

C'est aussi le bon endroit pour faire des sauvegardes : un `tar` du volume = backup complet
de Jenkins (à faire à froid, conteneur stoppé). Pour restaurer dans un autre environnement,
on extrait le tar dans un nouveau volume et on remonte le tout.

