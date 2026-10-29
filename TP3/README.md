# TP3 : Déployer une application distribuée

**Binôme : Ons BAROUDI et Nawfel ELHammadi**

## Étape 1

![Architecture](architecture-tp3.png)

## Étape 2

**1. Quels sont les différents rôles de la VM connectée à Internet dans votre architecture ?**

En gros, `vm-calc` joue trois rôles. D'abord, c'est la porte d'entrée de l'application : c'est la seule VM qui a une IP flottante, donc c'est elle qui reçoit les requêtes des utilisateurs sur le port 50000, que Docker renvoie vers le port 80 du conteneur `calculator`. Ensuite, c'est elle qui coordonne les calculs : `calculator` ne calcule rien lui-même, il découpe l'expression et envoie chaque opération au bon service sur le réseau privé, puis renvoie le résultat à l'utilisateur. Enfin, elle nous sert de point de passage pour l'administration : comme les 4 autres VM n'ont pas d'adresse publique, on passe d'abord par `vm-calc` pour s'y connecter en SSH.

**2. Quels avantages apporte ce déploiement par rapport à une exécution de tous les services sur une seule machine ?**

Le premier avantage, c'est que les pannes restent isolées : si une VM tombe, seul son service est touché, alors qu'au TP1 tout tombait en même temps parce que les 5 conteneurs étaient sur la même machine. Chaque service a aussi ses propres ressources, donc si un service est très sollicité, il ne ralentit pas les autres. On peut également agrandir, redémarrer ou mettre à jour une VM sans toucher au reste de l'application. Et c'est plus sûr, puisque seule `vm-calc` est exposée à Internet, les services de calcul restent cachés dans le réseau privé.

## Étape 3

**1. Quels flux réseau sont nécessaires au fonctionnement de votre application ?**

Il y a trois types de flux. D'abord, celui des utilisateurs : ils envoient leurs calculs en HTTP vers l'IP flottante de `vm-calc` sur le port 5000, que Docker renvoie vers le port 80 du conteneur `calculator`. Ensuite, les flux internes : `calculator` envoie chaque opération en HTTP POST au bon service, sur le réseau privé, vers les ports 50001 (addition), 50002 (soustraction), 50003 (multiplication) et 50004 (division). Enfin, pour nous, il y a le SSH sur le port 22, vers `vm-calc` puis vers les autres VM en passant par elle. Les VM ont aussi besoin de sortir sur Internet pour installer Docker et cloner le dépôt, ce qui passe par le routeur.

**2. Pourquoi est-il préférable d'ouvrir uniquement les ports nécessaires plutôt que d'autoriser tout le trafic ?**

Parce que chaque port ouvert est une porte d'entrée possible pour une attaque. Si on ouvre tout, n'importe qui peut essayer de joindre tous les services de nos VM, même ceux qui ne devraient jamais être exposés. En n'ouvrant que le nécessaire, on réduit fortement les risques. Par exemple, les ports 50001 à 50004 ne sont accessibles qu'entre nos VM : personne depuis Internet ne peut appeler directement le service d'addition. En plus, ça rend la configuration plus claire : en lisant les règles, on voit tout de suite quels flux l'application utilise.

**3. Pouviez-vous utiliser Docker Compose pour déployer les services sur toutes les VM en simultané ? Pourquoi ?**

Non, parce que Docker Compose ne fonctionne que sur une seule machine : il lance tous les conteneurs sur le même hôte, celui sur lequel on l'exécute. On l'avait d'ailleurs vu au TP1, le réseau créé par Compose était en `"Scope": "local"`. Ici, chaque service est sur une VM différente, donc il a fallu se connecter à chaque VM et lancer son conteneur à la main. C'est aussi pour ça qu'on a dû utiliser `--add-host` : le DNS de Compose, qui permettait aux services de se trouver par leur nom, n'existe que sur une seule machine. Pour déployer automatiquement sur plusieurs machines en même temps, il faudrait un orchestrateur comme Kubernetes ou Docker Swarm.
