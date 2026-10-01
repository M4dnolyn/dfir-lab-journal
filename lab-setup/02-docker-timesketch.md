
02 — Déploiement de Timesketch via Docker
Objectif

Déployer Timesketch, la plateforme collaborative d'analyse et de visualisation de timelines forensiques, en utilisant Docker comme environnement d'exécution.

Timesketch repose sur une stack de cinq services conteneurisés :
Service 	Rôle
timesketch-web 	Application web principale
timesketch-worker 	Traitement asynchrone des imports
opensearch 	Moteur d'indexation et de recherche des événements
postgres 	Base de données relationnelle (utilisateurs, sketches)
redis 	File de messages entre web et worker
nginx 	Reverse proxy HTTP/HTTPS
Prérequis

    VM Ubuntu 24.04 LTS opérationnelle (voir 01-vm-provisioning.md)
    RAM VM : 8 Go minimum — OpenSearch seul alloue 2 Go au démarrage
    Disque VM : 80 Go — les images Docker de la stack pèsent ~6 Go à elles seules
    Accès internet depuis la VM pour le téléchargement des images

1. Pourquoi le dépôt officiel Docker et pas docker.io

Ubuntu propose dans ses dépôts standard le paquet docker.io, qui est une version communautaire de Docker maintenue par Canonical. Ce paquet présente deux limitations pour ce lab :

    Il ne fournit pas docker-compose-plugin (la commande docker compose intégrée au client Docker), uniquement l'ancien binaire standalone docker-compose — qui est lui-même introuvable dans Ubuntu 24.04.
    Il prend du retard sur les versions amont : Docker 29.x n'est pas disponible via docker.io.

Le dépôt officiel de Docker (download.docker.com) fournit docker-ce, docker-ce-cli, containerd.io et docker-compose-plugin dans leurs versions les plus récentes, correctement packagées pour Ubuntu.

    ⚠️ Si docker.io ou docker-compose ont déjà été installés, les supprimer avant de procéder — ils entreront en conflit avec docker-ce.

2. Installation de Docker
2.1 Suppression des éventuels paquets conflictuels

sudo apt remove docker.io docker-compose containerd runc -y
sudo apt autoremove -y

2.2 Installation des prérequis

sudo apt update
sudo apt install ca-certificates curl gnupg -y

2.3 Ajout de la clé GPG officielle Docker

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /tmp/docker.gpg
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg /tmp/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

Vérification — le fichier doit exister et peser environ 2 760 octets :

ls -lh /etc/apt/keyrings/docker.gpg

2.4 Ajout du dépôt Docker

La commande suivante doit être saisie en une seule ligne — les retours à la ligne dans un terminal Ubuntu peuvent tronquer la commande et produire un fichier docker.list vide sans erreur visible :

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list

Vérification — le fichier ne doit pas être vide :

cat /etc/apt/sources.list.d/docker.list
# attendu : deb [arch=amd64 signed-by=...] https://download.docker.com/linux/ubuntu noble stable

2.5 Installation de Docker

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin -y

2.6 Démarrage et activation au boot

sudo systemctl enable --now docker

2.7 Vérification

sudo docker run --rm hello-world
sudo docker compose version

La sortie Hello from Docker! confirme que le démon fonctionne et peut télécharger des images. docker compose version doit afficher v5.x.x.
3. Déploiement de Timesketch
3.1 Téléchargement du script de déploiement officiel

cd ~
curl -s -O https://raw.githubusercontent.com/google/timesketch/master/contrib/deploy_timesketch.sh
chmod 755 deploy_timesketch.sh

    ⚠️ L'URL doit être copiée exactement — toute autocorrection du navigateur ou du terminal (notamment githubusercontent → githubuserconsent) rend le script inaccessible sans message d'erreur explicite.

3.2 Exécution du script

sudo ./deploy_timesketch.sh

Le script effectue les opérations suivantes :

    Vérifie la présence de Docker et docker compose
    Règle vm.max_map_count à 262144 (requis par OpenSearch)
    Génère les fichiers de configuration dans ~/timesketch/
    Télécharge les images Docker de la stack

À la question Would you like to start the containers? [y/N] — répondre N. Les conteneurs seront démarrés manuellement à l'étape suivante pour un meilleur contrôle de la séquence de démarrage.
3.3 Réglage permanent de vm.max_map_count

Le script règle vm.max_map_count pour la session courante uniquement. Pour que ce réglage persiste après un redémarrage de la VM :

echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-opensearch.conf
sudo sysctl -p /etc/sysctl.d/99-opensearch.conf

Vérification :

sysctl vm.max_map_count   # attendu : vm.max_map_count = 262144

3.4 Démarrage de la stack

cd ~/timesketch
sudo docker compose up -d

Le premier démarrage télécharge les images manquantes (~1,5 Go). Durée variable selon la connexion — compter 5 à 15 minutes.

Suivre la progression :

sudo docker compose ps

Attendre que tous les services affichent le statut healthy ou running avant de continuer. OpenSearch est le service le plus lent à démarrer (30 à 60 secondes après les autres).

Résultat attendu :

