# LAB 7 - Docker en Production & Sécurité

### Mise en Situation

Félicitations, votre application Web (de Vote - Lab 6 bonus) est un succès en développement \! La direction souhaite maintenant la déployer en production. Votre rôle en tant qu'ingénieur DevOps/SRE est de vous assurer que l'environnement Docker est non seulement fonctionnel, mais aussi **sécurisé, robuste et optimisé**.

Dans ce lab nous verrons : 
* les étapes essentielles du durcissement d'un environnement Docker
* Les outils pour checker la conformité à ceetains standard de sécurité comme CIS benchmark
* La configuration du daemon Docker
* Le scan de vulnérabilités de vos images.

Nous allons transformer notre installation Docker de base en une forteresse prête pour la production.

-----

## Partie 1 : Comprendre le Stockage Docker

Le moteur de stockage (storage driver) est le cœur de Docker. Il gère la manière dont les images et les conteneurs sont stockés sur le disque. Comprendre son fonctionnement est essentiel pour optimiser les performances et la gestion de l'espace.

**1. Identifiez votre Moteur de Stockage**
Docker supporte plusieurs moteurs comme `overlay2`, `aufs`, `devicemapper`, etc. `overlay2` est le standard moderne sur la plupart des systèmes Linux.

  * Utilisez la commande `docker info` pour trouver le moteur de stockage utilisé par votre installation Docker.

    ```bash
    docker info | grep "Storage Driver"
    ```
  * **Observation :**

    Vous verrez probablement `Storage Driver: overlay2`. 
    Ce moteur utilise un système de fichiers en couches (Union File System) qui est très efficace. Une image Docker n'est pas un gros fichier monolithique, mais une superposition de couches en lecture seule. Quand vous lancez un conteneur, Docker ajoute une fine couche en écriture par-dessus. C'est ce qui rend le démarrage des conteneurs si rapide et efficace en termes d'espace disque.

**2. Visualisez les couches d'une image**

  * Utilisez la commande `docker history` pour voir les différentes couches qui composent l'image `redis` du lab précédant.
  * Allez sur Docker hub pour remonter les images jusqu'au `Scratch` : https://hub.docker.com/_/scratch 

    ```bash
    docker history redis:alpine
    ```

  * **Observation :**

    Vous verrez chaque instruction du `Dockerfile` de Redis qui a créé une nouvelle couche. C'est la preuve visuelle de ce système de fichiers en couches.

-----

## Partie 2 : Auditer votre Configuration avec les Benchmarks CIS

Le **Center for Internet Security (CIS)** publie des guides de "durcissement" (hardening) qui sont des standards de l'industrie. Heureusement, il existe des outils open-source pour auditer automatiquement notre configuration par rapport à ces recommandations.

**1. Lancez le script d'audit `docker-bench-security`**
Cet outil, fourni par Docker, est lui-même un conteneur qui va inspecter la configuration de votre hôte et de votre daemon Docker.

  * Exécutez le conteneur de benchmark. La longue commande est nécessaire pour lui donner accès aux informations de l'hôte qu'il doit auditer.

    ```bash
    docker run -it --net host --pid host --userns host --cap-add audit_control \
    -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
    -e DOCKER_API_VERSION=1.44 \
    -v /var/lib:/var/lib:z \
    -v /var/run/docker.sock:/var/run/docker.sock:z \
    -v /usr/lib/systemd:/usr/lib/systemd:z \
    -v /etc:/etc:z --label docker_bench_security \
    --privileged \
    docker/docker-bench-security
    ```

  * **Observation :**

    Le script va générer un long rapport avec des sections, des avertissements (`[WARN]`) et des notes (`[INFO]`). Ne vous inquiétez pas si vous avez beaucoup d'avertissements, c'est normal sur une installation par défaut.

**2. Analysez quelques résultats clés**

  * **`[WARN] 1.1 - Ensure a separate partition for containers is used`** : En production, il est recommandé de monter `/var/lib/docker` sur sa propre partition pour éviter qu'un conteneur hors de contrôle ne remplisse le disque racine du système.
  * **`[WARN] 4.1 - Ensure a user for the container has been created`** : C'est une alerte critique. Elle vérifie si vos conteneurs tournent avec l'utilisateur `root`, ce qui est une mauvaise pratique de sécurité.
  * **`[WARN] 5.12 - Ensure the default ulimit is configured appropriately`** : Limiter les ressources par défaut empêche les attaques par déni de service (DoS).

Ce rapport est votre **liste de tâches** pour sécuriser votre environnement.


## Partie 3 : Durcissement du Daemon Docker (`daemon.json`)

