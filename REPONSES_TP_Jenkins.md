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

---

## P.6 — Pipeline Declarative (Jenkinsfile)

### Fichiers livrés

- **`Jenkinsfile`** à la racine du repo — Pipeline Declarative en 4 stages.
- **`jenkins/Dockerfile`** — image Jenkins custom qui ajoute le client `docker` CLI et
  Node.js 20 au-dessus de l'image officielle (utile pour les stages Install, Test, Build).

### Structure du Jenkinsfile

```groovy
pipeline {
  agent any
  environment { NODE_ENV = 'production'; APP_NAME = 'proshop-v2' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 30, unit: 'MINUTES')
    timestamps()
  }
  stages {
    stage('Checkout') { steps { checkout scm; sh 'git log --oneline -5' } }
    stage('Install')  { steps { sh 'npm ci' } }
    stage('Test')     { steps { sh '... smoke test ...' } }
    stage('Build')    { steps { sh 'docker build -t proshop-backend:ci-${BUILD_NUMBER} -f backend/Dockerfile .' } }
  }
  post {
    success { echo "Pipeline OK - build #${BUILD_NUMBER} reussi" }
    failure { echo "Pipeline ECHOUE - voir les logs ci-dessus" }
    always  { echo "Pipeline termine (succes ou echec)" }
  }
}
```

### Mise en place dans Jenkins

1. New Item → `proshop-pipeline` → Type **Pipeline**.
2. Definition : **Pipeline script from SCM** → Git → URL repo → branche `*/main` → Script path
   `Jenkinsfile`.
3. Save → Build Now.
4. La **Stage View** affiche les 4 stages en colonnes, avec leur durée respective.

### Expérimentation d'un échec volontaire (Q21)

J'ai introduit volontairement `sh 'cette-commande-nexiste-pas-volontairement'` dans le stage
Test, poussé sur main, relancé le pipeline → observation :

- Stages **Checkout** et **Install** passent au vert.
- Stage **Test** échoue immédiatement à la première sh-commande inconnue (`exit code: 127`).
- Le pipeline **s'arrête net** au stage Test : le stage **Build n'est PAS exécuté** (skip
  automatique). La Stage View affiche le 4ème stage en gris/transparent.
- Le `post { failure { ... } }` s'exécute pour logger un message d'erreur.
- Le `post { always { ... } }` s'exécute aussi (succès ou échec).
- Le build est marqué **FAILURE** dans la Build History (boule rouge).

Après correction (`git commit + push`) et nouveau build, les 4 stages repassent au vert.

### Réponses Q18-Q21

**Q18 — Pipeline as Code : avantages**

Le principe **Pipeline as Code** consiste à définir le pipeline CI/CD dans un fichier
versionné dans le repo Git du projet (`Jenkinsfile`), au même titre que le code applicatif.
C'est l'opposé de la config "click-ops" où on configure le job uniquement dans l'UI Jenkins.

Avantages concrets de stocker le Jenkinsfile dans le repo :

1. **Versionning et historique**. Chaque modification du pipeline est tracée par git :
   qui a changé quoi, quand, pourquoi (via le message de commit). On peut faire un `git blame`
   sur une ligne du pipeline pour comprendre son origine. Avec la config UI, on n'a qu'un
   snapshot courant et un journal Jenkins très limité.

2. **Branches indépendantes**. Sur une PR `feat/new-payment`, on peut modifier le Jenkinsfile
   pour ajouter un nouveau stage spécifique à cette feature, sans impacter le pipeline de
   `main`. Une fois la PR mergée, le nouveau pipeline est actif. Sans Pipeline as Code, le
   pipeline est unique et global → on doit faire des modifications conditionnelles fragiles.

3. **Revue de code**. Le Jenkinsfile passe par le processus de PR review classique : un autre
   dev peut commenter, demander des changements, valider. Modifier la config Jenkins en UI
   bypass toute revue.

4. **Reproductibilité et bus factor**. Si on perd le serveur Jenkins (crash, migration, panne
   de disque), le Jenkinsfile reste dans le repo. Recréer le job = pointer le nouveau Jenkins
   vers le repo. La connaissance n'est plus emprisonnée dans l'instance Jenkins.

5. **Tests des changements de pipeline**. On peut tester une modification du Jenkinsfile sur
   une branche feature avant de la merger. Avec la config UI, modifier le pipeline impacte
   immédiatement la prod.

6. **Documentation auto-portée**. Le Jenkinsfile **est** la documentation officielle du
   pipeline. Un nouveau dev qui arrive dans l'équipe lit le Jenkinsfile et comprend tout
   (avec les commentaires). En UI, il faut des screenshots ou un partage d'écran.

**Q19 — Declarative vs Scripted, recommandation pour un débutant**

Jenkins propose **deux syntaxes** pour les Jenkinsfiles, basées toutes deux sur Groovy :

| Critère                | Declarative                                  | Scripted                                |
| ---------------------- | -------------------------------------------- | --------------------------------------- |
| **Syntaxe**            | Structurée : `pipeline { agent { ... } stages { stage('X') { steps { ... } } } }` | Groovy "libre" : `node { stage('X') { sh 'cmd' } }` |
| **Lisibilité**         | Bonne — structure imposée, sections explicites | Variable — dépend de la discipline du dev |
| **Erreurs de syntaxe** | Détectées au parsing avant exécution         | Découvertes uniquement en runtime       |
| **Puissance**          | Limitée (mais ~95% des besoins couverts)     | Totale (Groovy complet : boucles, classes, fonctions) |
| **Validation IDE**     | Plugin Jenkins valide la syntaxe              | Plus difficile                          |
| **Stages parallèles**  | Syntaxe dédiée `parallel { ... }`             | Faisable mais plus verbeux              |
| **Intégration Blue Ocean** | Excellente (parsing structuré)            | Limitée                                 |

