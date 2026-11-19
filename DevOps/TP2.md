## Etape 1

### 1. Pourquoi Jenkins doit-il communiquer avec VM-K8s et VM-Registry ?

Jenkins construit lui-même les images Docker de la calculatrice, sur VM-Jenkins. Il doit ensuite communiquer avec deux machines, pour deux raisons différentes :

- **VM-Registry (192.168.20.134:5000)** : Jenkins y **publie** les images qu'il a construites (`docker push`). Sans cela, elles resteraient sur VM-Jenkins, et les nœuds du cluster ne pourraient pas les télécharger. Le script `configure_jenkins_registry.sh` déclare ce Registry comme « insecure » dans `/etc/docker/daemon.json`, car il fonctionne en HTTP.
- **VM-K8s, et plus précisément l'API server (https://192.168.20.215:6443)** : Jenkins lui envoie les manifestes pour **déployer** les nouvelles versions (`kubectl apply`). Il utilise une identité dédiée, le ServiceAccount `jenkins-deployer`, qui n'a que les droits nécessaires : gérer les Deployments et les Services du namespace `default`, mais pas voir les nœuds.

Jenkins n'envoie jamais d'image au cluster : ce sont les nœuds qui vont la chercher dans le Registry, d'après les manifestes.

### 2. Quel est le rôle de Jenkins dans la chaîne CI/CD ?

Jenkins est un serveur d'automatisation : il orchestre les étapes du pipeline CI/CD. À chaque nouveau commit, il enchaîne automatiquement :

1. la récupération du code depuis GitHub ;
2. la construction des images Docker ;
3. les tests ;
4. la publication des images dans le Registry ;
5. le déploiement sur Kubernetes.

Il remplace ainsi toutes les opérations faites à la main au TP1, et prévient l'équipe en cas d'échec. Il ne remplace ni Git (qui stocke le code), ni Docker (qui construit les images), ni le Registry (qui les stocke), ni Kubernetes (qui exécute l'application) : il appelle chacun de ces outils au bon moment.

### 2.1 Test d'un premier job Jenkins : les étapes réalisées

Après avoir lancé le build #1 du job `public-git-checkout`, nous avons cliqué sur le numéro du build, puis sur **Sortie de la console**. Le journal complet est le suivant :

Le job a réalisé les étapes suivantes.

**1. Déclenchement et préparation**
- `Started by user admin`
- `Running as SYSTEM`
- l'utilisateur Linux `jenkins`.
- `Building in workspace /var/lib/jenkins/workspace/public-git-checkout` : Jenkins utilise le **workspace** du job, un dossier dédié sur VM-Jenkins, dans lequel le code est cloné.

**2. Préparation de Git**
- `The recommended git tool is: NONE` 
- `No credentials specified` : aucun identifiant n'est utilisé, car le dépôt est public.

**3. Récupération du code depuis GitHub**
- `git init` : création d'un dépôt Git vide dans le workspace.
- `git fetch ... +refs/heads/*:refs/remotes/origin/*` : téléchargement de toutes les branches et de tous les tags du dépôt `Ons-Baroudi/Calculator-CICD`. La partie `+refs/heads/*:refs/remotes/origin/*` signifie : « copie chaque branche du dépôt distant sous le nom `origin/<branche>` ».
- `git config remote.origin.url ...` 
- `Avoid second fetch` : le code vient d'être téléchargé, Jenkins évite de le télécharger une seconde fois.

**4. Choix de la version exacte du code**
- `git rev-parse refs/remotes/origin/main^{commit}` : Jenkins cherche le dernier commit de la branche `main`, conformément au réglage `*/main` du job.
- `Checking out Revision 713bf62eb706dcf90be29b017e239195b2fbe75b` : il trouve le commit `713bf62...`.
- `git checkout -f 713bf62...` 
- `Commit message: "Update README with additional information"` : le message de ce commit est affiché, pour savoir quelle version du code a été traitée.

**5. Fin du build**
- `First time build. Skipping changelog.` : pour un premier build, il n'y a pas de build précédent auquel comparer, donc pas de liste de modifications.
- `Finished: SUCCESS` : toutes les étapes ont réussi.
