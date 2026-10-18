# TP2 : Le Cloud Infrastructure (IaaS)
Binôme : Ons Baroudi, Nawfal ElHammadi 


- Objectif : créer et administrer des machines virtuelles dans le Cloud OpenStack de l'INSA, comprendre le rôle des réseaux virtuels, des routeurs, des IP flottantes et des groupes de sécurité, faire évoluer une VM (redimensionnement, snapshot), puis piloter OpenStack en ligne de commande.

- OpensTACK = Cloud IaaS

- Au TP1, nous étions à la fois fournisseur et client : VirtualBox tournait sur notre PC et nous réglions nous-mêmes le matériel virtuel et le réseau. Ici, l'INSA fournit l'infrastructure et nous commandons des ressources à la demande, via l'interface Horizon puis via la CLI.

## Étape 1 : Construire un réseau virtuel

Notre projet (ens-insa-tp1-15) dispose de quotas : 6 instances, 6 vCPU, 6 Go de RAM et 2 IP flottantes, déjà allouées au projet (194.199.106.216 et 194.199.106.229). 
Ces quotas garantissent un partage équitable de l'infrastructure mutualisée.
Nous avons créé le réseau privé net-discovery avec le sous-réseau subnet-discovery (192.168.10.0/24, passerelle 192.168.10.1, DHCP activé). 

Juste après sa création, ce réseau était isolé : une VM posée dessus n'aurait pu parler qu'aux autres VM du même réseau.
Nous l'avons ensuite raccordé au routeur routerdefault en lui ajoutant une interface 192.168.10.1. Ce routeur relie désormais deux réseaux privés du projet (default, 192.168.34.0/24, et net-discovery) au réseau externe vpn-external.

La topologie montre aussi trois réseaux externes gérés par l'INSA : vpn-external (192.168.31.0/24 et 194.199.106.0/24, d'où viennent nos IP flottantes), public et storage-network. Nous ne pouvons pas y brancher nos VM : l'accès au monde extérieur reste du ressort du fournisseur, c'est la frontière du modèle IaaS.

Enfin, comme OpenStack bloque par défaut tout trafic entrant vers une VM, nous avons créé le groupe de sécurité sec-grp-tp2 avec deux règles entrantes depuis 0.0.0.0/0 : SSH (TCP 22) et ICMP (ping). Le groupe default, lui, n'autorise en entrée que les VM appartenant au même groupe.

### Checkpoint

**1. Quel est le rôle d'un réseau privé dans une infrastructure Cloud ? D'un routeur externe ?**

- Le réseau privé isole les ressources d'un projet de celles des autres utilisateurs du Cloud. Grâce au multi-tenancy, plusieurs projets partagent la même infrastructure physique sans se voir ni interférer au niveau réseau.

Il permet aussi aux VM d'un même projet de communiquer entre elles. Ses adresses sont privées (ici 192.168.10.0/24) : elles ne sont pas joignables depuis Internet et peuvent être réutilisées par d'autres projets sans conflit.

- Le routeur externe relie ce réseau privé au réseau externe du fournisseur (vpn-external).

Il sert de passerelle aux VM (192.168.10.1) et traduit leurs adresses privées (NAT) pour qu'elles accèdent à Internet, par exemple pour installer des paquets. C'est aussi lui qui traduit les IP flottantes vers les adresses privées pour rendre une VM joignable depuis l'extérieur. Notre topologie montre qu'un seul routeur relie plusieurs réseaux privés (`default` et `net-discovery`) à une même sortie.

**2. Pourquoi un fournisseur Cloud propose-t-il autant de catégories de ressources différentes ?**

Dans un Cloud IaaS, le client construit lui-même son infrastructure, brique par brique.
Le fournisseur propose donc séparément chaque type de ressource : calcul (instances, vCPU, RAM), stockage (volumes, instantanés), réseau (réseaux, routeurs, IP flottantes, groupes de sécurité) et services complémentaires (DNS, orchestration, identité).

Ce découpage permet de ne prendre que ce dont on a besoin, car chaque service ne nécessite pas les mêmes ressources : un service de calcul consomme surtout du CPU, une base de données du disque et de la mémoire. Chaque ressource peut aussi être ajustée indépendamment, ce qui correspond à l'élasticité du Cloud. Enfin, chaque catégorie est mesurée et limitée séparément : notre projet a des quotas distincts et la Vue d'ensemble comptabilise les VCPU-heures, RAM-heures et Go-heures, ce qui permet un partage équitable et une facturation à l'usage. Ce découpage reflète l'architecture d'OpenStack, faite de services indépendants (Nova, Neutron, Cinder, Glance, Keystone) qui communiquent par des API.

**3. Chemin d'un paquet envoyé depuis l'extérieur vers une VM sur net-discovery**