Concrètement, **Declarative** ressemble à du YAML structuré avec des sections fixes :
`pipeline`, `agent`, `environment`, `options`, `stages`, `post`. C'est très balisé.

**Scripted** est du Groovy "à la main" : `node { ... }` à la racine, puis on appelle `stage`,
`sh`, `dir`, etc. comme des fonctions Groovy.

**Recommandation pour un débutant** : **Declarative**, sans hésiter. Les raisons :

1. La structure imposée force à respecter les bonnes pratiques (séparation stages / steps /
   post).
2. Les erreurs de syntaxe sont détectées immédiatement, pas en plein milieu d'un build long.
3. La doc officielle Jenkins est plus claire pour Declarative.
4. 95% des pipelines de production tiennent en Declarative.
5. Si on a besoin de Groovy custom (boucles complexes, etc.), on peut **encapsuler dans un
   bloc `script { ... }`** au sein d'un stage Declarative — meilleur des deux mondes.

Scripted reste utile pour des pipelines très dynamiques (ex: déployer N stages selon le contenu
d'un fichier de config), mais c'est rare. On y vient quand on en a vraiment besoin.

**Q20 — Le bloc `post { }`**

Le bloc `post { }` définit des actions à exécuter **après** la fin du pipeline (ou d'un stage
si imbriqué dans `stages.stage`). C'est le mécanisme principal pour faire du *cleanup*, des
notifications, des publications d'artefacts conditionnelles, etc.

Les trois conditions principales :

- **`always { ... }`** : s'exécute **toujours**, quel que soit le résultat (succès, échec,
  abort, instable, etc.). Cas d'usage typiques :
  - Cleanup d'un workspace (`deleteDir()`)
  - Stop d'un conteneur de test (`docker stop test-db`)
  - Publication des logs (`archiveArtifacts 'logs/*.log'`)
  - Métriques de durée envoyées à un système de monitoring

- **`success { ... }`** : s'exécute **uniquement** si le pipeline termine en SUCCESS. Cas
  d'usage :
  - Notification Slack/email "Build #42 vert"
  - Tag de l'image Docker comme `latest` (on ne tague pas latest si le build a foiré)
  - Déclencher un autre pipeline aval (déploiement)

- **`failure { ... }`** : s'exécute **uniquement** si le pipeline termine en FAILURE. Cas
  d'usage :
  - Notification d'incident (page d'astreinte)
  - Rollback automatique d'un déploiement partiel
  - Création automatique d'un ticket dans Jira

Autres conditions disponibles : `aborted`, `unstable`, `changed` (résultat différent du
précédent build), `fixed` (passe de FAILURE à SUCCESS), `regression` (passe de SUCCESS à
FAILURE).

L'ordre d'exécution est garanti : `always` puis `changed` puis le résultat spécifique. Les
conditions sont indépendantes : si on déclare `always` et `success`, les deux s'exécutent en
cas de succès.

**Q21 — Que se passe-t-il quand un stage échoue ?**

Le comportement par défaut d'un pipeline Declarative en cas d'échec d'un stage :

1. **Arrêt immédiat du pipeline**. Le stage en cours retourne un exit code non-zéro (ou lève
   une exception), Jenkins marque le stage en FAILURE.
2. **Les stages suivants ne sont PAS exécutés**. Ils sont marqués comme "Not executed" dans
   la Stage View — visuellement en transparence/gris, alignés avec les autres mais sans
   couleur ni durée.
3. **Le pipeline complet est marqué FAILURE** (boule rouge dans Build History, badge "X" dans
   la Stage View).
4. **Le bloc `post { failure { ... } }` s'exécute** (s'il est défini), pour permettre une
   notification.
5. **Le bloc `post { always { ... } }` s'exécute aussi** (pour le cleanup).

C'est le comportement souhaité 99% du temps : si le test échoue, on ne veut pas builder
l'image, ni la pousser, ni déployer. Le principe : **fail fast**.

Dans la Stage View concrètement, après mon expérimentation Q21 :

```
Checkout    Install    Test          Build
   ✓           ✓          ✗            ·
  3s         11s        2s          (skip)
```

Les stages réussis gardent leur durée. Le stage en échec affiche la croix rouge. Les stages
suivants sont en pointillés / transparents.

Si on **veut absolument** qu'un stage continue malgré l'échec d'un précédent (cas très rare,
ex: publier les logs même en cas de fail), on peut utiliser :
- `catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') { ... }` autour des commandes
- Ou un stage avec `when { not { equals expected: 'FAILURE', actual: currentBuild.result } }`

Mais ce sont des exceptions. Le comportement par défaut "arrêt au premier échec" est la
bonne pratique.

---

## P.7 — Pipeline CI/CD complet (7 stages)

### Architecture

Le `Jenkinsfile` (Partie 7) enrichit celui de P.6 avec 3 stages supplémentaires :

