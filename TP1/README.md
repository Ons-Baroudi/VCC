# TP1 : De la machine au conteneur

**Binôme : Ons Baroudi, Nawfal Elhammadi**

Objectif du TP : rendre une application portable en passant de la machine virtuelle au conteneur, et d'une application àseul conteneur à une application multi-conteneurs.

## ETAPE 1:  Découverte de la virtualisation

Nous avons importé une VM Ubuntu 26.04 dans VirtualBox (1 vCPU, 2048 Mo, disque de 25 Gio) et configuré une redirection de port (TCP 5555 de l'hôte vers 22 de la VM) pour y accéder en SSH. Dans la VM : `nproc` affiche 1 CPU, `top` montre 97 processus et environ 311 Mo de RAM utilisés sur 1643 Mo visibles, `df -h` indique 23 Go pour la ram , et l'IP est 10.0.2.15.

### Checkpoint

**1. Qu'est-ce qu'une VM ?**

Une machine virtuelle (VM) est une représentation logicielle d'une machine physique. L'hyperviseur, ici VirtualBox (type 2, déjà installé sur l'OS de l'ordinateur de l'INSA), crée des ressources virtuelles (vCPU, vRAM, vDisk, vNIC) à partir du matériel réel et les présente à l'OS invité (Ubuntu 26.04, qui tourne dans la VM). Nous l'avons vérifié : `nproc` n'affiche qu'un CPU alors que notre PC en a plusieurs, et la VM voit un disque de 23 Go qui n'est en réalité qu'un fichier .vdi sur notre disque.

**2. Deux avantages de la virtualisation en environnement professionnel**

- La mutualisation des ressources physiques est un vrai avantage car plusieurs VM peuvent partager un même serveur physique, au lieu d'un serveur par application souvent sous-utilisé (20 %, 40 % et 15 % de CPU regroupés en un seul serveur à 75 %). Ainsi, on a moins de machines à acheter, à maintenir et à alimenter.
- Un autre avantage est la facilité de gestion : une VM se crée à partir d'une image avec un OS préinstallé (l'OVA que nous avons importé). Cette même VM peut ensuite être clonée, redimensionnée selon nos besoins et sauvegardée par snapshot.
- Le snapshot, qui est une capture instantanée de l'état de la VM à l'instant t, permet de conserver une copie de la VM avant une mise à jour risquée. Si l'ancienne version est perdue lors de la mise à jour, on dispose toujours du duplicata, ce qui est très pratique.

**3. Différence entre travailler sur son ordinateur et dans une VM**

- La VM a des ressources limitées mais  un surcoût aussi. En effet,  elle n'a que ce qu'on lui alloue (1 CPU, 2048 Mo), mais  son OS consomme déjà 97 processus et 311 Mo de RAM sans aucune application.
- La VM est isolée grâce à son réseau privé, séparé de l'extérieur. En NAT, elle a une adresse privée (10.0.2.15) injoignable depuis notre PC : il a fallu une redirection de port pour s'y connecter en SSH (port 5555 de notre machine physique vers le port 22 de la VM).
- Etant donnée que la vm est séparée de la machine hôte, si elle tombe en panne, la machine hôte n'est pas affectée. En revanche, si la machine hôte tombe en panne, toutes ses VM s'arrêtent.
  
## Étape 2 : Conteneurs
Dans cet etape, nous avons installé Docker dans la vm depuis le dépôt officiel. 
`docker run hello-world` a téléchargé l'image depuis Docker Hub (digest sha256:5e2309...), puis a créé et exécuté un conteneur, qui apparaît ensuite en `Exited (0)` dans `docker container ls --all`. Ensuite, le conteneur 'cnt1' lancé depuis l'image ubuntu et il  vaut 45,6 MO.

**1. Qu'est-ce qu'un conteneur Docker ?**
Le conteneur Docker est une application avec ses dépendances et son environnement d'exécution, isolée et sans besoin virtualiser toute une machine. 
Il s'exécute sur le noyau de l'OS hôte (qui est notre vm), qui lui fournit ses propres processus, son propre système de fichiers et sa propre interface réseau.
Aussi, il a une interface (observé lors du tp) eth0 et son IP est 172.17.0.2. Et il est crée à partir d'une image et vit tant que son processus principal tourne; `hello-world` s'est arrêté seul dès la fin de son programme.

**2. Quelle différence fondamentale entre une machine virtuelle et un conteneur ?**
- Une VM  embarque son propre OS complet, avec son noyau, au-dessus d'un hyperviseur ( et rend son materiel physique virtuel).
- Or, les conteneurs partagent le kernel de l'os hote et n'embarquent que l'application et ses bibliothèques, au-dessus d'un container.
c'est pour ça , on a observé lors du tp que   l'image ubuntu fait 45,6 Mo (car elle ne contient pas de noyau), contre 25 Gio de disque pour la VM
- Aussi,puisqu'il s'exécute directement sur le kernel,  le conteneur démarre instantanément alors que la VM doit démarrer un OS de 97 processus (elle doit passer l'hypervisuer , et l'hyperviseur doit demander au kernel hôte, pour qu'elle puisse finalement démarrer).
  
**3. Pourquoi les conteneurs sont-ils adaptés au déploiement dans le Cloud ?**
Comme on l'a vu au tp, le conteneur est lèger (45 mo) car  n'embarque ni OS ni kernel, seulement l'application et ses bibliothèques. Il se télécharge et démarre donc en quelques secondes. et c'est grâce à cette légèreté, on peut multiplier les conteneurs d'une application quand la charge augmente et les supprimer quand elle baisse.

## Étape 3 Docker
### Checkpoint
**1. Pourquoi un Dockerfile est-il préférable à la configuration manuelle d'un conteneur ?**

On a Configuré à la main dans `cnt1` : `apt update`, puis installation de `net-tools` et `ping`. Ces paquets n'existent que dans ce conteneur. Pour 50 ou 100 conteneurs, il faudrait refaire toutes ces étapes sur chacun : c'est long, pénible et chaque répétition risque une erreur ou un oubli.

Mais avec  Dockerfile, cette configuration devient automatique et rapide car on décrit une seule fois les étapes; image de base, code, dépendances, port, commande ( et c'est un processus très long) , `docker build` produit une image, et chaque conteneur créé à partir d'elle est configuré à l'identique. Pour une mise à jour, on modifie une ligne du Dockerfile et on reconstruit, au lieu de reconfigurer chaque conteneur.

**2. Quelle différence existe entre une image Docker et un conteneur Docker ?**
l'image docker est exécuté dans le conteneur. L'image est le modèle qui englobe l'application, ses dépendances et sa configuration (ici `hello-api:1.0`, 198 Mo, construite à partir du Dockerfile).

Alors que le conteneur est l'instance qui exécute cette image, avec son propre processus, rzo. Le conteneur hello-api qu'on a crée tourne avec l'adresse ip 172.17.0.3. Lors de l'execution de l'image, le conteneur crée une interface passerelle , et l'adresse de cette interface ets la même du conteneur (donc dans notre cas 172.17.0.3).

 Une image donne autant de conteneurs qu'on veut, et modifier un conteneur ne modifie pas l'image(image= modèle immuable).

 ## 4) Déployer une application multi-conteneurs
 ### Checkpoint
 Le dossier `calculator` contient 5 microservices (calc, sum, sub, mul, div), donc 5 Dockerfiles. `docker compose build` a construit les 5 images, puis `docker compose up` a créé le réseau `calculator_default` et lancé les 5 conteneurs, chacun sur son port (addition 50001, soustraction 50002, multiplication 50003, division 50004, calculator 80).
 - Le port 80 n'est pas joignable depuis la VM
**1. Pourquoi Docker Compose est-il préférable au lancement manuel de plusieurs conteneurs ?**
   À la main, il aurait fallu enchaîner 5 `docker build`, créer un réseau, 5 `docker run` avec les bons noms et ports, puis tout arrêter un par un. C'est l'enchaînement à risque et long (on peut produire des erreurs,oubli d'une étape, erreur de configuration).
   Avec Compose, une commande construit les 5 images, une autre crée le réseau et lance l'application, une dernière supprime tout.
   
**2. Quel est le rôle du fichier docker-compose.yml ?**
Il décrit toute l'application dans un seul fichier : les 5 services de la calculatrice(sum,sub,mult,div et calculatrice), le Dockerfile de chacun et les ports publiés (seul `calculator` expose 50000:80, les autres restent sur le réseau interne)

**3. Dans quels cas Docker Compose pourrait-il atteindre ses limites ?**

ompose déploie tout sur un seul hôte : le réseau qu'il a créé a d'ailleurs une portée locale.Il n'existe que sur notre VM. Si cette VM tombe, les 5 services tombent. Aussi, Compose ne sait pas répartir les services sur plusieurs VM ou plusieurs Clouds. 
