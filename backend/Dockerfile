# ═══════════════════════════════════════════════════════════════
#  Dockerfile — Backend ProShop v2 (Node.js / Express)
#  Multi-stage : on installe les deps prod uniquement dans l'image finale.
# ═══════════════════════════════════════════════════════════════

# ─── STAGE 1 : Dépendances production ──────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app

# Copier les manifests d'abord pour profiter du cache Docker tant que
# package.json / package-lock.json ne change pas.
COPY package*.json ./

# `npm ci --omit=dev` installe uniquement les dépendances de production
# (ignore devDependencies comme nodemon, concurrently).
RUN npm ci --omit=dev


# ─── STAGE 2 : Runtime ─────────────────────────────────────────
FROM node:20-alpine AS runtime

# Variables d'env utiles + signaux par défaut
ENV NODE_ENV=production
ENV PORT=5000

WORKDIR /app

# Copier les node_modules construits dans le stage précédent
COPY --from=deps /app/node_modules ./node_modules

# Copier le code source du backend uniquement
COPY package*.json ./
COPY backend ./backend

# Sécurité : on switch sur l'utilisateur non-root `node` fourni par
# l'image officielle (UID 1000).
USER node

# Documentation : on expose le port 5000 (Express).
EXPOSE 5000

# Healthcheck : Express répond `404 notFound` pour les routes inconnues,
# mais répond bien sur `/api/products` (route publique). On utilise wget
# (présent dans alpine) pour un check léger.
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD wget -qO- http://localhost:5000/api/products > /dev/null || exit 1

# Le seeder n'est pas exécuté ici — il sera lancé manuellement via
# `docker compose run backend npm run data:import` une fois MongoDB up.
CMD ["node", "backend/server.js"]