```
1. Checkout       — recupere le code Git (checkout scm)
2. Install        — npm ci backend + frontend (--include=dev)
3. Lint           — npm run lint (eslint backend, --max-warnings=0)
4. Test           — smoke test : fichiers presents + JSON valide
5. Build Docker   — build images backend + frontend (tags ci-${BUILD_NUMBER})
6. Deploy         — docker compose up -d --no-deps  (uniquement main)
7. Health Check   — wget http://proxy/api/health (retry 6×5s)
```

Trois mécanismes critiques :

- **`when { branch 'main' }`** sur les stages 6 et 7 → le déploiement ne s'exécute que sur
  la branche `main`, jamais sur une branche feature ou une PR (Q22).
- **`--no-deps`** sur `docker compose up` → Jenkins ne se redéploie pas lui-même (le conteneur
  Jenkins qui exécute le pipeline est dans la même stack, il s'auto-tuerait sinon).
- **`retry(6) { sleep 5; wget ... }`** sur le Health Check → tolère le temps de redémarrage des
  conteneurs (~30s max).

### Choix d'implémentation : Lint

J'ai installé ESLint v8 + configuré `backend/.eslintrc.json` avec `eslint:recommended`. Ajout
d'un script `lint` dans `package.json` : `eslint backend --max-warnings=0`.

Le fichier `seeder.js` utilise `colors` via la pollution globale `String.prototype.red` (pattern
upstream) — non-lintable, je l'ai exclu via `.eslintignore` à la racine.

### Choix d'implémentation : Health Check

Le backend n'avait pas d'endpoint dédié. J'ai ajouté `GET /api/health` qui retourne
`{status:'ok', service:'proshop-backend', timestamp}` **sans dépendre de MongoDB** (volontaire :
on veut détecter un crash applicatif même si la DB est en panne — sinon health == DB).

Le Dockerfile backend a aussi été mis à jour pour utiliser ce endpoint dans son `HEALTHCHECK`
intégré (au lieu de `/api/products` qui dépend de MongoDB).

### Déclencheur : Poll SCM

Le PDF demande un webhook GitHub → Jenkins. Or Jenkins tourne en localhost (`127.0.0.1:8080`),
GitHub ne peut pas l'atteindre depuis Internet sans tunnel (ngrok ou Cloudflare Tunnel).

J'ai donc opté pour la solution **Poll SCM** : Jenkins demande lui-même à Git toutes les
2 minutes "y a-t-il du neuf sur main ?". C'est versionné dans le Jenkinsfile :

```groovy
triggers {
  pollSCM('H/2 * * * *')   // toutes les 2 min, le 'H' randomise la minute
}
```

**Comportement observé** : après un `git push`, Jenkins déclenche automatiquement un nouveau
build dans les 2 minutes, avec la mention "Started by an SCM change" dans Build History.

Le tradeoff : un webhook serait instantané (~1s), Poll SCM ajoute une latence de 0-120s. En
contrepartie, c'est plus simple à mettre en place et plus résilient (si Jenkins est en panne
au moment du push, il rattrapera au redémarrage). Pour une **démo live**, un vrai webhook via
ngrok est plus impressionnant — mentionné dans la stratégie de présentation orale.

### Validation observée

Premier build avec polling actif :

```
Stage View
─────────────────────────────────────────────────────────────────
Checkout    Install   Lint    Test    Build Docker   Deploy   Health Check
   ✓           ✓        ✓       ✓         ✓             ✓          ✓
  3s         52s       4s      1s        85s           18s         7s
```

Build #N+1 déclenché automatiquement après un `git push` cosmétique :

> _Build History — #N+1 — Started by an SCM change_

### Réponses Q22-Q26

**Q22 — Pourquoi Deploy après Lint et Test ?**

Le pipeline applique la règle du **fail fast** : on veut détecter un problème **avant** de
déployer, pas après. La séquence "Lint → Test → Build → Deploy" est conçue pour échouer le
plus tôt possible :

1. **Lint en premier** : un problème de style ou un `var unused` se détecte en 2 secondes.
   Pourquoi attendre 5 minutes de build Docker avant de découvrir une faute de syntaxe ?
2. **Test ensuite** : si les tests cassent, le code a un bug fonctionnel. Inutile de builder
   l'image qui contiendra ce bug.
3. **Build Docker** : si l'image ne se build pas (Dockerfile cassé, dépendance manquante), il
   n'y a rien à déployer de toute façon.
4. **Deploy en dernier** : à ce stade, tout est validé. On déploie une image dont on est sûr.

**Que se passerait-il sans cet ordre ?**

Si on déploie avant les tests, on met en production un code potentiellement cassé. Les
utilisateurs (ou le système de monitoring) découvrent le bug avant nous. C'est l'équivalent
d'envoyer un produit sans contrôle qualité — on rembourse à grand frais après plutôt que
d'investir 5 minutes en amont.

Pire encore : sans test post-déploiement (notre Health Check stage 7), on pourrait
déployer un binaire qui ne démarre même pas, et ne s'en rendre compte que quand un utilisateur
tape sur une 502. Avec le Health Check, le pipeline reste rouge tant que l'app ne répond pas.

C'est aussi pour ça que `Deploy` a la condition `when { branch 'main' }` : sur une PR ou une
feature branch, on **n'a pas envie** de déployer en production. Seul `main` (la branche
release) déclenche le déploiement.

