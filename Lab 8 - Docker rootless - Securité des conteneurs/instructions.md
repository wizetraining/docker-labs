# LAB 8 - Sécurité et Supervision : Docker Rootless & Monitoring Multi-Utilisateurs

### Introduction : Le problème du "Root" dans Docker

Par défaut, le service Docker (le *daemon* `dockerd`) fonctionne avec les privilèges administrateur (`root`). De plus, pour qu'un utilisateur normal puisse utiliser la commande `docker`, on l'ajoute souvent au groupe système `docker`.

**Le danger :** Un utilisateur appartenant au groupe `docker` a virtuellement les droits `root` sur toute la machine. Il peut lancer un conteneur qui monte le disque dur principal et modifier les mots de passe du système \!

Pour pallier ce problème, deux concepts existent :

1.  **Docker Rootless :** Permet d'exécuter le daemon Docker et les conteneurs en tant qu'utilisateur non-privilégié. Si le conteneur est compromis, l'attaquant n'a pas les droits `root` sur la machine hôte.
2.  **Alternatives natives :** Des technologies comme **Podman** ont été conçues dès le départ pour être "Rootless" et sans daemon central.

### Objectifs du Lab

1.  Démontrer la faille de sécurité du Docker classique (Rootful).
2.  Installer et configurer Docker Rootless pour un utilisateur spécifique.
3.  Identifier quel utilisateur fait tourner quel conteneur sur une machine partagée.
4.  Mettre en place des outils de supervision (Monitoring) pour surveiller les ressources.

-----

## Partie 1 : Rappel de la faille de sécurité (Rootful)

Comme vu dans le **Lab 7**, le simple utilisateur `test_user` a pu lire un fichier critique de l'hôte grâce à Docker.
C'est inacceptable sur un serveur partagé.

-----

## Partie 2 : Mise en place de Docker Rootless

Nous allons maintenant configurer Docker pour qu'il s'exécute **uniquement** dans l'espace utilisateur de `plb`.

Docker Rootless est documenté https://docs.docker.com/engine/security/rootless/

**1. Prérequis système**

Quittez la session `test_user` (tapez `exit`) pour revenir sur votre compte principal `plb`, et installez les dépendances nécessaires aux espaces de noms utilisateurs (User Namespaces).

```bash
exit # Retour au compte principal
sudo apt-get update
sudo apt-get install -y uidmap dbus-user-session
```

**2. Désactiver le Docker "Rootful" pour cet utilisateur**
Retirez l'utilisateur `plb` du groupe `docker` pour lui enlever ses super-pouvoirs.

```bash
sudo gpasswd -d plb docker
```

Et plus important: Le démon Docker système est déjà en cours d’exécution, Désactivez le :

```bash
sudo systemctl disable --now docker.service docker.socket
sudo rm /var/run/docker.sock
```

**Important :** Tant que vous ne les avez pas arrêtés et désactivés, vous utilisez toujours Docker en mode **root (rootful)**.


**3. Installation de Docker Rootless**
Toujours en tant que utilisateur `plb`, lancez le script d'installation Rootless officiel fourni par Docker.

```bash
dockerd-rootless-setuptool.sh install
```

**4. Configuration de l'environnement**

Le script va afficher des lignes à ajouter dans votre profil (généralement `~/.bashrc`). Sans cela, la commande `docker` tentera toujours de contacter le daemon `root`.

<details><summary>Correction (Commandes à exécuter)</summary>

```bash
# Ajoutez ces lignes au fichier .bashrc de test_user
echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock' >> ~/.bashrc

# Rechargez la configuration
source ~/.bashrc
```

Vérifiez que vous êtes bien en Rootless :

```bash
docker info | grep "rootless"
```

*(Vous devriez voir `rootless: true` ou context `rootless`)*

</details>

**5. Test de sécurité**
Refaites l'attaque de la Partie 1 (tentative de lire `/etc/shadow` via le volume `-v /:/host`).
L'opération échouera avec un message *Permission denied*. Votre machine est désormais sécurisée \!

**6. Controle du demon Docker**
Docker étant désormais installé en rootless, l'utiliseur `plb` peux arreter, démarrer, charger son demon Docker

```bash
systemctl --user status docker
systemctl --user stop docker
systemctl --user enable docker
systemctl --user start docker
```
-----

## Partie 3 : Multi-utilisateurs : Qui lance quoi ?

Dans un environnement où plusieurs utilisateurs ont leur propre instance Rootless, comment l'administrateur système peut-il superviser l'activité ?

**1. Lancer un conteneur en Rootless**

Toujours en tant que `plb`, lancez un serveur Nginx.

```bash
docker run -d --name nginx-dev -p 8080:80 nginx
```

