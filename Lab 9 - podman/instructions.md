# LAB 9 - Podman : L'alternative Sécurisée et Native

### Prérequis

Connectez-vous avec un utilisateur standard (par exemple `plb`). 

---

## Partie 0 : Désisntallation total de Docker

Nous allons complètement désinstaller tout de docker pour installer Podman

```bash
# --- Arrêt des services et sockets Docker ---
sudo systemctl stop docker.service docker.socket

# --- Désinstallation des paquets Docker (Moteur et Desktop) ---
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras docker-desktop

# --- Nettoyage des dépendances et paquets orphelins ---
sudo apt-get autoremove -y --purge

# --- Suppression de toutes les données Docker (Images, Conteneurs, Volumes) ---
sudo rm -rf /var/lib/docker /var/lib/containerd

# --- Suppression des configurations Docker au niveau utilisateur ---
rm -rf ~/.docker

# --- Nettoyage des traces spécifiques à Docker Desktop ---
sudo rm -f /usr/local/bin/com.docker.cli
rm -rf ~/.local/share/docker-desktop
```

Une fois ces commandes exécutées, votre système Ubuntu sera parfaitement "propre". Pour s'assurer qu'il ne reste rien, tapez :

```bash
which docker
```

la commande ne devait rien retourner.
---

## Partie 1 : Installation et configuration zéro

Contrairement à Docker Rootless où il fallait lancer des scripts, ajouter des variables d'environnement (`DOCKER_HOST`) et démarrer un service utilisateur, Podman ne demande **rien de tout ça**.

**1. Installer Podman (via un compte avec droits sudo)**
```bash
sudo apt-get update
sudo apt-get install -y podman
```

**2. Vérifier l'installation (en tant qu'utilisateur standard)**
```bash
podman info
```
*Note : Dans le résultat, cherche la ligne `rootless: true`. Podman détecte automatiquement que tu n'es pas root et s'auto-configure.*

* Quel version de Podman vous avez installé ?

```bash
podman --version
```

**3. Installer podman-compose**

L'outil `podman-compose` est relativement récent et n'a pas été inclus dans les paquets par défaut. C'est Script Python. Pour l'installer :

```bash
sudo apt-get update

# installe pip3 : le gestionnaire de paquets pythons
sudo apt-get install -y python3-pip

# Installer la dernière version de podman-compose pour tous les utilisateurs du système
sudo pip3 install podman-compose
```

Une fois l'installation terminée, vérifie que la commande répond bien :

```bash
podman-compose --version
```
---

## Partie 2 : Le lancement (L'expérience Docker-like)

Prouvons que la transition est transparente pour un développeur.

**1. L'alias magique**
Configure ton terminal pour que la commande `docker` appelle en fait `podman` :
```bash
echo "alias docker=podman" >> ~/.bashrc
source ~/.bashrc
```

**2. Lancer un serveur web**
Utilisons exactement la même commande que pour Docker :
```bash
docker run -d --name web-podman -p 8080:80 nginx
```
*Note : Podman peut demander de quel registre tu veux tirer l'image (Docker Hub, Quay.io...). Choisis `docker.io/library/nginx`. oOu bien l'on configure le registre dans le fichier `/etc/containers/registries.conf` par l'indication `unqualified-search-registries = ["docker.io"]`*

**3. Vérification**
```bash
docker ps
curl http://localhost:8080
```
> *Succès : Le serveur tourne parfaitement.*

---

## Partie 3 : Exploration Système (Où est le Daemon ?)

C'est ici que la différence architecturale devient flagrante.

**1. Cherchons le Daemon central**
Dans un autre terminal, vérifie s'il y a un service central Podman qui tourne en arrière-plan :
```bash
ps aux | grep podmand
```
> *Résultat : Rien. Il n'y a pas de daemon central.*

**2. Qui possède vraiment le processus Nginx ?**
Cherchons le processus de notre conteneur sur la machine hôte :
```bash
ps -u $USER f
```
> *Observation : Vous verrez un programme léger appelé `conmon` (Container Monitor) qui est un processus enfant direct de ta session utilisateur, et qui surveille le processus `nginx` en dessous. Tout vous appartient directement !*

**3. Où sont les images et les disques virtuels ?**
En Docker classique, tout est dans `/var/lib/docker` (réservé à root). Regarde où Podman range tes affaires :
```bash
ls -la ~/.local/share/containers/storage/
```
> *Observation : Votre environnement conteneurisé vit entièrement dans votre dossier personnel.*