**Q23 — Gestion des secrets dans Jenkins**

Jenkins propose le **Credentials Manager** (Manage Jenkins → Credentials). On y stocke :
- mots de passe et tokens
- clés SSH
- fichiers de config (kubeconfig, par exemple)
- combinaisons username + mot de passe

Les credentials sont **chiffrés au repos** dans `/var/jenkins_home/credentials.xml` avec la
master key Jenkins. On ne peut pas les lire en clair via l'UI une fois enregistrés (juste les
remplacer).

Dans un Jenkinsfile, on les utilise avec `withCredentials([...])` :

```groovy
stage('Push Docker') {
  steps {
    withCredentials([usernamePassword(
        credentialsId: 'docker-hub-creds',
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS')]) {
      sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
      sh 'docker push monorg/monimage:latest'
    }
  }
}
```

À l'exécution :
- `DOCKER_USER` et `DOCKER_PASS` sont injectées comme variables d'environnement uniquement
  pendant le bloc `withCredentials` — pas dans le reste du pipeline.
- Jenkins **masque automatiquement** la valeur dans les logs (remplacée par `****`), même si
  une commande l'écrit en clair par erreur.

**Pourquoi ne PAS écrire les secrets dans le Jenkinsfile ?**

1. **Le Jenkinsfile est versionné dans Git** — donc le secret se retrouverait dans
   l'historique git public. Même supprimer le secret dans un commit ultérieur ne le retire pas
   de l'historique (`git log -p` le retrouve). Le secret est compromis pour toujours.
2. **Anyone with read access to the repo verrait le secret** — devs juniors, contractors,
   bots, audit, etc.
3. **Pas de masquage dans les logs Jenkins** — si on fait `sh "curl -H 'Authorization: Bearer
   abc123'"`, le token apparaît dans la Console Output.
4. **Rotation impossible** — pour changer le secret, il faut un commit + push. Impensable en
   incident sécurité.

La règle est universelle dans le CI/CD : **secrets dans le secret store, jamais dans le code**.

**Q24 — Webhook GitHub → Jenkins : flux détaillé**

Un webhook est un mécanisme **push** où GitHub notifie Jenkins quand un événement se produit
(par opposition au polling, où Jenkins demande à GitHub si quelque chose a changé). Flux exact :

1. **Le développeur fait `git push origin main`**.
2. GitHub reçoit le push et l'enregistre dans son repo.
3. GitHub consulte la **configuration des webhooks du repo** (Settings → Webhooks).
4. Pour chaque webhook configuré avec l'événement "push", GitHub fait une requête
   HTTP **POST** vers l'URL configurée (typiquement `http://jenkins.example.com:8080/github-webhook/`).
5. Le body de la requête contient un payload JSON volumineux avec : nom du repo, branche,
   liste des commits, auteur, URL de comparaison, etc. GitHub signe la requête avec un secret
   partagé pour que Jenkins puisse vérifier l'origine.
6. **Le plugin "GitHub" de Jenkins** (installé via Manage Plugins) écoute sur `/github-webhook/`.
   Il parse le payload, vérifie la signature.
7. Le plugin identifie **quels jobs Jenkins ont ce repo en SCM** et lance un build pour chacun.
8. Jenkins lance le pipeline, qui exécute `checkout scm`, récupère le dernier commit, etc.

**Avantages vs polling** :
- **Latence** : < 1 seconde après le push (vs 0-120s en polling à 2 min).
- **Économie de bande passante** : pas de poll inutile quand rien ne change.
- **Précision** : on sait exactement quel commit a déclenché le build (via le payload).

**Inconvénients** :
- Nécessite que Jenkins soit **accessible depuis Internet** (donc IP publique + ouverture
  firewall, ou tunnel ngrok / Cloudflare). Bloquant pour notre setup localhost.
- Si Jenkins est down au moment du push, le webhook est **perdu** (GitHub fait quelques
  retries puis abandonne). En polling, Jenkins rattrape au prochain poll après redémarrage.
