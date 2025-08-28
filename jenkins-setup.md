# Configuration Jenkins pour mon-app-js

## Étapes de configuration dans Jenkins

### 1. Créer un nouveau job
1. Aller dans Jenkins Dashboard
2. Cliquer sur "New Item"
3. Nom du job : `mon-app-js-pipeline`
4. Type : `Pipeline`
5. Cliquer "OK"

### 2. Configuration du Pipeline
Dans la section **Pipeline** :

**Definition** : `Pipeline script from SCM`

**SCM** : `Git`

**Repository URL** : `https://github.com/VirgileCaujolle/mon-app-js.git`

**Credentials** : (laisser vide si repository public)

**Branch Specifier** : `*/main`

**Script Path** : `Jenkinsfile`

### 3. Configuration des triggers (optionnel)
Dans **Build Triggers** :
- ☑ `GitHub hook trigger for GITScm polling`
- ☑ `Poll SCM` : `H/5 * * * *` (toutes les 5 minutes)

### 4. Permissions Docker
Assurez-vous que Jenkins a accès à Docker :

```bash
# Dans le conteneur Jenkins ou sur le serveur Jenkins
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 5. Plugins requis
Vérifiez que ces plugins sont installés :
- Git plugin
- Pipeline plugin
- Docker Pipeline plugin

### 6. Variables d'environnement (optionnel)
Dans **Environment Variables** :
- `DOCKER_HOST` : si Docker n'est pas local
- `NODE_ENV` : `production`

## Test de la configuration

1. Cliquer sur "Build Now"
2. Vérifier les logs dans "Console Output"
3. L'application devrait être accessible sur `http://localhost:3000`

## Résolution de problèmes

### Erreur Git
- Vérifier que l'URL du repository est correcte
- Vérifier les permissions d'accès au repository

### Erreur Docker
- Vérifier que Docker est installé et accessible
- Vérifier les permissions Docker pour l'utilisateur Jenkins

### Erreur de port
- Vérifier qu'aucun autre service n'utilise le port 3000
- Modifier le port dans le Jenkinsfile si nécessaire
