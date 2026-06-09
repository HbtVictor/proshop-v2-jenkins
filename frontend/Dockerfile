# ═══════════════════════════════════════════════════════════════
#  Dockerfile — Frontend ProShop v2 (React via react-scripts)
#  Multi-stage : on build le bundle avec Node, on sert avec Nginx.
#  L'image finale ne contient AUCUN Node.js — juste Nginx + fichiers statiques.
# ═══════════════════════════════════════════════════════════════

# ─── STAGE 1 : Build du bundle React ────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Copier les manifests d'abord (cache Docker)
COPY package*.json ./

# Installer toutes les deps (y compris devDependencies — react-scripts est en deps)
RUN npm ci

# Copier le reste du code et lancer le build de prod
COPY . .

# CI=false : sinon react-scripts considère les warnings comme des erreurs en CI.
# Le bundle produit dans /app/build est statique.
RUN CI=false npm run build


# ─── STAGE 2 : Runtime — Nginx léger qui sert les fichiers statiques ─────
FROM nginx:1.27-alpine AS runtime

# On supprime le default.conf de Nginx et on en met un custom adapté au
# routing client-side de React (toute route inconnue → index.html).
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copier le bundle statique depuis le stage builder
COPY --from=builder /app/build /usr/share/nginx/html

# Sécurité : exécution sous utilisateur non-root (`nginx` UID 101 dans alpine).
# Nginx a besoin d'accéder à /var/cache/nginx et /var/run/nginx.pid → on les chown.
RUN chown -R nginx:nginx /var/cache/nginx /var/log/nginx /etc/nginx/conf.d \
    && touch /var/run/nginx.pid \
    && chown nginx:nginx /var/run/nginx.pid

USER nginx

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/ > /dev/null || exit 1

CMD ["nginx", "-g", "daemon off;"]