- Configuration en 2 endroits : côté GitHub (Settings → Webhooks → ajouter l'URL) et côté
  Jenkins (plugin GitHub + secret partagé).

**Pour notre TP**, on utilise polling SCM ; pour la démo live, on activera ngrok.

**Q25 — Pipeline Jenkins vs déploiement manuel : gains concrets**

Comparons un déploiement classique d'une nouvelle version de ProShop :

| Étape                          | Manuel             | Pipeline Jenkins |
| ------------------------------ | ------------------ | ---------------- |
| Pull du code                   | `git pull` (1 min) | automatique      |
| Install dépendances backend    | manuel (1 min)     | automatique      |
| Install dépendances frontend   | manuel (1 min)     | automatique      |
| Lint                           | _souvent oublié_   | systématique     |
| Tests unitaires                | _souvent oublié_   | bloquants        |
| Build des 2 images Docker      | manuel (3 min)     | automatique      |
| `docker compose up`            | manuel (1 min)     | automatique      |
| Vérif que ça marche            | _curl à la main_   | health check auto + retry |
| **Temps humain total**         | **~10 min**        | **~30 secondes** |
| **Risque d'erreur humaine**    | élevé (oublis, fautes de frappe) | nul       |
| **Reproductibilité**           | dépend du dev      | identique        |

**Gains concrets** :

- **Temps humain** : le dev cliquant "Push" récupère 10 minutes de sa journée à chaque
  release. Sur 100 releases par an, c'est ~16h.
- **Fiabilité** : aucune étape n'est oubliée. Sans pipeline, un soir tard, on saute les tests
  pour aller plus vite — et on déploie un bug.
- **Traçabilité** : chaque build a un numéro, un log complet, une stage view avec les durées.
  En cas d'incident, on a une timeline exacte. Manuel = "qui a fait quoi quand ?" sans réponse.
- **Reproductibilité** : la nouvelle développeuse qui arrive lance le pipeline depuis son
  premier jour avec exactement le même résultat que le tech lead. Manuel = onboarding de 2
  semaines pour comprendre les rituels.
- **Audit / conformité** : pour SOC2, ISO 27001, RGPD, on peut prouver que chaque déploiement
  est passé par des contrôles (lint, test, build, health). Manuel = aucune preuve.
- **Confidence** : on déploie le vendredi soir sans peur, parce qu'on sait qu'un health check
  va valider automatiquement et qu'on peut rollback en 1 commande.

**Q26 — Blue Ocean dans Jenkins**

**Blue Ocean** est une UI alternative pour Jenkins, lancée par CloudBees en 2017 (et désormais
en maintenance — son successeur est intégré directement dans l'UI moderne de Jenkins
post-2024). Avant Blue Ocean, l'UI Jenkins était considérée comme datée et difficile à
comprendre pour les nouveaux utilisateurs.

**Problème d'interface résolu** :

L'UI classique Jenkins (héritée de Hudson 2005) montre les builds comme une liste verticale
de jobs, avec un menu de gauche envahissant et une terminologie technique (Workspace, Build
Triggers, Permalinks...). Pour visualiser un pipeline avec 7 stages, il fallait :
- ouvrir le build,
- cliquer sur "Pipeline Steps",
- naviguer dans une arborescence de steps,
- chaque stage occupait une ligne (pas une colonne),
- impossible de voir d'un coup d'œil "quel stage a échoué et pourquoi".

**Blue Ocean** introduit une UI orientée pipeline :
- **Vue principale en colonnes** : chaque stage en colonne verticale, durée, statut visuel.
- **Animations** quand un stage tourne (le rond pulse).
- **Drill-down par stage** : un clic sur un stage en échec ouvre directement les logs de ce
  stage seul, pas tout le Console Output.
- **Vue par branche** : pour un job Multibranch, on voit toutes les branches en parallèle avec
  le statut de chacune.
- **Editor visuel** : créer un Jenkinsfile via drag-and-drop (utile pour les débutants).

C'est essentiellement une **modernisation UX** sans changer le moteur Jenkins. Mêmes jobs,
mêmes pipelines, mêmes plugins — juste une interface plus lisible. Les équipes qui adoptent
Blue Ocean reportent typiquement une réduction du temps de diagnostic d'incident (on voit
immédiatement où ça a foiré) et une meilleure adoption par les profils non-DevOps (frontend
devs, QA, etc.) qui n'aimaient pas l'UI classique.

---

## P.8 — Tagging Docker

### Stratégie de tagging mise en place

Le pipeline Jenkins (stage **Build Docker** + stage **Push GHCR**) produit **trois tags**
pour chaque image (backend et frontend) à chaque build sur `main` :

| Tag                    | Type        | Mutabilité | Cas d'usage                              |
| ---------------------- | ----------- | ---------- | ---------------------------------------- |
| `ci-${BUILD_NUMBER}`   | technique   | immuable   | Debug interne / traçabilité build Jenkins |
| `v1.0`                 | release     | immuable   | Déploiement production reproductible     |
| `latest`               | rolling     | mouvante   | Démo / docs / dev                        |

Les images sont poussées sur **GHCR** (`ghcr.io/hbtvictor/proshop-backend` et
`ghcr.io/hbtvictor/proshop-frontend`).

### Credentials Jenkins

Conformément à Q23, le PAT GitHub (scope `write:packages`) est stocké dans Jenkins via
**Manage Jenkins → Credentials → System → Global** :
- Kind : `Username with password`
- Scope : Global
- Username : `HbtVictor`
- Password : `ghp_...` (PAT GitHub, jamais dans le code)
- ID : `ghcr-credentials`

Le pipeline y accède via `withCredentials([usernamePassword(credentialsId: 'ghcr-credentials', ...)])`,
ce qui rend les variables `GHCR_USER` et `GHCR_TOKEN` disponibles uniquement dans le bloc.
Jenkins masque automatiquement le PAT dans les logs (`****`).

### Procédure de rollback documentée

Voir [`JENKINS_DEPLOYMENT.md`](./JENKINS_DEPLOYMENT.md) section "Procédure de rollback" pour
les 3 scénarios de rollback détaillés (rollback local urgent < 2 min, rollback définitif via
git revert, rollback vers un build CI spécifique).

### Réponses Q27-Q28

**Q27 — Pourquoi versionner les images Docker ?**

Versionner les images Docker = produire un tag **immuable et reproductible** pour chaque
version stable de l'application. Concrètement, on construit une image et on lui attribue un
tag comme `v1.2.3` qui ne sera **jamais** réutilisé pour une autre image.

**Bénéfices** :

1. **Reproductibilité** : `docker pull monapp:v1.2.3` retourne exactement la même image
   aujourd'hui que dans 6 mois. Le SHA256 ne change pas. C'est essentiel pour rejouer un build
   précis en cas d'investigation post-incident.

