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

> _Sections P.2 à P.9 — à compléter au fil des étapes._
