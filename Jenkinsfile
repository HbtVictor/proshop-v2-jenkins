// ═══════════════════════════════════════════════════════════════
//  Jenkinsfile — Pipeline CI/CD complet ProShop v2 (Parties 7 + 8).
//
//  8 stages :
//    1. Checkout       — recup du code
//    2. Install        — npm ci backend + frontend
//    3. Lint           — ESLint backend (--max-warnings=0)
//    4. Test           — smoke test integrite + JSON valide
//    5. Build Docker   — build images backend + frontend (tags v1.0, latest, ci-N)
//    6. Push GHCR      — login + push vers ghcr.io (uniquement main)
//    7. Deploy         — docker compose up -d  (uniquement main)
//    8. Health Check   — attend que /api/health repond 200 (retry)
//
//  Declenchement :
//    - Poll SCM toutes les 2 min (Q24 : webhook non possible en local)
//    - Build Now manuel
//
//  Strategie de tagging (Partie 8) :
//    - ci-${BUILD_NUMBER} : tag unique par build CI (utile pour debug)
//    - ${APP_VERSION}     : tag de version immuable (ex: v1.0)
//    - latest             : tag mouvant (toujours = derniere release main)
// ═══════════════════════════════════════════════════════════════

pipeline {
  agent any

  environment {
    NODE_ENV     = 'production'
    APP_NAME     = 'proshop-v2'
    APP_VERSION  = 'v1.0'

    // GHCR registry — l'image est lowercase (requis par GHCR)
    REGISTRY     = 'ghcr.io'
    GHCR_OWNER   = 'hbtvictor'

    // Tags Docker locaux (referencent l'image apres build, avant push)
    IMG_BACKEND_BASE  = "${REGISTRY}/${GHCR_OWNER}/proshop-backend"
    IMG_FRONTEND_BASE = "${REGISTRY}/${GHCR_OWNER}/proshop-frontend"
    IMG_BACKEND_CI    = "${IMG_BACKEND_BASE}:ci-${env.BUILD_NUMBER}"
    IMG_FRONTEND_CI   = "${IMG_FRONTEND_BASE}:ci-${env.BUILD_NUMBER}"
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 45, unit: 'MINUTES')
    timestamps()
    disableConcurrentBuilds()
  }

  triggers {
    pollSCM('H/2 * * * *')
  }

  stages {

    // ─── Stage 1 : Checkout ───────────────────────────────────
    stage('Checkout') {
      steps {
        echo "[Stage 1/8] Checkout"
        checkout scm
        sh 'git log --oneline -3'
      }
    }

    // ─── Stage 2 : Install ────────────────────────────────────
    stage('Install') {
      steps {
        echo "[Stage 2/8] Install dependencies"
        sh 'npm ci --include=dev'
        sh 'npm ci --include=dev --prefix frontend'
      }
    }

    // ─── Stage 3 : Lint ────────────────────────────────────────
    stage('Lint') {
      steps {
        echo "[Stage 3/8] ESLint backend"
        sh 'npm run lint'
      }
    }

    // ─── Stage 4 : Test ────────────────────────────────────────
    stage('Test') {
      steps {
        echo "[Stage 4/8] Smoke tests"
        sh '''
          test -f backend/server.js || (echo "MISSING backend/server.js" && exit 1)
          test -f frontend/package.json || (echo "MISSING frontend/package.json" && exit 1)
          test -f docker-compose.yml || (echo "MISSING docker-compose.yml" && exit 1)
          node -e "JSON.parse(require('fs').readFileSync('package.json','utf8'))" \
            && echo "package.json JSON valide"
        '''
      }
    }

    // ─── Stage 5 : Build Docker + tagging ──────────────────────
    //  On build chaque image une fois, puis on ajoute plusieurs tags :
    //    - ci-${BUILD_NUMBER} (unique par build)
    //    - ${APP_VERSION}     (ex: v1.0)
    //    - latest             (mouvant — toujours la derniere)
    stage('Build Docker') {
      steps {
        echo "[Stage 5/8] Build images + multi-tag"
        sh '''
          docker build -t ${IMG_BACKEND_CI} -f backend/Dockerfile .
          docker tag ${IMG_BACKEND_CI} ${IMG_BACKEND_BASE}:${APP_VERSION}
          docker tag ${IMG_BACKEND_CI} ${IMG_BACKEND_BASE}:latest

          docker build -t ${IMG_FRONTEND_CI} -f frontend/Dockerfile ./frontend
          docker tag ${IMG_FRONTEND_CI} ${IMG_FRONTEND_BASE}:${APP_VERSION}
          docker tag ${IMG_FRONTEND_CI} ${IMG_FRONTEND_BASE}:latest

          docker images | grep -E "proshop-(backend|frontend)" | head -10
        '''
      }
    }

    // ─── Stage 6 : Push GHCR ───────────────────────────────────
    //  Uniquement sur main. Utilise un credential Jenkins de type
    //  "Username with password" — l'username GitHub + un PAT
    //  (scope write:packages). Le PAT n'apparait JAMAIS dans les logs.
    stage('Push GHCR') {
      when {
        branch 'main'
      }
      steps {
        echo "[Stage 6/8] Push images vers ${REGISTRY}"
        withCredentials([usernamePassword(
            credentialsId: 'ghcr-credentials',
            usernameVariable: 'GHCR_USER',
            passwordVariable: 'GHCR_TOKEN')]) {
          // login via stdin pour ne pas exposer le token sur la ligne de commande
          sh '''
            echo "$GHCR_TOKEN" | docker login ${REGISTRY} -u "$GHCR_USER" --password-stdin
            docker push ${IMG_BACKEND_BASE}:${APP_VERSION}
            docker push ${IMG_BACKEND_BASE}:latest
            docker push ${IMG_FRONTEND_BASE}:${APP_VERSION}
            docker push ${IMG_FRONTEND_BASE}:latest
            docker logout ${REGISTRY}
          '''
        }
      }
    }

    // ─── Stage 7 : Deploy ──────────────────────────────────────
    stage('Deploy') {
      when {
        branch 'main'
      }
      steps {
        echo "[Stage 7/8] Deploy via docker compose"
        // On tague les images avec :local (attendu par docker-compose.yml)
        // puis on lance up -d --no-deps (sans toucher a Jenkins).
        sh '''
          docker tag ${IMG_BACKEND_BASE}:${APP_VERSION} proshop-backend:local
          docker tag ${IMG_FRONTEND_BASE}:${APP_VERSION} proshop-frontend:local
          docker compose up -d --no-deps mongo backend frontend proxy
        '''
        sh 'docker compose ps'
      }
    }

    // ─── Stage 8 : Health Check ───────────────────────────────
    stage('Health Check') {
      when {
        branch 'main'
      }
      steps {
        echo "[Stage 8/8] Health check via proxy"
        retry(6) {
          sh '''
            sleep 5
            wget -qO- http://proxy/api/health
          '''
        }
        echo "Health check OK"
      }
    }

  }

  post {
    success {
      echo "Pipeline OK — build #${BUILD_NUMBER} (${APP_VERSION})"
    }
    failure {
      echo "Pipeline ECHOUE au stage : ${STAGE_NAME ?: '?'}"
    }
    always {
      echo "Pipeline termine — nettoyage des tags ci-${BUILD_NUMBER}"
      // On supprime UNIQUEMENT les tags ci-N (specifiques au build),
      // pas les tags v1.0/latest qui doivent rester pour le deploiement.
      sh 'docker rmi ${IMG_BACKEND_CI} 2>/dev/null || true'
      sh 'docker rmi ${IMG_FRONTEND_CI} 2>/dev/null || true'
    }
  }
}
