// ═══════════════════════════════════════════════════════════════
//  Jenkinsfile — Pipeline Declarative ProShop v2 (Partie 6).
//
//  4 stages d'apprentissage :
//    1. Checkout — clone le code depuis Git (fait par Jenkins SCM).
//    2. Install  — installe les dependances backend.
//    3. Test     — verifie integrite des fichiers cles (smoke test).
//    4. Build    — build l'image Docker du backend.
//
//  La partie 7 enrichira ce fichier avec Lint, Deploy, Health Check
//  et un webhook GitHub.
// ═══════════════════════════════════════════════════════════════

pipeline {
  // Sur quel noeud Jenkins le pipeline tourne. `any` = n'importe quel
  // executor disponible (ici l'executor integre du master).
  agent any

  // Variables d'environnement disponibles dans tous les stages.
  environment {
    NODE_ENV  = 'production'
    APP_NAME  = 'proshop-v2'
  }

  // Options globales du pipeline.
  options {
    // Garde uniquement les 10 derniers builds (gain de place).
    buildDiscarder(logRotator(numToKeepStr: '10'))
    // Timeout global au cas ou un stage se bloque.
    timeout(time: 30, unit: 'MINUTES')
    // Affiche le timestamp devant chaque ligne de log.
    timestamps()
  }

  stages {
    // ─── Stage 1 : Checkout ────────────────────────────────────
    stage('Checkout') {
      steps {
        echo "Recuperation du code depuis Git..."
        checkout scm
        sh 'git log --oneline -5'
      }
    }

    // ─── Stage 2 : Install ─────────────────────────────────────
    stage('Install') {
      steps {
        echo "Installation des dependances backend (npm ci)..."
        // `npm ci` au lieu de `npm install` : strict avec package-lock.json,
        // c'est ce qui est recommande en CI.
        sh 'npm ci'
      }
    }

    // ─── Stage 3 : Test ────────────────────────────────────────
    stage('Test') {
      steps {
        echo "Tests de coherence du projet..."
        // ECHEC VOLONTAIRE — Q21 du TP : observer le comportement
        // d'un pipeline qui echoue. Sera corrige au commit suivant.
        sh 'cette-commande-nexiste-pas-volontairement'
        sh '''
          test -f backend/server.js || (echo "MISSING backend/server.js" && exit 1)
          test -f frontend/package.json || (echo "MISSING frontend/package.json" && exit 1)
          test -f docker-compose.yml || (echo "MISSING docker-compose.yml" && exit 1)
          node -e "JSON.parse(require('fs').readFileSync('package.json','utf8'))" \
            && echo "package.json est un JSON valide"
        '''
      }
    }

    // ─── Stage 4 : Build Docker ────────────────────────────────
    stage('Build') {
      steps {
        echo "Build de l'image Docker backend..."
        // On utilise le client docker CLI present dans l'image Jenkins
        // custom + le socket Docker monte → DooD.
        sh 'docker build -t proshop-backend:ci-${BUILD_NUMBER} -f backend/Dockerfile .'
        sh 'docker images | grep proshop-backend | head -3'
      }
    }
  }

  // ─── Actions post-execution ───────────────────────────────────
  post {
    success {
      echo "Pipeline OK - build #${BUILD_NUMBER} reussi"
    }
    failure {
      echo "Pipeline ECHOUE - voir les logs ci-dessus"
    }
    always {
      echo "Pipeline termine (succes ou echec)"
    }
  }
}