---

## Partie 4 : Test de Sécurité (Le Crash Test)

Refaisons exactement l'attaque de la Partie 1 du Lab précédent, où un utilisateur malveillant essaie de lire les mots de passe de l'administrateur.

**1. L'attaque (montage de la racine)**
```bash
podman run -it -v /:/host ubuntu cat /host/etc/shadow
```

**Que se passe-t-il ?** l'on obtiens un **`Permission denied`** net et sans bavure. 

**2. Tentative de contournement**
Avec Docker, l'attaquant pouvait rajouter `--privileged` ou `--userns=host` pour contourner la sécurité car le daemon Docker tournait en root. Essayons avec Podman :
```bash
podman run -it --privileged -v /:/host ubuntu cat /host/etc/shadow
```
> *Résultat : Toujours `Permission denied` !*
> **Explication :** Puisque l'on a lancé la commande Podman avec un utilisateur standard, le maximum de droits que le conteneur peut obtenir sur l'hôte, même en mode "privilégié", **ce sont vos propres droits d'utilisateur standard**. Il est mathématiquement impossible pour Podman de lire un fichier appartenant à root. La sécurité est plus élevée.

---

## Partie 5 : Les Quadlets (La magie Systemd)

Puisque Podman n'a pas de daemon pour redémarrer vos conteneurs si le serveur redémarre, comment fait-on en production ? On utilise **Quadlet**, qui transforme les conteneurs en vrais services Linux natifs.

**1. Créer le dossier pour les Quadlets**
```bash
mkdir -p ~/.config/systemd/user/
```

Les **Quadlets** (la fonctionnalité qui lit les fichiers `.container`) sont une "nouveauté" introduite à partir de la version **4.4 de Podman**. 

Comme nous sommes sur Ubuntu 22.04, le paquet installé par défaut via `apt-get` est une version 3.x (souvent la 3.4.4).

Grâce à Podman on peut générer ces fichiers Unit Systemd à partir d'un conteur créé.

**2. Lancez un conteneur normalement (une première fois)**
On va utiliser un nouveau nom pour éviter les conflits :
```bash
podman run -d --name site-genere -p 8082:80 nginx
```

**3. La Magie de Podman : Générer le service**
Demandez à Podman de regarder ce conteneur qui tourne et de générer automatiquement le fichier de configuration `systemd` complexe correspondant :

```bash
cd ~/.config/systemd/user/
podman generate systemd --name site-genere --files --new
```

*Note : L'option `--new` est géniale. Elle dit à systemd : "Chaque fois que tu démarres le service, crée un conteneur tout neuf, et quand tu l'arrêtes, supprime-le proprement".*

**4. Nettoyer le conteneur manuel**
Puisque systemd va maintenant gérer la création/suppression grâce à l'option `--new`, on supprime le conteneur qu'on a lancé à l'étape 2 :
```bash
podman rm -f site-genere
```

**5. Lancer le service via Systemd**
Maintenant, on indique à systemd de prendre le relais :
```bash
systemctl --user daemon-reload
```
Le fichier généré s'appelle généralement `container-<nom>.service`. On le démarre :
```bash
systemctl --user start container-site-genere.service
```

**6. Vérifier que ça marche**
Vérifie le statut côté Linux :
```bash
systemctl --user status container-site-genere.service
```
Et vérifie que le conteneur tourne bien côté Podman :
```bash
podman ps
```

**Le grand avantage :** Si vous voulez que ce conteneur démarre tout seul quand on allume le serveur Ubuntu (même sans se connecter), il suffit de taper :
```bash
systemctl --user enable container-site-genere.service
loginctl enable-linger $USER
```

C'est l'une des fonctionnalités les plus puissantes de Podman par rapport à Docker pour la gestion de serveurs en production !

> *Succès : Systemd a lu votre fichier, a appelé Podman en arrière-plan et a lancé le conteneur. Si le serveur physique redémarre, `systemd` relancera ce conteneur automatiquement pour vous, sans aucun besoin de droits root !*

---

## Partie 6 : Pod-man compose

Reprenez un fichier docker-compose utilisé dans les Labs précédents et lancer la stack à l'aide de la commande :

```bash
podman-compose up -d
```

### Bilan de l'exploration

En quelques minutes, nous avons déployé des conteneurs totalement sécurisés, stockés dans notre espace personnel, insensibles aux attaques d'élévation de privilèges, et gérés nativement par le système d'exploitation de la machine. C'est la puissance de Podman !