Une VM de net-discovery n'a qu'une adresse privée, inconnue depuis l'extérieur : sans IP flottante, aucun paquet venant de l'extérieur ne peut l'atteindre. Une fois une IP flottante associée (194.199.106.216), le chemin observé sur la topologie est :

1. le paquet, destiné à l'IP flottante, arrive sur le réseau externe vpn-external ;
2. il atteint la patte externe de routerdefault, qui traduit l'IP flottante en adresse privée de la VM ;
3. le routeur le transmet sur net-discovery par son interface 192.168.10.1 ;
4. le groupe de sécurité de la VM le filtre : il ne passe que si une règle l'autorise (TCP 22 pour SSH, ICMP pour ping) ;
5. la VM le reçoit et le service concerné le traite.

Le routeur ne fait que traduire les adresses et transférer les paquets : la connexion SSH est établie directement entre notre machine et le serveur SSH de la VM. Le réseau default n'est pas traversé.

## Étape 2 : Création d'une VM

Nous avons créé vm-discovery (image ubuntu-24.04, gabarit insa-tiny-1.1.10, réseau net-discovery, groupe sec-grp-tp2, paire de clés vm-discovery-key). 
Le DHCP lui a attribué l'adresse privée 192.168.10.181. Nous lui avons associé l'IP flottante 194.199.106.216, puis nous nous sommes connectés depuis notre PC avec `ssh -i vm-discovery-key.pem ubuntu@194.199.106.216`.

Le nom du gabarit se décode ainsi : 1 vCPU, 1 Go de RAM, 10 Go de disque. 

### Checkpoint

**1. Quelles sont les étapes nécessaires pour se connecter en SSH à une nouvelle VM OpenStack depuis la machine hôte ?**

1. Créer un réseau privé et son sous-réseau (`net-discovery`, 192.168.10.0/24, passerelle 192.168.10.1, DHCP activé), pour que la VM obtienne une adresse.
2. Raccorder ce sous-réseau au routeur `routerdefault` (interface 192.168.10.1), qui le relie au réseau externe `vpn-external`.
3. Créer un groupe de sécurité (`sec-grp-tp2`) autorisant SSH (TCP 22) et ICMP en entrée, car tout trafic entrant est bloqué par défaut.
4. Créer une paire de clés (`vm-discovery-key`) : OpenStack génère les deux clés, nous téléchargeons et protégeons la clé privée.
5. Créer la VM sur ce réseau, avec ce groupe et cette paire de clés. Au démarrage, cloud-init dépose la clé publique dans `~/.ssh/authorized_keys`.
6. Associer une IP flottante (194.199.106.216) au port de la VM (192.168.10.181).
7. Se connecter : `ssh -i vm-discovery-key.pem ubuntu@194.199.106.216`.

**2. Rôle de l'adresse IP privée, de l'IP flottante et des groupes de sécurité dans cette connexion**

- L'adresse privée (192.168.10.181) est la véritable adresse de la VM sur `net-discovery`, attribuée par DHCP. C'est la seule configurée sur sa carte réseau, celle qui permet aux VM du projet de communiquer, et celle vers laquelle la connexion SSH est finalement acheminée.
- L'IP flottante (194.199.106.216) est une adresse publique appartenant au projet, prise dans le pool de `vpn-external`. Le routeur traduit chaque paquet qui lui est destiné vers 192.168.10.181, et inversement. La VM ne la connaît pas : `ip a` ne l'affiche pas, alors que `curl ifconfig.me` exécuté depuis la VM la renvoie.
- Les groupes de sécurité sont un pare-feu appliqué à la carte réseau de la VM. Tout trafic entrant est bloqué sauf ce que les règles autorisent : notre règle TCP 22 laisse passer SSH. Ils sont à état : les réponses aux connexions autorisées repassent automatiquement.

En résumé, l'IP privée identifie la VM dans son réseau, l'IP flottante la rend adressable depuis l'extérieur, et le groupe de sécurité décide quelles connexions peuvent l'atteindre.

**3. Pourquoi distinguer IP privée et IP flottante plutôt que donner une adresse publique à chaque VM ?**

D'abord, parce que les adresses publiques sont rares : notre projet n'en a que 2 pour un quota de 6 instances, et à l'échelle du fournisseur les adresses IPv4 publiques sont limitées et coûteuses. Les adresses privées, elles, peuvent être réutilisées par chaque projet sans conflit.

Ensuite, pour la sécurité : toutes les VM n'ont pas besoin d'être exposées. Une base de données doit rester sur le réseau privé ; seule la VM qui porte le service utile doit être accessible.

Enfin, l'IP flottante appartient au projet et non à la VM : on peut la dissocier d'une VM et l'associer à une autre, par exemple pour remplacer une VM en panne sans changer l'adresse que connaissent les utilisateurs. La VM n'a rien à configurer, c'est le routeur qui fait toute la traduction.