**2. L'analyse côté Système (en tant qu'administrateur)**
Ouvrez un **deuxième terminal** sur la même machine, connecté en tant que `root`. Nous allons inspecter les processus système.

Cherchez le processus `nginx` :

```bash
ps aux | grep nginx
```

<details><summary>Analyse des résultats</summary>

Exemple de résultat:

```
root@plb:~# ps aux | grep nginx
plb       660274  0.1  0.0  14860  9216 ?        Ss   07:26   0:00 nginx: master process nginx -g daemon off;
100100    660339  0.0  0.0  15316  3760 ?        S    07:26   0:00 nginx: worker process
100100    660340  0.0  0.0  15316  3760 ?        S    07:26   0:00 nginx: worker process
100100    660341  0.0  0.0  15316  3760 ?        S    07:26   0:00 nginx: worker process
100100    660342  0.0  0.0  15316  3760 ?        S    07:26   0:00 nginx: worker process
100100    660343  0.0  0.0  15316  3760 ?        S    07:26   0:00 nginx: worker process
100100    660344  0.0  0.0  15316  3760 ?        S    07:26   0:00 nginx: worker process
100100    660345  0.0  0.0  15316  3760 ?        S    07:26   0:00 nginx: worker process
100100    660346  0.0  0.0  15316  3696 ?        S    07:26   0:00 nginx: worker process
root      660362  0.0  0.0  11648  2648 pts/5    S+   07:27   0:00 grep --color=auto nginx
```

Contrairement au Docker classique où tous les processus conteneurisés appartiennent à l'utilisateur `root`, ici vous verrez que le processus `nginx` appartient à `plb` (ou à un sous-UID mappé à cet utilisateur) \!

Si un conteneur consomme trop de CPU, l'administrateur système peut immédiatement voir **quel utilisateur Linux** est responsable en regardant simplement la colonne propriétaire de la commande `ps` ou `top`.

</details>

Listez tous les procésus lancés par l'utilisateur `plb`:

```bash
ps -u plb f
```

<details><summary>Analyse des résultats</summary>

Exemple de résultat:

```
 659887 ?        Ssl    0:00  \_ rootlesskit --state-dir=/run/user/1000/dockerd-rootless --net=slirp4netns --mtu=65520 --slirp4netns-sandbox=auto --slirp4netns-seccomp=auto 
 659899 ?        Sl     0:00  |   \_ /proc/self/exe --state-dir=/run/user/1000/dockerd-rootless --net=slirp4netns --mtu=65520 --slirp4netns-sandbox=auto --slirp4netns-seccom
 659924 ?        Sl     0:00  |   |   \_ dockerd
 659949 ?        Ssl    0:00  |   |       \_ containerd --config /run/user/1000/docker/containerd/containerd.toml
 660294 ?        Sl     0:00  |   |       \_ /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 8080 -container-ip 172.17.0.2 -container-port 80 -use-listen-fd
 660301 ?        Sl     0:00  |   |       \_ /usr/bin/docker-proxy -proto tcp -host-ip ::1 -host-port 8080 -container-ip 172.17.0.2 -container-port 80 -use-listen-fd
 659916 ?        S      0:00  |   \_ slirp4netns --mtu 65520 -r 3 --disable-host-loopback --enable-sandbox --enable-seccomp 659899 tap0
 660247 ?        Sl     0:00  \_ /usr/bin/containerd-shim-runc-v2 -namespace moby -id ac1a99f145e10a05d0d920fb9f8b0e4b75274280ea610d616a2c41b35de66655 -address /run/user/100
 660274 ?        Ss     0:00      \_ nginx: master process nginx -g daemon off;

```

</details>

#### Supervision Multi-utilisateurs : La vue de l'administrateur (Root)

En tant qu'administrateur du système, vous devez être capable de surveiller l'activité de tous les utilisateurs, même s'ils utilisent Docker Rootless.

**1. Exécution de la commande standard**
Connectez-vous sur votre compte principal (ou `root`) et lancez :
```bash
sudo docker ps -a
```

* **Voyez-vous les conteneurs des utilisateurs ?** 

<details><summary>Analyse des résultats</summary>

Non, la liste est probablement vide ou ne montre que les conteneurs lancés par l'instance Docker système ("Rootful"), ou une erreur parce que nous avons arreté le docker rootfull.

</details>

* **Pourquoi selon vous ?** 

<details><summary>Analyse des résultats</summary>

Car Docker Rootless repose sur une isolation totale. Chaque utilisateur possède son propre processus **daemon** (le moteur Docker) et son propre **socket** (le canal de communication). La commande `docker` par défaut tente de communiquer avec le socket système situé dans `/var/run/docker.sock`, qui n'a aucune connaissance des moteurs privés lancés par l'utilisateur `plb` ou `test_user`.

