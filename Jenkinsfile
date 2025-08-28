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
                deleteDir()
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/VirgileCaujolle/mon-app-js.git']]
                ])
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installation des dépendances Node.js...'
                script {
                    sh '''
                        echo "Vérification de la présence du package.json..."
                        ls -la package.json
                        echo "Installation des dépendances..."
                        docker run --rm -v $(pwd):/app -w /app node:18-alpine sh -c "npm ci"
                    '''
                }
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
                script {
                    sh '''
                        echo "Arrêt du conteneur existant s'il existe..."
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                        
                        echo "Démarrage du nouveau conteneur..."
                        docker run -d --name ${CONTAINER_NAME} -p 3000:3000 ${IMAGE_NAME}
                        
                        echo "Attente du démarrage de l'application..."
                        sleep 15
                        
                        echo "Vérification que le conteneur est en cours d'exécution..."
                        docker ps | grep ${CONTAINER_NAME}
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                echo 'Vérification de santé de l\'application...'
                script {
                    try {
                        sh '''
                            echo "Test de connectivité sur le health endpoint..."
                            for i in {1..5}; do
                                if curl -f http://localhost:3000/health; then
                                    echo "Application déployée avec succès et répond correctement"
                                    exit 0
                                else
                                    echo "Tentative $i/5 échouée, nouvelle tentative dans 5 secondes..."
                                    sleep 5
                                fi
                            done
                            echo "Health check échoué après 5 tentatives"
                            exit 1
                        '''
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo "Warning: Health check failed: ${e.getMessage()}"
                        sh '''
                            echo "Logs du conteneur pour diagnostic:"
                            docker logs ${CONTAINER_NAME} || true
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Nettoyage des ressources temporaires...'
            sh '''
                # Nettoyer les images Docker non utilisées
                docker image prune -f || true
            '''
        }
        success {
            echo 'Pipeline exécuté avec succès!'
            echo "Application accessible sur: http://localhost:3000"
        }
        failure {
            echo 'Le pipeline a échoué!'
            sh '''
                # Arrêter le conteneur en cas d'échec
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
            '''
        }
    }
}
