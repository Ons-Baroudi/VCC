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