Beaucoup de recommandations du benchmark CIS se corrigent en configurant le daemon Docker via un seul fichier : `/etc/docker/daemon.json`.

### A. Démonstration de la faille de sécurité (Rootful)

Prouvons qu'un simple développeur membre du groupe `docker` peut pirater la machine hôte.

**1. Création d'un utilisateur "test"**
En tant qu'administrateur (ou via `sudo`), créez un utilisateur et ajoutez-le au groupe docker.

```bash
sudo adduser test_user
# (Mettez un mot de passe simple, tapez Entrée pour le reste)

sudo usermod -aG docker test_user
```

**2. L'attaque de l'élévation de privilèges**
Connectez-vous en tant que `test_user` et tentez de lire le fichier `/etc/shadow` de la machine hôte (qui contient les mots de passe chiffrés et qui est strictement réservé à `root`).

```bash
su - test_user

# Tentative de lecture normale (échoue, Permission denied)
cat /etc/shadow

# L'attaque : on lance un conteneur en montant la racine de l'hôte (/) dans le dossier /host du conteneur
docker run -it -v /:/host ubuntu cat /host/etc/shadow
```

**Que s'est-il passé ?**
Le fichier s'est affiché \! L'utilisateur `test_user` vient de lire un fichier critique de l'hôte grâce à Docker. C'est inacceptable sur un serveur partagé.

### B. Durcissement de Docker (Rootful)

**1. Créez ou modifiez le fichier `daemon.json`**

  * Ouvrez (ou créez) ce fichier avec des privilèges `sudo`.

    ```bash
    sudo nano /etc/docker/daemon.json
    ```

**2. Appliquez des configurations de sécurité**

  * Copiez le contenu ci-dessous dans votre fichier `daemon.json`. Chaque ligne est une mesure de sécurité.

<details><summary>Contenu recommandé pour `daemon.json`</summary>

```json
{
  "icc": false,
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true,
  "default-ulimits": {
    "nproc": {
      "Hard": 1024,
      "Name": "nproc",
      "Soft": 1024
    },
    "nofile": {
      "Hard": 65536,
      "Name": "nofile",
      "Soft": 65536
    }
  }
}
```

</details>

  * **Explication des paramètres clés :**
      * `"icc": false` : Désactive la communication inter-conteneurs sur le réseau par défaut. C'est un principe de moindre privilège.
      * `"userns-remap": "default"` : Active le mapping des namespaces utilisateurs. Un `root` dans le conteneur sera mappé à un utilisateur non-privilégié sur l'hôte, une mesure de sécurité majeure.
      * `"log-opts"` : Configure la rotation des logs pour éviter de saturer le disque.
      * `"no-new-privileges": true` : Empêche les conteneurs d'escalader leurs privilèges.
      * `"default-ulimits"` : Applique des limites de ressources par défaut à tous les conteneurs.

**3. Redémarrez le daemon Docker**

  * Pour que les changements soient pris en compte, vous devez redémarrer Docker.

    ```bash
    sudo systemctl restart docker
    ```

### C. Réessayez l'attaque de la partie A 

Vous devrez constater que vous êtes empechés.

-----

## Partie 4 : Scan de Vulnérabilités des Images

Une image, même officielle, peut contenir des librairies avec des failles de sécurité connues (CVEs). Il est **impératif** de scanner vos images avant de les déployer. Nous allons utiliser **Trivy**, un scanner de vulnérabilités open-source très populaire.

**1. Installez Trivy**

  * Suivez les instructions d'installation pour le système Ubuntu :

    ```bash
    sudo apt-get install wget apt-transport-https gnupg lsb-release
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
    echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
    sudo apt-get update
    sudo apt-get install trivy
    ```

**2. Scannez une image**

  * Scannons une image potentiellement "ancienne" pour voir ce que Trivy trouve.

    ```bash
    # Nous scannons une image de l'application de vote du lab précédent
    trivy image python:3.9-slim
    ```

**3. Analysez les résultats**

  * **Observation :** Trivy va afficher un tableau des vulnérabilités trouvées, classées par sévérité (`HIGH`, `CRITICAL`...). Il vous donne la librairie affectée, la CVE associée et la version qui corrige la faille.

  * **Action :** La décision à prendre dépend de la criticité. Une faille `CRITICAL` sur une librairie exposée à internet (comme `openssl`) doit être corrigée immédiatement, souvent en mettant à jour l'image de base (`FROM python:3.9-slim` vers `FROM python:3.11-slim` par exemple) et en reconstruisant.

L'intégration d'un scan de vulnérabilités dans votre pipeline CI/CD est une pratique standard en production.