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

## Étape 2

**1. Quelles opérations réalisées manuellement dans le TP1 ont été automatisées par Jenkins ?**

Dans le TP1, pour mettre la calculatrice en ligne, on a fait à la main :
1. le clonage du code (`git clone`) ;
2. la construction des images des 5 microservices (`docker build` dans chaque dossier : calc, sum, sub, mul, div) ;
3. le tag et l'envoi des images vers le Registry (`docker tag` + `docker push`) ;
4. le déploiement sur le cluster (`kubectl apply` des manifestes).

Avec Jenkins, le pipeline décrit dans le Jenkinsfile fait automatiquement :
- **le clonage du dépôt GitHub** : stage `Declarative: Checkout SCM`, ajouté automatiquement par Jenkins en mode « Pipeline script from SCM » ;
- **la construction des 5 images Docker** : stages `Build calc-service` à `Build div-service`, qui exécutent `docker build -t calculator-<service>:latest .` dans chaque dossier.

**Le lancement est aussi automatisé** : grâce à la scrutation du dépôt (`H/5 * * * *`), Jenkins vérifie toutes les 5 minutes s'il y a un nouveau commit sur `main` et lance le build tout seul. Le build #2 a démarré à la suite de notre commit sur le README (`2ec0cb4`), sans aucun clic.

L'installation et la configuration de l'infrastructure (cluster Kubernetes, Registry, containerd) restent manuelles : Jenkins ne fait que s'en servir.
**2. Votre pipeline comporte plusieurs étapes. Que se passe-t-il si la construction de l'image calculator-sub échoue ?**

Si la commande `docker build` du stage `Build sub-service` échoue, elle renvoie un code de sortie différent de 0 et Jenkins considère l'étape comme échouée. Le pipeline s'arrête immédiatement :
- les stages suivants (`Build mul-service` et `Build div-service`) ne sont pas exécutés = echec de pipeline.
- 
**3. Le Webhook GitHub est la méthode la plus utilisée en entreprise dans une chaîne CI. Quelles opérations devriez-vous faire pour la mettre en place dans votre architecture ?**

Avec le polling actuel, c'est Jenkins qui interroge GitHub toutes les 5 minutes pour savoir s'il y a un nouveau commit. Avec un webhook, c'est l'inverse : dès qu'un push est fait sur le dépôt, GitHub envoie une requête HTTP à Jenkins pour lui dire de lancer le pipeline. Le build démarre immédiatement, sans délai et sans requêtes inutiles.

Il faut donc que GitHub puisse joindre Jenkins depuis Internet. Or, dans notre architecture, VM-Jenkins n'a qu'une adresse privée (`192.168.20.73`) et n'est accessible que via un rebond SSH par VM-K8s. Les opérations à faire sont :

1. **Rendre Jenkins joignable** : associer une IP flottante à VM-Jenkins dans OpenStack et ouvrir le port 8080 dans son groupe de sécurité.
2. **Configurer le webhook sur GitHub** : dans le dépôt Calculator-CICD

## Étape 3

**1. Pourquoi Jenkins doit-il disposer d'un accès à l'API Kubernetes ?**

Jenkins est un orchestrateur de la chaîne CI/CD : il ne remplace ni Docker ni Kubernetes, il les pilote. Pour automatiser la dernière étape, le déploiement des microservices de la calculatrice, il doit pouvoir exécuter `kubectl apply` sur les manifestes du dossier `kubernetes/`.

Or `kubectl` ne fait qu'envoyer des requêtes à l'API server, qui est le seul point d'entrée du cluster : c'est lui qui enregistre l'état désiré (dans etcd), puis le scheduler et les controllers créent ou mettent à jour les pods. Sans accès à l'API, le déploiement resterait manuel (`kubectl apply` tapé sur VM-K8s, comme au TP1).

**2. Quelles étapes du job Jenkins correspondent à la CI, et lesquelles correspondent à la CD ?**

**CI (intégration continue)** : à chaque commit, le code est récupéré et construit automatiquement, pour vérifier qu'il s'intègre sans casser le build.
- le clonage du dépôt GitHub : stage `Declarative: Checkout SCM`, ajouté automatiquement par Jenkins en mode « Pipeline script from SCM » (ainsi que le stage `Checkout`, qui n'est qu'un `echo`) ;
- la construction des 5 images Docker : stages `Build calc-service` à `Build div-service`, qui exécutent `docker build -t 192.168.20.134:5000/calculator-<service>:2.0 .` dans chaque dossier.

**CD**, en deux niveaux :
- **Continuous Delivery (livraison continue)** : stage `Push images`. Les images versionnées sont publiées dans le Registry (`docker push`) : la nouvelle version est livrée et prête à être déployée sur le cluster.
- **Continuous Deployment (déploiement continu)** : stage `Deploy to Kubernetes`. Les manifestes du dossier `kubernetes/` sont appliqués sur le cluster (`kubectl apply`) : la nouvelle version est mise en service automatiquement, sans intervention humaine.


**3. Quel est le parcours d'une modification du code source depuis le "git push" jusqu'à l'application Calculator ?**

1. Le développeur pousse sa modification sur `main` (**Git**, **GitHub**).
2. Jenkins détecte le nouveau commit grâce au polling toutes les 5 minutes (**Jenkins**).
3. Jenkins lit le Jenkinsfile et clone le dépôt (**Jenkins**, **Git**).
4. Les 5 images sont construites (**Docker**, `docker build`).
5. Les images sont publiées dans le Registry `192.168.20.134:5000` (**Docker**, `docker push`, **Registry**).
6. Les manifestes sont appliqués sur le cluster (**kubectl**, **API Kubernetes**, avec le kubeconfig et les droits RBAC).
7. Kubernetes remplace les anciens pods par les nouveaux, par rolling update (**Kubernetes** : scheduler, controllers).
8. Les nœuds téléchargent l'image et lancent les conteneurs (**kubelet**, **containerd**).
9. Les Services relient les microservices entre eux et exposent calc (**Services**, **CoreDNS**, **Calico**).
10. L'utilisateur interroge la calculatrice sur le NodePort `32333` (`curl`).

Jenkins automatise les étapes 2 à 6, Kubernetes réalise les étapes 7 à 10.

**Limite :** la version `2.0` est fixe. Pour un nouveau commit, les manifestes ne changent pas, donc `kubectl apply` ne redéploie rien. Il faudrait une version unique par build (numéro de build ou hash du commit).
