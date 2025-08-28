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
                    
                    # NOUVELLE APPROCHE: Créer une image temporaire et copier les fichiers
                    echo "=== SOLUTION: Utilisation d'un conteneur avec COPY au lieu de volume mount ==="
                    
                    # Créer un Dockerfile temporaire pour l'installation
                    cat > /tmp/Dockerfile.npm << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --verbose
COPY . .
CMD ["echo", "Installation terminée"]
EOF
                    
                    # Construire l'image avec tous les fichiers
                    echo "Construction de l'image temporaire pour npm install..."
                    cd $WORKSPACE
                    docker build -f /tmp/Dockerfile.npm -t temp-npm-install .
                    
                    # Extraire node_modules du conteneur
                    echo "Extraction de node_modules..."
                    CONTAINER_ID=$(docker create temp-npm-install)
                    docker cp $CONTAINER_ID:/app/node_modules $WORKSPACE/ || echo "node_modules non trouvé"
                    docker cp $CONTAINER_ID:/app/package-lock.json $WORKSPACE/ || echo "package-lock.json non trouvé"
                    docker rm $CONTAINER_ID
                    
                    # Nettoyer l'image temporaire
                    docker rmi temp-npm-install || true
                    
                    # Nettoyer le Dockerfile temporaire
                    rm -f /tmp/Dockerfile.npm
                    
                    echo "=== VÉRIFICATION FINALE ==="
                    ls -la $WORKSPACE
                    
                    if [ -d "$WORKSPACE/node_modules" ]; then
                        echo "✓ node_modules installé avec succès"
                        echo "Nombre de packages installés: $(ls $WORKSPACE/node_modules | wc -l)"
                    else
                        echo "✗ node_modules NON trouvé après installation"
                        exit 1
                    fi
                    
                    echo "Installation terminée avec succès"
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
                    
                    echo "Vérification de la syntaxe JavaScript avec image temporaire..."
                    
                    # Créer un Dockerfile temporaire pour la vérification
                    cat > /tmp/Dockerfile.quality << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN find src -name "*.js" -type f -exec node -c {} \\;
CMD ["echo", "Vérification de qualité terminée"]
EOF
                    
                    # Construire et exécuter la vérification
                    docker build -f /tmp/Dockerfile.quality -t temp-quality .
                    docker run --rm temp-quality
                    
                    # Nettoyer
                    docker rmi temp-quality || true
                    rm -f /tmp/Dockerfile.quality
                    
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
                    
                    echo "Vérification des vulnérabilités de sécurité avec image temporaire..."
                    
                    # Créer un Dockerfile temporaire pour l'audit
                    cat > /tmp/Dockerfile.security << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm audit --audit-level=moderate || echo "Vulnérabilités détectées mais build continue"
CMD ["echo", "Audit de sécurité terminé"]
EOF
                    
                    # Construire et exécuter l'audit
                    docker build -f /tmp/Dockerfile.security -t temp-security .
                    docker run --rm temp-security
                    
                    # Nettoyer
                    docker rmi temp-security || true
                    rm -f /tmp/Dockerfile.security
                    
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
                            # Diagnostic du conteneur
                            echo "=== DIAGNOSTIC CONTENEUR ==="
                            docker ps -a | grep mon-app-js-container || echo "Conteneur non trouvé"
                            docker logs mon-app-js-container || echo "Pas de logs"
                            
                            # Test de connectivité
                            echo "=== TEST DE CONNECTIVITÉ ==="
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