2. **Rollback rapide et fiable** : si `v1.3.0` est cassée, on déploie `v1.2.3` en une
   commande. On est sûr de retomber exactement sur le comportement précédent — pas une variante
   "à peu près similaire".

3. **Multi-environnement** : staging tourne en `v1.3.0-rc1`, production en `v1.2.3`. On peut
   tester explicitement la version avant de la propager. Sans tags fixes, "staging et prod
   tournent la même version" est une affirmation invérifiable.

4. **Audit et traçabilité** : un ticket support "le bug est en production depuis le 12 juin"
   peut être croisé avec "v1.2.4 a été déployée le 11 juin" → on identifie le commit en cause
   en quelques minutes.

5. **Sécurité** : un scanner Trivy / Snyk peut être tagué sur des versions spécifiques. On
   peut dire "v1.2.3 a des vulnérabilités HIGH connues, v1.2.4 les corrige" — base d'un plan
   de patching.

**Limites du tag `:latest` utilisé seul** :

`:latest` est juste un **pointeur mouvant**. Il ne contient pas d'information de version. Les
problèmes :

- **Imprédictible** : qui sait quelle image est sur `:latest` ? Celle pushée il y a 5
  minutes ? Hier ? La semaine dernière ? Personne ne peut le dire sans aller voir l'historique
  de la registry.
- **Caching d'images** : un nœud Kubernetes peut avoir une vieille image `:latest` en cache
  local et continuer à l'utiliser même si une nouvelle a été pushée. Sans `imagePullPolicy:
  Always`, on ne rejoue jamais le pull → "tu as déployé la nouvelle version ?" "Oui" "Mais
  les logs montrent l'ancienne…".
- **Rollback impossible** : `:latest` est par définition la dernière version. Si la dernière
  est cassée, il n'y a pas de "version d'avant `:latest`" — il faut connaître `:v1.2.3`
  pour rollback.
- **Pas d'historique** : on ne peut pas dire "qu'est-ce qui tournait en prod le 12 juin à
  14h ?" — `:latest` à cette date n'existe plus, il a été écrasé.

`:latest` est utile pour les démos publiques et la doc ("essayez `docker run monapp:latest`")
mais **ne doit jamais être l'unique tag** d'un environnement productif.

**Q28 — Risque de déployer en prod avec `:latest` uniquement**

Scénario concret d'incident :

> 16h30 vendredi. Le dev pousse un commit "fix: refactor petite chose" sur main. Le pipeline
> CI/CD se déclenche, push l'image taguée `:latest` sur la registry. La prod est configurée
> avec `image: proshop-backend:latest` et `imagePullPolicy: Always`. Le déploiement automatique
> redéploie la nouvelle image. Tout est encore vert dans le monitoring (le health check
> répond 200).
>
> 18h00. Le pic d'affluence du vendredi soir arrive. Sous charge, le "petit refactor" provoque
> un memory leak. Les pods commencent à crasher. Le système d'auto-scaling tente de monter de
> nouveaux pods qui crashent à leur tour. Les utilisateurs voient des 502.
>
> 18h15. L'astreinte est appelée. Le dev qui a poussé est parti en weekend. Le tech lead
> arrive, identifie que ça a commencé après 16h30. Il regarde l'historique CI → "ah, il y a eu
> un build à 16h30, peut-être lié ?". Mais il ne sait pas **quelle image** était sur `:latest`
> avant ce build (elle a été écrasée), ni s'il faut rebuild un commit précédent ou pull une
> image qui existerait encore quelque part.
>
> 18h30. Il décide de revert le commit fautif sur main et de pousser. Le pipeline CI tourne
> pour 8 minutes. Au final, il a fallu **2 heures** pour rétablir le service.
>
> 19h00. Service rétabli. Bilan : 90 minutes de panne en plein pic du vendredi soir, ~10K€
> de chiffre d'affaires perdu, des avis Trustpilot négatifs, et le tech lead a passé sa
> soirée à débugger.

**Avec un tagging propre `v1.2.3` + `latest`** :

À 18h15, le tech lead aurait pu faire :
```bash
docker pull ghcr.io/.../proshop-backend:v1.2.2  # version précédente immuable
docker tag ghcr.io/.../proshop-backend:v1.2.2 ghcr.io/.../proshop-backend:latest
kubectl rollout restart deployment/backend
```

Service rétabli en **< 5 minutes**. Sans tag de version, c'est mathématiquement impossible.

C'est pour ça que toutes les pipelines de production sérieuses publient au minimum 2 tags
par release : un immuable (`vX.Y.Z`) ET un alias rolling (`latest`), et que les manifests
de déploiement référencent **toujours** le tag immuable.

---

## P.9 — Production : déploiement, panne, rollback

### Tests fonctionnels après déploiement par le pipeline

Après le build Jenkins #N qui a déployé via Compose, les tests suivants ont été effectués :

| Test                                          | Résultat                                                |
| --------------------------------------------- | ------------------------------------------------------- |
| `GET http://127.0.0.1/` (frontend)            | HTTP 200, bundle React servi via Nginx                  |
| `GET http://127.0.0.1/api/health`             | HTTP 200, `{status:'ok', timestamp:...}`                |
| `GET http://127.0.0.1/api/products`           | HTTP 200, JSON des 6 produits seedés                    |
| Navigation frontend → catalogue produits      | OK, les 6 produits s'affichent avec images              |
| Connexion utilisateur (admin@email.com)       | OK (creds créés par le seeder)                          |
| Persistance MongoDB (down + up backend)       | OK, données intactes (volume `proshop-mongo-data`)      |

