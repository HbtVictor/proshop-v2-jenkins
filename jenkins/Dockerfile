# ═══════════════════════════════════════════════════════════════
#  Image Jenkins custom — ajoute le client Docker CLI au-dessus de
#  l'image officielle. Le démon Docker reste celui de l'hôte (DooD)
#  grâce au socket monté en volume dans docker-compose.yml.
# ═══════════════════════════════════════════════════════════════

FROM jenkins/jenkins:lts-jdk17

USER root

# Installation : docker CLI + docker compose plugin + nodejs (pour les stages Install/Test)
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates curl gnupg lsb-release \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg \
      | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
 && chmod a+r /etc/apt/keyrings/docker.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release; echo $VERSION_CODENAME) stable" \
      > /etc/apt/sources.list.d/docker.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends \
      docker-ce-cli docker-compose-plugin \
 && curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
 && apt-get install -y --no-install-recommends nodejs \
 && rm -rf /var/lib/apt/lists/*

# Retour à l'utilisateur jenkins par défaut. On le laissera tourner en
# root via `user: root` dans le compose (nécessaire pour le socket Docker).
USER jenkins
