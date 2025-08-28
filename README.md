# Mon App JS - TP Jenkins

Application JavaScript simple pour démontrer un pipeline CI/CD avec Jenkins et Docker.

## Fonctionnalités

- Calculatrice JavaScript simple
- Interface web responsive
- Tests automatisés avec Jest
- Déploiement avec Docker
- Pipeline CI/CD avec Jenkins

## Structure du projet

```
mon-app-js/
├── src/
│   ├── index.html      # Interface utilisateur
│   ├── app.js          # Logique principale
│   ├── utils.js        # Fonctions utilitaires
│   └── styles.css      # Styles CSS
├── tests/
│   └── app.test.js     # Tests unitaires
├── package.json        # Configuration npm
├── server.js           # Serveur Express
├── Dockerfile          # Configuration Docker
├── docker-compose.yml  # Orchestration Docker
├── Jenkinsfile         # Pipeline CI/CD
└── README.md
```

## Utilisation locale

### Avec Node.js
```bash
npm install
npm start
```

### Avec Docker
```bash
docker build -t mon-app-js .
docker run -p 3000:3000 mon-app-js
```

### Avec Docker Compose
```bash
docker-compose up -d
```

L'application sera accessible sur http://localhost:3000

## Tests

```bash
npm test
```

## Pipeline Jenkins

Le pipeline Jenkins automatise :
1. Récupération du code
2. Installation des dépendances
3. Exécution des tests
4. Vérification de la qualité du code
5. Construction de l'image Docker
6. Analyse de sécurité
7. Déploiement
8. Vérification de santé

## Health Check

L'application expose un endpoint de santé : http://localhost:3000/health
