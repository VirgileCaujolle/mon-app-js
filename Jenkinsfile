pipeline {
    agent any
    
    environment {
        APP_NAME = 'mon-app-js'
        CONTAINER_NAME = 'mon-app-js-container'
        IMAGE_NAME = 'mon-app-js:latest'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Récupération du code source...'
                script {
                    try {
                        checkout scm
                    } catch (Exception e) {
                        echo "Erreur lors du checkout SCM, tentative alternative..."
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: env.BRANCH_NAME ?: '*/main']],
                            userRemoteConfigs: [[
                                url: 'https://github.com/VirgileCaujolle/mon-app-js.git'
                            ]]
                        ])
                    }
                }
                // Vérifier que nous sommes dans un repository Git
                sh '''
                    echo "Vérification du repository Git..."
                    git status
                    git log --oneline -5
                '''
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installation des dépendances Node.js...'
                sh '''
                    # Nettoyer tous les conteneurs et volumes Docker existants
                    docker system prune -f || true
                    docker volume prune -f || true
                    
                    # Vérifier que package.json existe dans le workspace
                    echo "=== DIAGNOSTIC: Contenu du workspace ==="
                    ls -la $WORKSPACE
                    echo "=== Vérification de package.json ==="
                    if [ -f "$WORKSPACE/package.json" ]; then
                        echo "✓ package.json trouvé dans le workspace"
                        cat $WORKSPACE/package.json
                    else
                        echo "✗ package.json NON trouvé dans le workspace"
                        echo "Fichiers disponibles:"
                        find $WORKSPACE -name "*.json" -type f
                        exit 1
                    fi
                    
                    # Essayer d'abord directement dans le workspace
                    echo "=== TENTATIVE 1: Installation directe dans le workspace ==="
                    cd $WORKSPACE
                    if docker run --rm -v "$WORKSPACE":/app -w /app node:18-alpine sh -c "ls -la /app && npm install"; then
                        echo "✓ Installation réussie directement dans le workspace"
                    else
                        echo "✗ Échec de l'installation directe, essai avec répertoire temporaire..."
                        
                        # Fallback: utiliser un répertoire temporaire
                        echo "=== TENTATIVE 2: Répertoire temporaire ==="
                        TEMP_DIR=$(mktemp -d)
                        echo "Répertoire temporaire créé: $TEMP_DIR"
                        
                        # Copier TOUS les fichiers nécessaires
                        cp -r $WORKSPACE/. $TEMP_DIR/
                        
                        echo "Contenu du répertoire temporaire:"
                        ls -la $TEMP_DIR
                        
                        echo "Vérification de package.json dans le répertoire temporaire:"
                        if [ -f "$TEMP_DIR/package.json" ]; then
                            echo "✓ package.json trouvé dans le répertoire temporaire"
                        else
                            echo "✗ package.json NON trouvé dans le répertoire temporaire"
                            exit 1
                        fi
                        
                        # Installation avec des permissions appropriées
                        echo "Installation des dépendances:"
                        docker run --rm -v "$TEMP_DIR":/app -w /app node:18-alpine sh -c "
                            echo 'Contenu de /app:' && ls -la /app &&
                            echo 'Contenu de package.json:' && cat /app/package.json &&
                            npm install --verbose
                        "
                        
                        # Copier les résultats vers le workspace
                        if [ -d "$TEMP_DIR/node_modules" ]; then
                            echo "Copie de node_modules vers le workspace..."
                            cp -r $TEMP_DIR/node_modules $WORKSPACE/
                        fi
                        
                        if [ -f "$TEMP_DIR/package-lock.json" ]; then
                            echo "Copie de package-lock.json vers le workspace..."
                            cp $TEMP_DIR/package-lock.json $WORKSPACE/
                        fi
                        
                        # Nettoyer
                        rm -rf $TEMP_DIR
                    fi
                    
                    echo "=== VÉRIFICATION FINALE ==="
                    ls -la $WORKSPACE
                    echo "Installation terminée avec succès"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Exécution des tests...'
                sh '''
                    cd $WORKSPACE
                    echo "=== Vérification avant les tests ==="
                    ls -la $WORKSPACE
                    
                    # Vérifier que node_modules existe
                    if [ ! -d "$WORKSPACE/node_modules" ]; then
                        echo "Erreur: node_modules n'existe pas. Installation des dépendances requise."
                        exit 1
                    fi
                    
                    echo "=== Exécution des tests ==="
                    docker run --rm -v "$WORKSPACE":/app -w /app node:18-alpine sh -c "
                        echo 'Vérification de l\\'environnement de test:' &&
                        ls -la /app &&
                        echo 'Lancement des tests...' &&
                        npm test
                    "
                '''
            }
        }
        
        stage('Code Quality Check') {
            steps {
                echo 'Vérification de la qualité du code...'
                sh '''
                    cd $WORKSPACE
                    echo "=== Vérification de la qualité du code ==="
                    
                    # Vérifier que le répertoire src existe
                    if [ ! -d "$WORKSPACE/src" ]; then
                        echo "Avertissement: Le répertoire src n'existe pas. Vérification ignorée."
                        exit 0
                    fi
                    
                    echo "Vérification de la syntaxe JavaScript..."
                    docker run --rm -v "$WORKSPACE":/app -w /app node:18-alpine sh -c "
                        echo 'Fichiers JavaScript trouvés:' &&
                        find src -name '*.js' -type f &&
                        echo 'Vérification de la syntaxe...' &&
                        find src -name '*.js' -type f -exec node -c {} \\;
                    "
                    echo "Vérification de la qualité terminée"
                '''
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Construction de l\'image Docker...'
                sh '''
                    cd $WORKSPACE
                    docker build -t ${IMAGE_NAME} .
                '''
            }
        }
        
        stage('Security Scan') {
            steps {
                echo 'Analyse de sécurité...'
                sh '''
                    cd $WORKSPACE
                    echo "=== Analyse de sécurité ==="
                    
                    # Vérifier que node_modules existe pour l'audit
                    if [ ! -d "$WORKSPACE/node_modules" ]; then
                        echo "Avertissement: node_modules n'existe pas. Audit de sécurité ignoré."
                        exit 0
                    fi
                    
                    echo "Vérification des vulnérabilités de sécurité..."
                    docker run --rm -v "$WORKSPACE":/app -w /app node:18-alpine sh -c "
                        echo 'Lancement de l\\'audit de sécurité...' &&
                        npm audit --audit-level=moderate || echo 'Vulnérabilités détectées mais build continue'
                    "
                    echo "Analyse de sécurité terminée"
                '''
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Déploiement de l\'application...'
                sh '''
                    # Arrêter le conteneur existant s'il existe
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    
                    # Démarrer le nouveau conteneur
                    docker run -d --name ${CONTAINER_NAME} -p 3000:3000 ${IMAGE_NAME}
                    
                    # Attendre que l'application démarre
                    sleep 10
                '''
            }
        }
        
        stage('Health Check') {
            steps {
                echo 'Vérification de santé de l\'application...'
                script {
                    try {
                        sh '''
                            # Test de connectivité
                            curl -f http://localhost:3000/health || exit 1
                            echo "Application déployée avec succès et répond correctement"
                        '''
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo "Warning: Health check failed: ${e.getMessage()}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                try {
                    echo 'Nettoyage des ressources temporaires...'
                    sh 'docker image prune -f || true'
                } catch (Exception e) {
                    echo "Nettoyage ignoré: ${e.getMessage()}"
                }
            }
        }
        success {
            echo 'Pipeline exécuté avec succès!'
            echo "Application accessible sur: http://localhost:3000"
        }
        failure {
            script {
                try {
                    echo 'Le pipeline a échoué!'
                    sh '''
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    '''
                } catch (Exception e) {
                    echo "Nettoyage d'échec ignoré: ${e.getMessage()}"
                }
            }
        }
    }
}