## Étape 3 : Publier une application sur Internet

Nous avons ajouté à `sec-grp-tp2` une règle entrante TCP sur le port 5000. Sur la VM, nous avons cloné le dépôt `insa-labs`, installé `python3.12-venv`, créé un environnement virtuel, installé Flask et lancé `python3 app.py`. Flask écoute sur `0.0.0.0:5000` et affiche les adresses 127.0.0.1 et 192.168.10.181, mais pas l'IP flottante.

`curl http://localhost:5000` dans la VM et `curl.exe http://194.199.106.216:5000` depuis notre PC renvoient la même réponse : `{"application":"INSA VCC","status":"running"}`. Sous Windows PowerShell, `curl` est un alias d'`Invoke-WebRequest`, d'où l'utilisation de `curl.exe`.

Ce déploiement manuel reproduit à la main ce que faisait le Dockerfile du TP1 : choisir un environnement, récupérer le code, installer les dépendances, lancer l'application. Il fonctionne, mais il faudrait tout refaire pour chaque nouvelle VM.
### Checkpoint

**1. Pourquoi les Security Groups sont-ils indispensables même lorsqu'une VM possède une adresse IP publique ?**

L'IP publique rend seulement la VM joignable : elle donne une adresse, mais ne décide pas qui peut entrer. Les groupes de sécurité définissent les règles de trafic autorisé ; ils fonctionnent comme un pare-feu placé sur la carte réseau de la VM, et OpenStack bloque par défaut tout trafic entrant. Sans eux, tous les services de la VM seraient exposés à Internet, y compris ceux qui ne devraient pas l'être, comme une base de données.

Dans notre cas, la VM avait déjà son IP flottante, mais hello-api n'est devenue accessible depuis notre PC qu'après l'ajout de la règle TCP 5000. Seuls SSH, ICMP et le port 5000 sont ouverts : on n'ouvre que ce qui est nécessaire. Un même groupe s'applique à toutes les VM qui le portent, et une modification est prise en compte immédiatement, sans redémarrage.

**2. Chemin d'une requête HTTP depuis notre ordinateur jusqu'à hello-api**

Depuis notre PC, `curl.exe http://194.199.106.216:5000` envoie un paquet destiné à l'IP flottante, port 5000.

1. Le paquet arrive sur le réseau externe `vpn-external`.
2. `routerdefault` remplace l'IP flottante par l'adresse privée 192.168.10.181. Seule l'adresse change, le port 5000 reste le même.
3. Le routeur transmet le paquet sur `net-discovery` par son interface 192.168.10.1.
4. `sec-grp-tp2` l'accepte grâce à la règle TCP 5000 entrante.
5. Le paquet arrive sur la carte réseau de la VM. Comme Flask écoute sur `0.0.0.0:5000`, il reçoit la requête et renvoie la réponse JSON.

La réponse fait le chemin inverse : le groupe de sécurité la laisse sortir (il se souvient de la connexion) et le routeur remet l'IP flottante comme adresse source. L'adresse source de la requête, elle, n'est pas modifiée à l'aller : la VM voit la vraie adresse de notre PC.
## Étape 4 : Gérer les ressources du Cloud

**Redimensionnement.** Nous avons passé `vm-discovery` du gabarit `insa-tiny-1.1.10` à `insa-small-2.1.10` (2 vCPU, 1 Go, 10 Go). Après confirmation, `nproc` affiche 2. La RAM et le disque sont inchangés, et tout a été conservé : fichiers (2,2 Go utilisés), adresse privée 192.168.10.181, même adresse MAC donc même port réseau, IP flottante et groupe de sécurité. Le bail DHCP renouvelé et l'arrêt de hello-api montrent que la VM a redémarré.

**Snapshot.** Nous avons créé l'instantané `ubuntu-tp2`. Dans OpenStack, il devient une image du catalogue Glance, de type "Instantané" et privée, à côté des 16 images publiques fournies par l'INSA. Contrairement à VirtualBox, où un snapshot est un point de restauration d'une seule VM, il sert ici de modèle pour créer de nouvelles VM.

**VM à partir du snapshot.** Nous avons créé `vm-snapshot` (gabarit `insa-small-2.1.10`, `net-discovery`). Elle a son propre nom, sa propre adresse (192.168.10.174) et de nouvelles clés d'hôte SSH, mais son disque contient tout ce que nous avions ajouté : `ls -a ~/workspace/insa-labs/hello-api` montre `app.py` et `.venv`, et hello-api démarre directement, sans `git clone` ni `pip install`.