NAME                STATUS
nginx               Up
opensearch          Up (healthy)
postgres            Up (healthy)
redis               Up (healthy)
timesketch-web      Up
timesketch-worker   Up

3.5 Création d'un utilisateur

Attendre une minute supplémentaire après que tous les services soient Up pour laisser timesketch-web terminer son initialisation, puis :

sudo docker compose exec timesketch-web tsctl create-user <nom_utilisateur>

Saisir et confirmer un mot de passe. Message attendu : User account for <nom_utilisateur> created/updated
3.6 Accès à l'interface web

Ouvrir Firefox dans la VM :

http://localhost

    ⚠️ Utiliser http:// et non https:// — aucun certificat SSL n'est configuré par défaut. Si le navigateur redirige automatiquement vers HTTPS, ouvrir une fenêtre de navigation privée (Ctrl+Maj+P) et retaper http://localhost.

4. Commandes de gestion quotidienne
Démarrer la stack

cd ~/timesketch
sudo docker compose up -d

Arrêter la stack (fin de session)

cd ~/timesketch
sudo docker compose stop

Vérifier l'état des services

sudo docker compose ps

Consulter les logs d'un service

sudo docker compose logs opensearch --tail 50
sudo docker compose logs timesketch-web --tail 50

Redémarrer un service spécifique

sudo docker compose restart opensearch

    ⚠️ Timesketch ne démarre pas automatiquement avec la VM. Lancer docker compose up -d manuellement au début de chaque session si Timesketch est requis pour l'analyse en cours.

5. Considérations sur les ressources
Service 	RAM allouée 	Note
OpenSearch 	2 Go (configuré par le script) 	Valeur minimale fonctionnelle
timesketch-web + worker 	~500 Mo 	Variable selon la charge
postgres + redis + nginx 	~300 Mo 	Stables, peu variables
Total stack 	~2,8 Go 	Sur 8 Go de RAM VM

Recommandation : ne pas faire tourner Timesketch et Splunk simultanément sauf nécessité. Splunk seul consomme ~1,5 Go supplémentaires — la marge restante pour l'OS et les outils d'analyse devient insuffisante.

# Arrêter Timesketch avant de lancer Splunk
cd ~/timesketch && sudo docker compose stop
sudo -u splunk /opt/splunk/bin/splunk start

6. Problèmes rencontrés et résolutions
docker-compose-plugin introuvable via apt

Symptôme :

E: Impossible de trouver le paquet docker-compose-plugin

Cause : tentative d'installation depuis les dépôts Ubuntu standard, qui ne fournissent pas ce paquet.

Résolution : ajouter le dépôt officiel Docker (download.docker.com) avant toute installation — section 2 de ce document.
docker.service échoue au démarrage via systemd

Symptôme :

Job for docker.service failed because the control process exited with error code.

Cause : démarrage systemd trop tôt après l'installation, en conflit avec un ancien socket ou une instance résiduelle de docker.io. Le démon dockerd lui-même fonctionne correctement — le problème est isolé à l'orchestration systemd.

Diagnostic : lancer sudo dockerd directement pour observer si le démon démarre sans erreur. S'il démarre, le problème est bien au niveau systemd.

Résolution :

sudo systemctl reset-failed docker.service docker.socket
sudo systemctl restart containerd
sudo systemctl start docker
sudo systemctl status docker

OpenSearch reste unhealthy indéfiniment

Symptôme :

[WARN] this node is unhealthy: health check failed on
       [/usr/share/opensearch/data/nodes/0]

Causes possibles (par ordre de probabilité) :

    Disque de la VM saturé — OpenSearch refuse d'écrire si l'espace disponible est insuffisant.

    df -h /   # vérifier que Use% < 90%

    vm.max_map_count trop bas — OpenSearch exige 262144 minimum.

    sysctl vm.max_map_count
    sudo sysctl -w vm.max_map_count=262144   # si valeur < 262144

    Mémoire insuffisante — la VM était initialement à 4,5 Go, ce qui provoquait une charge système à 52 (charge normale : 1 à 4). Résolution : augmenter la RAM VM à 8 Go depuis VirtualBox.

Séquence de diagnostic recommandée :

df -h /
sysctl vm.max_map_count
free -h
sudo docker compose logs opensearch --tail 30

docker compose exec reste figé (pas de réponse)

Symptôme : sudo docker compose exec timesketch-web tsctl create-user ne répond pas, Ctrl+C nécessaire pour interrompre.

Cause : timesketch-web n'a pas terminé son initialisation interne, ou la VM manque de ressources (RAM ou CPU saturés).

Résolution : attendre 2 à 3 minutes supplémentaires après que tous les services soient Up, puis relancer la commande. Vérifier free -h pour s'assurer que la mémoire disponible est suffisante (> 1 Go).
Résultat attendu en fin de déploiement

✅ Docker 29.x installé depuis le dépôt officiel
✅ docker compose version v5.x.x
✅ Stack Timesketch : 6 services à l'état Up/healthy
✅ Utilisateur Timesketch créé
✅ Interface accessible sur http://localhost
✅ vm.max_map_count = 262144 (persistant après redémarrage)
✅ Swap 4 Go actif (voir 01-vm-provisioning.md)

