# LAB 1 - Découvrir Docker - Cgroups & Namespace

Cet TP sera effectué sur la machine hôte. Assurez vous d'avoir Docker installé.

## Partie 1 : Installation de Docker

Se réferrer à [la documentation officielle](https://docs.docker.com/engine/install/ubuntu/)

<details><summary>Instructions d'installation</summary>

Exécutez la commande suivante pour désinstaller tous les paquets en conflit :

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Ajouter la clé GPG officielle de Docker

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Configurer les dépots et Mettre à jour l'index des paquets apt

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Installation de la dernière version de Docker Engine

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```

</details>


Vérifiez que l'installation de Docker Engine est réussie en exécutant l'image `hello-world`

```bash
docker version
sudo systemctl status docker
sudo docker run hello-world
```

Pour éviter de taper sudo à chaque commande, ajoutez votre utilisateur au groupe docker.

```bash
sudo groupadd docker # Peut déjà exister
sudo usermod -aG docker $USER
```

Attention : Vous devrez vous déconnecter et vous reconnecter (ou redémarrer) pour que ce changement soit pris en compte.

## Partie 2 : Exploration de l'environnement Docker avec les commandes de base

À l'aide de quelques commandes de base de Docker, explorez l'environnement: 

* afficher l'aide de `docker` pour voir les options dont vous disposez

<details><summary>Correction</summary>

```bash
docker --help
```

</details>


* afficher la version de docker

<details><summary>Correction</summary>

```bash
docker version
```

</details>

* Lister les images présentes sur la machine

<details><summary>Correction</summary>

```bash
docker image ls
```

</details>

* Lister les conteneurs en cours d'exécution sur la machine de deux manières

<details><summary>Correction</summary>


```bash
docker ps
```

```bash
docker container ls
```

</details>

* Lancez un conteneur détaché du terminal dont l'image est `nginx:1.23`

<details><summary>Correction</summary>

```bash
sudo docker run -d nginx:1.23
```

</details>

* Lancez à nouveau un conteneur détaché du terminal dont l'image est `nginx:latest` et nom du conteneur `web` et inspecter ses caractérisques : container ID, adresse IP, Commande de lancement du conteneur ainsi que les arguments 

<details><summary>Correction</summary>

```bash
sudo docker run --name web -d nginx:latest
```

```bash
sudo docker inspect web
```

</details>

* visualisez les logs du conteneur `web`

<details><summary>Correction</summary>

```bash
sudo docker logs web
```

</details>

* Arreter le conteneur

<details><summary>Correction</summary>

```bash
sudo docker stop web
```

</details>

* Lister tous les conteneurs y compris ceux arrêtés sur la machine

<details><summary>Correction</summary>

```bash
sudo docker ps -a
```

</details>

* Relancez à nouveau le conteneur Web. 
  * Est-ce que vous pouvez accéder à votre application via votre navigateur ? Pourquoi ?
  * Est-ce que vous pouvez lister les fichers contenus dans votre conteneur ?

## Partie 3 : Sous le capot - Découverte des Namespaces et Cgroups

Jusqu'à présent, nous avons utilisé Docker pour lancer des applications. Mais comment un conteneur est-il réellement isolé de la machine hôte ? La magie repose sur deux technologies du noyau Linux : les **Namespaces** et les **Cgroups**.

  * **Namespaces (Cloisonnement)** : Ils créent une "illusion d'optique" pour le processus. Chaque conteneur obtient son propre ensemble de ressources (réseau, liste des processus, nom d'hôte), lui faisant croire qu'il est sur sa propre machine. C'est l'**isolation de la vue**.

  * **Cgroups (Contrôle)** : Ils limitent la quantité de ressources système (CPU, mémoire, disque) qu'un conteneur a le droit de consommer. C'est la **limitation des ressources**.

-----

### Les Namespaces en action (L'art de l'isolation)

Nous allons lancer un conteneur et comparer ce qu'on voit *dedans* par rapport à *dehors*.

**1. L'isolation des processus (PID Namespace)**

Lancez un conteneur interactif `ubuntu` :

```bash
docker run -it --rm busybox sh
```

Maintenant que vous êtes dans le shell du conteneur, listez les processus en cours :

```bash
# À l'intérieur du conteneur
ps aux
```

**Observation** : Vous ne voyez que très peu de processus \! Votre commande `bash` est le processus numéro 1 (PID 1). Pour le conteneur, il n'existe rien d'autre. C'est le **PID Namespace** en action. Sur votre machine hôte, la même commande afficherait des centaines de processus y compris ce processus conteneurisé.

**2. L'isolation du réseau (Network Namespace)**

Dans le même conteneur, regardez la configuration réseau :

```bash
# Toujours à l'intérieur du conteneur
ifconfig -a
```

**Observation** : Le conteneur possède sa propre interface réseau (souvent `eth0`) avec une adresse IP privée (ex: `172.17.0.2`). Il est déconnecté du "vrai" réseau de votre machine. C'est le **Network Namespace**.

Tapez `exit` pour quitter le conteneur.

-----

### Les Cgroups en action (La limitation des ressources)

Maintenant, voyons comment Docker peut contraindre un conteneur. Nous allons lancer un conteneur avec une limite de mémoire très stricte et voir ce qui se passe quand il essaie de la dépasser.

**1. Limiter la mémoire**

Lancez un conteneur `ubuntu` en lui accordant seulement 100 mégaoctets de mémoire vive :

```bash
# --memory="100m" est la directive Cgroup
docker run -it --rm --memory="100m" --memory-swap="100m" ubuntu bash
```

*`--memory-swap="100m"` empêche le conteneur d'utiliser le disque si la RAM est pleine.*

Le conteneur est lancé. Essayons de consommer plus de 100 Mo de mémoire. Pour cela, nous allons utiliser un petit outil de test de charge. Installez-le :

```bash
# À l'intérieur du conteneur
apt-get update && apt-get install -y stress-ng
```

Maintenant, lancez un test qui alloue 200 Mo de mémoire :

```bash
# Cette commande va tenter de dépasser la limite
stress-ng --vm 1 --vm-bytes 200M &
```

**Observation** : La commande va s'arrêter brutalement et le conteneur se terminera avec le message **"Killed"**. Votre conteneur n'a pas planté votre machine hôte ; il a été arrêté net par le noyau Linux car le **Cgroup** a appliqué la règle que vous aviez fixée.

Pour confirmer, inspectez le dernier conteneur qui s'est arrêté (`docker ps -l`) et cherchez la raison de sa sortie (`docker inspect <CONTAINER_ID>`). Vous verrez que son état est `OOMKilled: true` (Out Of Memory Killed). C'est la preuve ultime du contrôle des Cgroups \!

Dans certain cas le conteneur ne se termine pas immediatement quoi qu'il y'a eu un `OOMKilled`; la raison c'est que Docker lance le conteneur avec un processus principal, qui a l'identifiant de processus (PID) 1 à l'intérieur du conteneur. Ce dernier lance un processus "enfant" qui commence à consommer une quantité de mémoire vive (RAM) qui dépasse la limite allouée au conteneur. Le noyau Linux de la machine hôte détecte cette surconsommation. Son mécanisme de secours, appelé `OOMKiller` (Out Of Memory Killer), s'active pour protéger la stabilité du système. Il identifie le processus le plus fautif et le "tue" brutalement (en lui envoyant un signal `SIGKILL`).

<details><summary>Plus de détails</summary>

```bash
stress-ng --vm 1 --vm-bytes 200M &
[1] 552
root@6d0009da658b:/# stress-ng: info:  [552] defaulting to a 1 day, 0 secs run per stressor
stress-ng: info:  [552] dispatching hogs: 1 vm

root@6d0009da658b:/# ps
    PID TTY          TIME CMD
      1 pts/0    00:00:00 bash
    552 pts/0    00:00:00 stress-ng
    553 pts/0    00:00:00 stress-ng-vm
    618 pts/0    00:00:00 stress-ng-vm           <----- PID de la tâche de Stress
    619 pts/0    00:00:00 ps
root@6d0009da658b:/# ps
    PID TTY          TIME CMD
      1 pts/0    00:00:00 bash
    552 pts/0    00:00:00 stress-ng
    553 pts/0    00:00:00 stress-ng-vm
    658 pts/0    00:00:00 stress-ng-vm           <----- Nouveau PID après que le Processus ai été tué OOMKilled
    659 pts/0    00:00:00 ps
root@6d0009da658b:/# ps
    PID TTY          TIME CMD
      1 pts/0    00:00:00 bash
    552 pts/0    00:00:00 stress-ng
    553 pts/0    00:00:00 stress-ng-vm
    701 pts/0    00:00:00 stress-ng-vm           <----- Nouveau PID après que le Processus ai été tué OOMKilled
    702 pts/0    00:00:00 ps
root@6d0009da658b:/# ps
    PID TTY          TIME CMD
      1 pts/0    00:00:00 bash
    552 pts/0    00:00:00 stress-ng
    553 pts/0    00:00:00 stress-ng-vm
    729 pts/0    00:00:00 stress-ng-vm           <----- Nouveau PID encore; on peut constaté qu'il a été tué plusieurs fois
    730 pts/0    00:00:00 ps
```

</details>

-----

## Partie 4 : Les limites de l'isolation

Les namespaces sont puissants, mais **ils ne créent pas une machine virtuelle impénétrable.** Le noyau Linux de l'hôte est toujours le maître et peut voir tout ce qui se passe à l'intérieur des conteneurs.

Démontrons-le en espionnant une variable d'environnement "secrète" depuis l'hôte.

**1. Lancez un conteneur Apache avec une variable d'environment servant de mot de passe**

Nous allons démarrer un serveur web Apache et lui passer un mot de passe via une variable d'environnement. C'est une pratique courante mais risquée.

```bash
docker run -d --name apache-secret -e PASSWORD=MySecretPassw0rd httpd:alpine
```

**2. Trouvez le "vrai" PID du processus**

Le conteneur est lancé. Depuis l'hôte, utilisons la commande `docker top` pour voir les processus qui tournent à l'intérieur et, surtout, leur PID du point de vue du système d'exploitation hôte.

```bash
docker top apache-secret
```

Vous devriez voir un ou plusieurs processus `httpd`. Repérez le PID du processus principal (celui qui est lancé par `root`). **Notez ce numéro PID**, nous en aurons besoin. Dans l'exemple ci-dessus, ce serait `12345`.

**3. Espionnez les variables d'environnement depuis l'hôte**

Sur les systèmes Linux, le dossier `/proc` contient des informations sur tous les processus en cours. Le fichier `/proc/PID/environ` contient toutes les variables d'environnement d'un processus.

Maintenant, utilisons le PID que vous avez noté pour lire ce fichier **depuis la machine hôte**. Vous aurez besoin de `sudo` car vous accédez à des informations sur un processus qui ne vous appartient pas.

Remplacez `VOTRE_PID` par le numéro que vous avez noté :

```bash
# La commande `tr` remplace les caractères nuls par des sauts de ligne pour rendre la sortie lisible
sudo cat /proc/VOTRE_PID/environ | tr '\0' '\n'
```

Pour trouver directement notre secret, filtrons la sortie :

```bash
sudo cat /proc/VOTRE_PID/environ | tr '\0' '\n' | grep PASSWORD
```

**Observation et conclusion**

Bingo \! 🏆 Le mot de passe `PASSWORD=MySecretPassw0rd` est clairement visible en texte brut par l'administrateur de la machine hôte.

C'est une leçon de sécurité cruciale : **l'isolation des namespaces protège les conteneurs les uns des autres, mais pas de l'hôte**. C'est pourquoi, en production, il ne faut jamais passer de secrets sensibles (mots de passe, clés d'API) directement via des variables d'environnement. On utilise plutôt des mécanismes sécurisés comme **Docker Secrets** ou des coffres-forts numériques (vaults) qui montent les secrets dans le conteneur de manière plus sûre.