### Simulation de panne backend (P.9 point 3)

Procédure exécutée :

```bash
# 1. État de départ
curl http://127.0.0.1/api/health   # → 200 {status: 'ok'}

# 2. Arrêt brutal du backend (simulation crash applicatif)
docker stop proshop-backend

# 3. Observation du comportement
curl http://127.0.0.1/                # → 200 (frontend OK, Nginx sert le bundle statique)
curl http://127.0.0.1/api/health      # → 504 Gateway Timeout (puis 502 Bad Gateway)
curl http://127.0.0.1/api/products    # → 502 Bad Gateway

docker logs proshop-proxy --tail 5
# → 2026/06/10 08:31:34 [error] *33 connect() failed (113: Host is unreachable)
#    while connecting to upstream: http://172.21.0.4:5000/api/products
```

**Observations** :

- **Le frontend continue de fonctionner** parce qu'il est servi statiquement par le conteneur
  frontend (Nginx interne) — indépendant du backend.
- **Les routes `/api/*` retournent 502 Bad Gateway** : Nginx ne peut pas atteindre le backend,
  il génère une page d'erreur 502 standard.
- **Les logs Nginx sont explicites** : `connect() failed (113: Host is unreachable)` avec
  l'URL upstream complète. Permet de diagnostiquer en 5 secondes "le backend est down".
- **MongoDB et Jenkins continuent de tourner** : la panne est isolée au service en cause.

### Récupération

```bash
docker start proshop-backend
# → ~10 secondes plus tard :
curl http://127.0.0.1/api/health    # → 200 {status: 'ok'}
```

Temps total de récupération : **~11 secondes** (entre `docker start` et `/api/health 200`).
Aucune perte de données : le volume MongoDB est resté monté tout du long.

### Procédure de rollback vers v1.0

Voir [`JENKINS_DEPLOYMENT.md`](./JENKINS_DEPLOYMENT.md) section "Procédure de rollback" pour
les 3 scénarios complets. Le scénario d'urgence ("v1.1 cassée → revenir à v1.0 en < 5 min") :

```bash
# 1. Stopper les services applicatifs (sans toucher à MongoDB ni Jenkins)
docker compose stop backend frontend proxy

# 2. Re-taguer les images v1.0 (déjà sur GHCR) comme :local
docker tag ghcr.io/hbtvictor/proshop-backend:v1.0 proshop-backend:local
docker tag ghcr.io/hbtvictor/proshop-frontend:v1.0 proshop-frontend:local

# 3. Redémarrer la stack avec les images stables
docker compose up -d --no-deps backend frontend proxy

# 4. Vérifier
curl http://127.0.0.1/api/health
```

**Durée mesurée** : 3-4 minutes (incluant pull si les images ne sont pas en cache local).
Critère du PDF (< 5 minutes) respecté.

### Réponses Q29-Q31

**Q29 — Différence Dev / Staging / Production**

Chacun a un rôle distinct et complémentaire dans le cycle de vie du logiciel :

**Dev (Développement)** : environnement personnel du développeur, sur sa machine.
- **Données** : données factices, base locale (peut être détruite à volonté).
- **Stabilité** : aucune attendue — l'app crash régulièrement parce qu'on code dessus.
- **Public** : un seul développeur (parfois quelques-uns qui partagent un dev shared).
- **Cycle de vie** : redéployé en continu via `npm run dev`, hot-reload activé.
- **Objectif** : itérer rapidement, expérimenter, debug.

**Staging** : environnement **iso-prod** où on valide une release candidate.
- **Données** : copie anonymisée de la prod ou data set de qualification représentatif.
- **Stabilité** : critique — un staging instable empêche de valider les releases.
- **Public** : l'équipe (QA, product owner, parfois client interne).
- **Cycle de vie** : déployé à chaque merge sur `main`, après les tests CI.
- **Objectif** : valider qu'une nouvelle version fonctionne en conditions réelles **avant**
  de la mettre en production.

**Production** : environnement live, utilisé par les vrais clients.
- **Données** : données réelles, sensibles, à protéger absolument.
- **Stabilité** : maximale (SLA 99.9% ou plus).
- **Public** : utilisateurs finaux, des milliers à millions de clients.
- **Cycle de vie** : déployé uniquement après validation staging + approbation manuelle.
- **Objectif** : servir les utilisateurs sans interruption ni dégradation.

Les trois environnements doivent être **identiques au niveau infrastructure** (même version
de Node, mêmes images Docker, même config Nginx, etc.). Seules les variables d'env (URI DB,
clés API, secrets) diffèrent. Cette iso-prod est la clé d'un bon staging : si staging est
légèrement différent, il ne détecte pas tous les bugs.

**Q30 — Pourquoi prévoir un plan de rollback ?**

Un plan de rollback est une **assurance** : la possibilité de revenir rapidement à un état
fonctionnel connu en cas d'incident. C'est une compétence opérationnelle, pas seulement
technique.

**Pourquoi systématiquement le prévoir** :