</details>

**2. Accéder aux conteneurs d'un utilisateur spécifique**
Pour voir les conteneurs de l'utilisateur `plb`, vous devez rediriger votre client Docker vers son socket privé. 

**Étape A : Trouver l'UID de l'utilisateur**
```bash
id -u plb
# Supposons que le résultat est 1001
```

**Étape B : Interroger le socket de l'utilisateur**
Utilisez la variable d'environnement `DOCKER_HOST` pour pointer vers le socket situé dans le répertoire de run de l'utilisateur :

```bash
# Remplacer 1001 par l'UID réel trouvé précédemment
sudo DOCKER_HOST=unix:///run/user/1001/docker.sock docker ps -a
```

**Note :** _Si l'utilisateur n'utilise pas systemd, le socket peut se trouver dans son répertoire home : `unix:///home/plb/.docker/run/docker.sock`._

**3. Analyse des processus système**
Vous pouvez également vérifier que les processus appartiennent bien à l'utilisateur au niveau de l'OS :
```bash
ps -u plb f
```
Vous verrez alors l'arbre des processus où le `dockerd` de l'utilisateur pilote les processus `containerd` et les applications (comme `nginx`) avec ses propres privilèges.

-----

## Partie 4 : Superviser (Monitorer) les Conteneurs

Que vous soyez en Rootful ou Rootless, vous devez savoir comment votre application se comporte (CPU, RAM, Réseau).

### A. Les outils natifs (CLI)

Retournez sur le terminal de `plb`.

**1. Consommation en temps réel (`docker stats`)**
Cette commande est l'équivalent de `top` ou du Gestionnaire de Tâches pour Docker.

```bash
docker stats
```

*Note : Appuyez sur `Ctrl+C` pour quitter.*

**2. Audit des événements (`docker events`)**
Utile pour savoir qui a fait quoi (démarrage, arrêt, crash de conteneurs).
Ouvrez un terminal, lancez `docker events`.
Dans un autre terminal, lancez un conteneur ou arrêtez-le. Vous verrez les événements s'afficher en temps réel. C'est crucial pour le debugging système.

### B. Outil interactif : ctop

Pour aller plus loin que `docker stats`, les administrateurs utilisent souvent **ctop** (Container Top), un outil tiers léger et très visuel.

**1. Installation de ctop (en tant qu'admin)**

```bash
sudo wget https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-amd64 -O /usr/local/bin/ctop
sudo chmod +x /usr/local/bin/ctop
```

**2. Utilisation (en tant que plb)**
Lancez l'interface :

```bash
ctop
```

<details><summary>Ce que vous pouvez faire avec ctop</summary>

  * Vous voyez une liste colorée de vos conteneurs.
  * Sélectionnez un conteneur et appuyez sur **Entrée**.
  * Un menu s'ouvre permettant d'afficher les logs instantanément, de stopper le conteneur, ou de voir un graphique détaillé de la mémoire \!

</details>

### C. Chalenges pour la collecte des logs et monitoring

Dans un vrai environnement de production, on n'utilise pas la ligne de commande pour monitorer. On déploie une stack dédiée (souvent via Docker Compose ou Kubernetes) :

* La sortie de la commande `docker events` **n'est pas stockée dans un fichier texte** par défaut. C'est un flux (stream) généré en temps réel directement depuis la mémoire par l'API du daemon Docker. Dès que vous fermez la commande, les événements futurs ne sont écrits nulle part.
* Quant aux logs en **Rootless**, ils se trouvent dans le dossier personnel de l'utilisateur qui a lancé le daemon Docker (jettez un coup d'oeil pur voir les logs de vos conteneurs) :
```bash
/home/<nom_utilisateur>/.local/share/docker/containers/*/*.log
```
Contrairement à un Docker classique où les logs applicatifs sont dans `/var/lib/docker/containers/`.

Pour envoyer ces données à la suite ELK par exemple, if faut adapter les points de collect de logs et d'évelements. 

<details><summary>Example de filebeat</summary>

Pour que Filebeat détecte automatiquement les nouveaux conteneurs Rootless et lise leurs logs, vous devez modifier sa configuration par défaut pour pointer vers le bon **socket** et le bon **répertoire**.

Extrait du `filebeat.yml` :
```yaml
filebeat.autodiscover:
  providers:
    - type: docker
      # 1. On pointe vers le daemon Rootless de l'utilisateur 1000
      host: "unix:///run/user/1000/docker.sock"
      templates:
        - condition:
            contains:
              docker.container.image: nginx
          config:
            - type: container
              paths:
                # 2. On pointe vers le répertoire caché de l'utilisateur
                - /home/<nom_utilisateur>/.local/share/docker/containers/${data.docker.container.id}/*.log
```

</details>