# Configuration Jenkins pour mon-app-js

## Étapes de configuration du pipeline Jenkins

### 1. Créer un nouveau job

1. Dans Jenkins, cliquez sur **"Nouveau Item"**
2. Nom : `mon-app-js-pipeline`
3. Type : **Pipeline**
4. Cliquez sur **OK**

### 2. Configuration du Pipeline

Dans l'onglet **Configuration** :

#### Section General
- ✅ **GitHub project** : `https://github.com/VirgileCaujolle/mon-app-js`

#### Section Build Triggers
- ✅ **GitHub hook trigger for GITScm polling**
- ✅ **Poll SCM** : `H/5 * * * *` (toutes les 5 minutes)

#### Section Pipeline
- **Definition** : `Pipeline script from SCM`
- **SCM** : `Git`
- **Repository URL** : `https://github.com/VirgileCaujolle/mon-app-js.git`
- **Credentials** : 
  - Si repository public : laisser vide
  - Si repository privé : ajouter credentials GitHub
- **Branches to build** : `*/main`
- **Script Path** : `Jenkinsfile`

### 3. Vérifications avant de lancer

Assurez-vous que :
- [ ] Jenkins peut accéder à Internet
- [ ] Docker est accessible depuis Jenkins
- [ ] Le repository GitHub est accessible
- [ ] Les plugins suivants sont installés :
  - Git plugin
  - Pipeline plugin
  - Docker Pipeline plugin
  - NodeJS plugin

### 4. Test de la configuration

1. Cliquez sur **"Construire maintenant"**
2. Observez les logs dans **"Console Output"**
3. Vérifiez que toutes les étapes se déroulent correctement

## Résolution des problèmes courants

### Erreur "not in a git directory"
```bash
# Dans le conteneur Jenkins, vérifier Git
docker exec -it jenkins-server bash
git --version
git config --list
```

### Problème d'accès Docker
```bash
# Vérifier l'accès Docker depuis Jenkins
docker exec -it jenkins-server docker ps
```

### Problème de permissions
```bash
# Vérifier les permissions du socket Docker
ls -la /var/run/docker.sock
```