1. **Murphy's Law** : à un moment, un déploiement va casser quelque chose qui n'avait pas été
   testé. Sans plan de rollback préparé à froid, on improvise dans la panique — recette pour
   aggraver l'incident.
2. **Temps de résolution divisé par 10** : un rollback documenté prend 3-5 minutes. Sans, on
   peut perdre 1-2 heures à diagnostiquer + débugger + déployer un patch.
3. **Confiance pour déployer** : si on sait qu'on peut rollback en 5 min, on déploie plus
   souvent (deploy-on-Friday OK !). Sans rollback, chaque déploiement est anxiogène, on les
   espace, on accumule des risques.
4. **Réversibilité = liberté d'expérimenter** : on peut tester des features risquées en
   staging puis prod, sachant qu'on peut revenir si besoin. C'est la base du déploiement
   progressif (canary, blue-green).

**Scénario réel où le rollback est la bonne décision** :

> Mardi 14h, on déploie `v2.3.0` qui contient une refacto de l'ORM. À 14h30, le monitoring
> signale que la latence p95 des requêtes a doublé (passée de 200ms à 400ms). Le P50 est
> normal. À 14h45, les premiers tickets support arrivent : "le panier prend du temps à charger".
> Le tech lead doit décider en 5 minutes :
>
> **Option A** : Investiguer le code pour identifier la régression de performance. Estimation :
> 2-4 heures (analyse query plan, comparaison des EXPLAIN PostgreSQL, identifier la requête
> N+1, écrire le fix, tester, déployer).
>
> **Option B** : Rollback vers `v2.2.5` (version d'hier, stable). Estimation : 5 minutes.
>
> Le bon choix est **B**. On rollback, on stabilise le service immédiatement. Ensuite, on
> investigue à froid, sans pression, et on prépare un patch `v2.3.1` qu'on déploiera
> correctement quand on aura compris.
>
> Le rollback n'est pas un aveu d'échec — c'est une décision opérationnelle saine qui protège
> les utilisateurs.

**Q31 — Diagnostiquer une panne backend avec les outils du TP**

Démarche en escalade (du plus rapide au plus profond) :

**Étape 1 — Vérifier l'état des conteneurs** (10 secondes)
```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```
Si `proshop-backend` est absent ou `Exited`, on a la cause de premier niveau : le conteneur
ne tourne pas. Si tous sont `Up`, on creuse plus loin.

**Étape 2 — Tester l'endpoint depuis l'extérieur** (10 secondes)
```bash
curl http://127.0.0.1/api/health      # via Nginx
curl http://127.0.0.1/api/products    # endpoint applicatif réel
```
Codes possibles :
- **200** : tout va bien, c'est peut-être un problème côté client.
- **502 Bad Gateway** : Nginx ne peut pas atteindre le backend (backend down ou réseau Docker
  cassé).
- **504 Gateway Timeout** : Nginx atteint le backend mais celui-ci ne répond pas dans le délai
  (backend bloqué sur une requête longue, deadlock, surcharge).
- **500** : backend démarre mais throws une exception (le code lui-même est cassé).

**Étape 3 — Lire les logs Nginx** (30 secondes)
```bash
docker logs proshop-proxy --tail 50
```
Les logs Nginx donnent l'URL upstream tentée et l'erreur précise : `connect() failed (113:
Host is unreachable)` = pas de réseau ; `(110: Operation timed out)` = backend bloqué.

**Étape 4 — Lire les logs backend** (1 minute)
```bash
docker logs proshop-backend --tail 100
```
Trois signaux typiques :
- Pas de logs récents → app crashée ou bloquée
- Stack trace d'erreur → bug applicatif identifiable directement (souvent ligne de code en
  cause)
- `MongoNotConnectedError` → la DB est inaccessible (pas le backend en soi)

**Étape 5 — Inspecter le conteneur de plus près** (1 minute)
```bash
docker inspect proshop-backend --format '{{.State.Status}} {{.State.Health.Status}}'
docker exec proshop-backend wget -qO- http://localhost:5000/api/health
docker stats proshop-backend --no-stream
```
Permet de voir l'état du healthcheck Docker intégré, tester l'endpoint **depuis l'intérieur**
du conteneur (élimine les problèmes réseau), et vérifier la consommation CPU/RAM (un backend
qui sature la RAM va swap et devenir lent).

**Étape 6 — Vérifier la DB** (30 secondes)
```bash
docker exec proshop-mongo mongosh --eval "db.adminCommand('ping')"
docker logs proshop-mongo --tail 50
```
Si MongoDB est down ou en restauration, le backend peut tourner mais répondre 500 sur
toutes les routes qui touchent à la DB.

**Étape 7 — Vérifier le réseau Docker** (1 minute)
```bash
docker network inspect proshop-net
docker exec proshop-proxy ping -c 2 backend
```
Détecte une éventuelle corruption du DNS interne Docker (rare mais possible après un crash
docker engine).

**Étape 8 — Consulter le pipeline Jenkins** (1 minute)
Aller sur http://127.0.0.1:8080 → `proshop-pipeline` → dernier build. Si le pipeline est
rouge, on a probablement la cause racine (un commit récent qui a cassé). Si le pipeline est
vert mais la prod est down, c'est plus subtil (config différente, données en cause, etc.).

**Total : moins de 5 minutes** pour avoir un diagnostic précis grâce à l'observabilité de
base (logs structurés via `docker logs`, healthchecks, isolation par conteneur).

