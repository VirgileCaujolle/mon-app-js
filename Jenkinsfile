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
                    
                    # Vérifier que package.json existe
                    echo "Contenu du workspace:"
                    ls -la $WORKSPACE
                    
                    # Copier les fichiers dans un répertoire temporaire et lancer npm install
                    TEMP_DIR=$(mktemp -d)
                    cp -r $WORKSPACE/. $TEMP_DIR/
                    
                    echo "Vérification du contenu du répertoire temporaire:"
                    ls -la $TEMP_DIR
                    
                    echo "Installation des dépendances dans le répertoire temporaire:"
                    docker run --rm -v $TEMP_DIR:/app -w /app node:18-alpine sh -c "npm install"
                    
                    # Copier node_modules et package-lock.json vers le workspace
                    cp -r $TEMP_DIR/node_modules $WORKSPACE/ || true
                    cp $TEMP_DIR/package-lock.json $WORKSPACE/ || true
                    
                    # Nettoyer
                    rm -rf $TEMP_DIR
                    
                    echo "Installation terminée"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Exécution des tests...'
                sh '''
                    cd $WORKSPACE
                    docker run --rm -v $WORKSPACE:/app -w /app node:18-alpine sh -c "npm test"
                '''
            }
        }
        
        stage('Code Quality Check') {
            steps {
                echo 'Vérification de la qualité du code...'
                sh '''
                    cd $WORKSPACE
                    echo "Vérification de la syntaxe JavaScript..."
                    docker run --rm -v $WORKSPACE:/app -w /app node:18-alpine sh -c "find src -name '*.js' -exec node -c {} \\;"
                    echo "Vérification terminée"
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
                    echo "Vérification des dépendances..."
                    docker run --rm -v $WORKSPACE:/app -w /app node:18-alpine sh -c "npm audit --audit-level=high || true"
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
