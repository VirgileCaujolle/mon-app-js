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
                    docker run --rm -v $(pwd):/app -w /app node:18-alpine sh -c "npm ci"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Exécution des tests...'
                sh '''
                    docker run --rm -v $(pwd):/app -w /app node:18-alpine sh -c "npm test"
                '''
            }
        }
        
        stage('Code Quality Check') {
            steps {
                echo 'Vérification de la qualité du code...'
                sh '''
                    echo "Vérification de la syntaxe JavaScript..."
                    docker run --rm -v $(pwd):/app -w /app node:18-alpine sh -c "find src -name '*.js' -exec node -c {} \\;"
                    echo "Vérification terminée"
                '''
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Construction de l\'image Docker...'
                sh '''
                    docker build -t ${IMAGE_NAME} .
                '''
            }
        }
        
        stage('Security Scan') {
            steps {
                echo 'Analyse de sécurité...'
                sh '''
                    echo "Vérification des dépendances..."
                    docker run --rm -v $(pwd):/app -w /app node:18-alpine sh -c "npm audit --audit-level=high || true"
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