Pour nous y connecter, nous lui avons associé temporairement l'IP flottante 194.199.106.229. La première tentative a échoué (timeout) : la VM n'avait que le groupe `default`, qui n'autorise aucune entrée depuis l'extérieur. L'ajout de `sec-grp-tp2` a résolu le problème sans redémarrage. Nous avons défini un mot de passe pour l'utilisateur `ubuntu` (`sudo passwd ubuntu`) afin d'accéder à la console, car les images Cloud n'en ont pas par défaut.
### Checkpoint

**1. Différences entre une image officielle et un snapshot**

Une image officielle (`ubuntu-24.04`) est fournie par l'INSA dans Glance : système vierge, publique, maintenue par le fournisseur et commune à tous les projets.

Un snapshot (`ubuntu-tp2`) est une copie du disque d'une de nos VM à un instant donné. Il devient lui aussi une image Glance, mais de type "Instantané" et privée à notre projet. Il contient tout ce que nous avions installé : sur `vm-snapshot`, le dépôt et le venv étaient déjà là. En revanche, il ne capture que le disque : ni la RAM (hello-api n'était pas lancée), ni le gabarit, ni l'adresse IP, ni l'IP flottante. Et alors qu'une image officielle est documentée, un snapshot fige une machine configurée à la main, sans garder la trace des commandes utilisées.

**2. Avantages du redimensionnement dans un environnement Cloud**

Il permet d'adapter les ressources d'une VM à ses besoins sans la réinstaller : c'est l'élasticité du Cloud. Nous sommes passés de 1 à 2 vCPU en quelques clics, en conservant le disque, les fichiers, l'adresse privée, le port réseau, l'IP flottante et les règles de sécurité. C'est aussi un avantage économique : on ne réserve et on ne paie que ce dont on a besoin, et on augmente quand la charge augmente.


**3. Quand créer une VM à partir d'un snapshot plutôt que d'une image vierge ?**

- Pour dupliquer un environnement déjà configuré, comme `vm-snapshot`, prête en quelques minutes avec hello-api.
- Pour absorber une montée en charge en lançant plusieurs VM identiques.
- Avant une opération risquée, pour pouvoir revenir à un état fonctionnel.
- Après une panne, pour reconstruire rapidement une machine et lui réassocier la même IP flottante.
- Pour fournir le même environnement à une équipe ou créer une copie de test d'une machine de production.

## Pour aller plus loin : la ligne de commande OpenStack

Horizon et la CLI sont deux clients des mêmes API (Keystone, Nova, Neutron, Glance). La CLI permet d'écrire, de rejouer et d'automatiser les opérations : c'est la première marche vers l'Infrastructure as Code.

**Adaptation à Windows.** Notre machine hôte étant sous Windows(parce qu'on a pas fait le tp à l'insa), nous avons adapté les commandes Linux du TP :
- `sudo apt install python3-openstackclient`  à  `python -m pip install python-openstackclient`
  
- `source <filename>-openrc.sh`  à  `$env:OS_CLOUD = "openstack"` avec le fichier `clouds.yaml`, qui contient les mêmes informations 

**Authentification.** Nous avons créé un identifiant d'application (`cli-tp2`, expirant le 31/10/2026). Il permet à la CLI de s'authentifier auprès de Keystone sans utiliser notre compte INSA, avec des droits limités au projet et une durée limitée. `openstack token issue` a renvoyé un jeton valable 24 heures, rattaché à notre projet.

**Vérifications.** `openstack keypair list` montre la clé publique `vm-discovery-key` : OpenStack ne conserve jamais la clé privée, qui reste sur notre PC. `openstack security group rule list sec-grp-tp2` affiche nos 5 règles (2 sorties automatiques, SSH, ICMP et TCP 5000), qui s'appliquent à toutes les VM portant ce groupe.

**Création de VM en CLI.** Une seule commande a remplacé les onglets de Horizon :

```
openstack server create --flavor insa-tiny-1.1.10 --image ubuntu-24.04 --network net-discovery --key-name vm-discovery-key --security-group sec-grp-tp2 vm-cli
```

- `vm-cli` (192.168.10.19) est apparue Active dans `openstack server list` et dans Horizon. `openstack server show` masque certains champs, comme le serveur physique qui l'héberge : le client n'a pas à le connaître.

- Quand on a dépalcé l'ip flottante à vm-cli :  `openstack server add floating ip vm-cli 194.199.106.229` a directement retiré l'IP de `vm-snapshot` pour l'associer à `vm-cli`
- 
- Or, une IP flottante ne pointe que vers un seul port à la fois. À la connexion, SSH a affiché `REMOTE HOST IDENTIFICATION HAS CHANGED`, car notre fichier `known_hosts` associait cette adresse à la clé d'hôte de `vm-snapshot`. Après avoir vérifié que ce changement venait de nous, nous avons supprimé l'ancienne empreinte avec `ssh-keygen -R 194.199.106.229` et nous sommes connectés à `vm-cli`.
