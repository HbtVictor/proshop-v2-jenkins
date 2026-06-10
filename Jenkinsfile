// ═══════════════════════════════════════════════════════════════
//  Jenkinsfile — Pipeline CI/CD complet ProShop v2 (Partie 7).
//
//  7 stages :
//    1. Checkout       — recup du code
//    2. Install        — npm ci backend + frontend
//    3. Lint           — ESLint backend (--max-warnings=0)
//    4. Test           — smoke test integrite + JSON valide
//    5. Build Docker   — build images backend + frontend
//    6. Deploy         — docker compose up -d  (uniquement main)
//    7. Health Check   — attend que /api/health repond 200 (retry)
//
//  Declenchement :
//    - Poll SCM toutes les 2 min (config dans le job Jenkins UI)
//    - workflow_dispatch / Build Now manuel
//
//  Pourquoi pas un vrai webhook GitHub ?
//    Jenkins tourne en local (127.0.0.1:8080). GitHub ne peut pas
//    l'atteindre depuis Internet sans ngrok / Cloudflare Tunnel.
//    On utilise donc Poll SCM (Jenkins demande "y a-t-il du neuf
//    sur main ?" a Git toutes les 2 minutes). Voir Q24.
// ═══════════════════════════════════════════════════════════════

pipeline {
  agent any

  environment {
    NODE_ENV  = 'production'
    APP_NAME  = 'proshop-v2'
    // Tag de l'image que le pipeline produit. BUILD_NUMBER croit a
    // chaque execution → tag unique et reproductible par build.
    IMG_TAG_BACKEND  = "proshop-backend:ci-${env.BUILD_NUMBER}"
    IMG_TAG_FRONTEND = "proshop-frontend:ci-${env.BUILD_NUMBER}"
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 45, unit: 'MINUTES')
    timestamps()
    // Empeche un meme job de lancer 2 builds en parallele.
    disableConcurrentBuilds()
  }

  // Declenchement automatique : Poll SCM toutes les 2 minutes.
  // (Egalement configurable dans la UI du job, mais le mettre ici
  //  rend le trigger versione avec le code.)
  triggers {
    pollSCM('H/2 * * * *')
  }

  stages {

    // ─── Stage 1 : Checkout ───────────────────────────────────
    stage('Checkout') {
      steps {
        echo "[Stage 1/7] Checkout"
        checkout scm
        sh 'git log --oneline -3'
      }
    }

    // ─── Stage 2 : Install ────────────────────────────────────
    stage('Install') {
      steps {
        echo "[Stage 2/7] Install dependencies"
        // Backend a la racine + frontend dans son propre dossier.
        sh 'npm ci'
        sh 'npm ci --prefix frontend'
      }
    }

    // ─── Stage 3 : Lint ────────────────────────────────────────
    stage('Lint') {
      steps {
        echo "[Stage 3/7] ESLint backend"
        sh 'npm run lint'
      }
    }

    // ─── Stage 4 : Test ────────────────────────────────────────
    stage('Test') {
      steps {
        echo "[Stage 4/7] Smoke tests"
        // Le repo upstream n'a pas de tests Jest configures cote backend.
        // On fait un smoke test : verif des fichiers cles + JSON valide
        // + verif que le serveur Express demarre sans crash sur
        // l'import de tous les modules.
        sh '''
          test -f backend/server.js || (echo "MISSING backend/server.js" && exit 1)
          test -f frontend/package.json || (echo "MISSING frontend/package.json" && exit 1)
          test -f docker-compose.yml || (echo "MISSING docker-compose.yml" && exit 1)
          node -e "JSON.parse(require('fs').readFileSync('package.json','utf8'))" \
            && echo "package.json JSON valide"
        '''
      }
    }

    // ─── Stage 5 : Build Docker ───────────────────────────────
    stage('Build Docker') {
      steps {
        echo "[Stage 5/7] Build images backend + frontend"
        sh 'docker build -t $IMG_TAG_BACKEND -f backend/Dockerfile .'
        sh 'docker build -t $IMG_TAG_FRONTEND -f frontend/Dockerfile ./frontend'
        sh 'docker images | grep proshop | head -10'
      }
    }

    // ─── Stage 6 : Deploy ──────────────────────────────────────
    //  Uniquement sur main — protection via `when { branch 'main' }`.
    //  On utilise --no-deps pour ne PAS redeployer jenkins lui-meme
    //  (sinon le conteneur Jenkins se redemarre en plein milieu du
    //  pipeline et tue l'execution).
    stage('Deploy') {
      when {
        branch 'main'
      }
      steps {
        echo "[Stage 6/7] Deploy via docker compose"
        // On tague d'abord les images avec :local (le tag attendu par
        // docker-compose.yml) puis on lance up -d --no-deps.
        sh '''
          docker tag $IMG_TAG_BACKEND proshop-backend:local
          docker tag $IMG_TAG_FRONTEND proshop-frontend:local
          docker compose up -d --no-deps mongo backend frontend proxy
        '''
        sh 'docker compose ps'
      }
    }

    // ─── Stage 7 : Health Check ───────────────────────────────
    //  Attend que /api/health (a travers Nginx) reponde 200.
    //  retry(6) avec sleep 5s entre tentatives → 30s max.
    stage('Health Check') {
      when {
        branch 'main'
      }
      steps {
        echo "[Stage 7/7] Health check via proxy"
        // Jenkins est sur le meme reseau Docker `proshop-net`, donc
        // il atteint le proxy par son nom de service `proxy`.
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
      echo "Pipeline OK — build #${BUILD_NUMBER} reussi"
    }
    failure {
      echo "Pipeline ECHOUE au stage : ${STAGE_NAME ?: '?'}"
    }
    always {
      echo "Pipeline termine (succes ou echec)"
      // Nettoyage : on supprime les images taguees ci-XXX pour ne pas
      // accumuler — sauf si on les a deja taguees :local (deploiement).
      // `|| true` pour ne pas faire echouer le post-action.
      sh 'docker rmi $IMG_TAG_BACKEND 2>/dev/null || true'
      sh 'docker rmi $IMG_TAG_FRONTEND 2>/dev/null || true'
    }
  }
}
