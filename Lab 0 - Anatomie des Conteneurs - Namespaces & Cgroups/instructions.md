# Lab 0 : Anatomie des Conteneurs (Namespaces & Cgroups)

**Objectif :** Comprendre comment Linux isole les processus (Namespaces) et limite leurs ressources (Cgroups). Vous allez construire manuellement les briques qui composent des outils comme Docker.

**Prérequis :**
  * Une distribution Linux moderne avec **Cgroups v2** (Ubuntu 22.04+, Debian 11+).
  * Accès `root` ou `sudo`.
  * Paquets recommandés : `iproute2`, `python3`, `stress` (optionnel), `curl`.
  * **Attention :** Ce lab manipule des paramètres noyau. Il est fortement recommandé de l'utiliser dans une VM jetable.

-----

## PARTIE 1 : Les Namespaces (L'Isolation)

Les namespaces (espaces de noms) modifient la "vision" qu'un processus a du système. Nous allons explorer les namespaces disponibles sous Linux.

### Exercice 1 : UTS Namespace (Hostname)

**Concept :** Isoler le nom d'hôte et de domaine.

1.  Ouvrez un terminal. Vérifiez votre hostname actuel :

```bash
hostname
```

2.  Créez un nouveau processus shell (`bash`) dans un nouvel espace de noms UTS :

```bash
sudo unshare --uts
```

3.  Changez le hostname dans ce nouveau shell :

```bash
hostname container-test
hostname
# Doit afficher "container-test"
```

4.  Ouvrez un **second terminal** sur l'hôte et vérifiez le hostname :

```bash
hostname
# Doit afficher le nom d'origine (isolation réussie !)
```

5.  Quittez le namespace dans le premier terminal :

```bash
exit
```

### Exercice 2 : Network Namespace (Réseau)

**Concept :** Avoir sa propre pile réseau (interfaces, routes, iptables).

1.  Créez un shell isolé du réseau :

```bash
sudo unshare --net
```

2.  Vérifiez l'état du réseau (il doit être vide) :

```bash
ip link
# Seule l'interface 'lo' est visible et DOWN
ping 8.8.8.8
# Erreur : Network is unreachable
```

3.  Récupérez le PID de ce shell (depuis l'intérieur du shell) :

```bash
echo $$
# Notez ce numéro (ex: 1234)
```

4.  **Depuis un second terminal (sur l'hôte)**, nous allons créer un "câble virtuel" (veth pair) :

```bash
# Remplacez PID_DU_SHELL par le numéro noté ci-dessus
PID_NS=1234

# Création de la paire de câbles
sudo ip link add veth_host type veth peer name veth_container

# Brancher un bout sur l'hôte (assignation d'IP directe)
sudo ip addr add 10.0.0.1/24 dev veth_host
sudo ip link set veth_host up

# Envoyer l'autre bout dans le namespace du conteneur
sudo ip link set veth_container netns $PID_NS
```

5.  **Retournez dans le premier terminal (le namespace)** et configurez l'interface reçue :

```bash
# Renommer l'interface (cosmétique standard)
ip link set veth_container name eth0
ip link set lo up
ip link set eth0 up
ip addr add 10.0.0.2/24 dev eth0
ip route add default via 10.0.0.1
```

6.  Testez la connectivité :

```bash
ping 10.0.0.1
# Ça marche ! Vous communiquez avec l'hôte via votre réseau privé.
```

### Exercice 3 : Mount Namespace (Fichiers)

**Concept :** Isoler les points de montage pour que le conteneur ne modifie pas les dossiers de l'hôte.

1.  Créez un namespace Mount :

```bash
sudo unshare --mount
```

2.  Créez un montage privé en mémoire vive pour `/tmp` (pour cacher les fichiers temporaires de l'hôte) :

```bash
mount -t tmpfs none /tmp
```

3.  Créez un fichier dans ce `/tmp` :

```bash
touch /tmp/fichier_secret_container
ls /tmp
```

4.  Vérifiez depuis un autre terminal sur l'hôte :

```bash
ls /tmp
# Le fichier n'apparaît pas. L'isolation est active.
```

### Exercice 4 : PID Namespace (Processus)

**Concept :** Avoir son propre PID 1 (comme init ou systemd).

1.  Lancez `unshare`. Notez les options `--fork` (pour que le processus enfant devienne PID 1) et `--mount-proc` (crucial pour que `ps` lise les bons processus) :

```bash
sudo unshare --pid --fork --mount-proc
```

2.  Vérifiez les processus :

```bash
sleep 1000 &
ps aux
```

*Observation :* - Vous ne voyez que vos processus internes. Votre shell `bash` a le PID 1.
- Vous ne voyez plus les processus de l'hôte.
- Sur un autre terminal (l'hôte), la commande `ps aux | grep sleep` montre le processus `sleep`, mais avec un PID réel système (ex: 4567), illustrant que le PID est "traduit".

### Exercice 5 : IPC, User & Time Namespaces (Pour aller plus loin)

* **IPC Namespace (`--ipc`) :** Isole la mémoire partagée et les files de messages entre les processus.
* **User Namespace (`--user`) :** Permet d'être `root` (UID 0) à l'intérieur du conteneur, mais d'être un utilisateur standard non privilégié sur la machine hôte. (Essentiel pour la sécurité).
* **Time Namespace (`--time`) :** Permet au conteneur d'avoir une horloge différente de l'hôte.

-----

## PARTIE 2 : Les Control Groups (Les Quotas - Cgroups v2)

Les Cgroups limitent la consommation ("Combien tu peux manger"). Les distributions modernes utilisent la hiérarchie unifiée **Cgroups v2**, située dans `/sys/fs/cgroup/`.

### Exercice 1 : Memory Cgroup (Limiter la RAM)

**Objectif :** Tuer un processus qui consomme trop de mémoire (OOM Killer).

1.  Définissez une variable pour le dossier cgroup et créez-le :

```bash
CG=/sys/fs/cgroup/lab_memory
sudo mkdir $CG
```

2.  Fixez une limite stricte à 100 MB et bloquez l'usage du SWAP :

```bash
# 100 MB = 100 * 1024 * 1024 = 104857600 octets
echo 104857600 | sudo tee $CG/memory.max
# Interdire le swap pour forcer le kill immédiat
echo 0 | sudo tee $CG/memory.swap.max
```

3.  Ouvrez un shell Python et ajoutez-le au Cgroup (dans Cgroups v2, on utilise `cgroup.procs`) :

```bash
# Ajoutez le PID du shell actuel au groupe
echo $$ | sudo tee $CG/cgroup.procs

# Lancez python
python3
```

4.  **Le Crash Test :** Allouez de la mémoire progressivement.

```python
# Allouer 80MB (Ça passe)
a = "a" * 1024 * 1024 * 80

# Allouer 40MB de plus (Total 120MB > 100MB Limite)
b = "b" * 1024 * 1024 * 40
```

*Résultat :* Votre processus Python doit être brutalement tué par le noyau avec le message `Killed`.

5.  Nettoyage (depuis un autre terminal) :

```bash
sudo rmdir /sys/fs/cgroup/lab_memory
```

### Exercice 2 : CPU Cgroup (Limiter le Processeur)

**Objectif :** Restreindre le temps processeur alloué à un groupe.

1.  Créez un Cgroup CPU :

```bash
# Nous activons le contrôleur cpu depuis la racine des cgroups
echo "+cpu" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
```

```bash
# On crée un Cgroup CPU
CG_CPU=/sys/fs/cgroup/lab_cpu
sudo mkdir $CG_CPU
```

2.  Configurez le quota CPU (Cgroups v2 utilise `cpu.max` avec le format `QUOTA PÉRIODE`) :

```bash
# Limite à 100 000 microsecondes par période de 100 000 (Équivaut à 1 CPU à 100% maximum)
echo "100000 100000" | sudo tee $CG_CPU/cpu.max
```

3.  Lancez un processus `stress` dans ce groupe :

```bash
# Si besoin installer la commande stress
sudo apt install stress
```

```bash
# On lance stress sur 4 CPU virtuels, mais on le confine dans le cgroup
sudo sh -c "echo \$$ > $CG_CPU/cgroup.procs && exec stress --cpu 4"
```


4.  Dans un autre terminal, lancez `htop` ou `top`.

*Observation :* Même si `stress` lance 4 threads essayant de consommer 400% de CPU, le total cumulé de ces 4 processus ne dépassera jamais **100%** (soit l'équivalent d'un seul cœur complet).

### Exercice 3 : IO Cgroup (Disque - Optionnel)

**Objectif :** Limiter la vitesse de lecture (io.max en v2).

1.  Repérez votre disque principal (ex: 8:0 pour sda, utilisez `ls -l /dev/sda` pour voir Major:Minor).
2.  Créez le cgroup et limitez la lecture à 1MB/s :

```bash
# Nous activons le contrôleur IO depuis la racine des cgroups
echo "+io" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
```

```bash
CG_IO=/sys/fs/cgroup/lab_io
sudo mkdir $CG_IO
# Format Cgroups v2: "MAJOR:MINOR rbps=BYTES_PER_SECOND"
echo "8:0 rbps=1048576" | sudo tee $CG_IO/io.max
```

3.  Testez la vitesse :

```bash
sudo sh -c "echo \$$ > $CG_IO/cgroup.procs && dd if=/dev/sda of=/dev/null bs=1M count=10 iflag=direct"
```

*Résultat :* La vitesse affichée par `dd` devrait être bridée autour de `1.0 MB/s`. Pour observer cela, patientez que la commande précédente finisse son exécution pour voir le rapport.

-----

## PARTIE 3 : Le "Real Deal" (Namespaces + Cgroups + RootFS)

Pour finir, nous allons créer un conteneur complet. Nous n'allons pas seulement isoler les processus, nous allons lui donner **son propre système de fichiers** (une image Alpine Linux vierge) grâce à `chroot`.

### Exercice Final : Création d'un vrai conteneur "from scratch"

1. Préparation du système de fichiers (Image de base)

Sur l'hôte, téléchargez une image "mini-rootfs" d'Alpine Linux et extrayez-la dans un dossier :

```bash
mkdir -p ~/mon_vrai_conteneur/rootfs
cd ~/mon_vrai_conteneur/rootfs

# Téléchargement des dossiers vitaux d'Alpine (/bin, /etc, /lib...)
curl -O https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-minirootfs-3.19.1-x86_64.tar.gz
tar xf alpine-minirootfs-3.19.1-x86_64.tar.gz
rm alpine-minirootfs-3.19.1-x86_64.tar.gz
```

2. Préparation des Cgroups (Sur l'hôte)

```bash
CG_CONT=/sys/fs/cgroup/mon_vrai_conteneur
sudo mkdir -p $CG_CONT

# Limite RAM : 200MB max
echo 209715200 | sudo tee $CG_CONT/memory.max
# Limite CPU : Equivalent 1 coeur max
echo "100000 100000" | sudo tee $CG_CONT/cpu.max
```

3. Lancement du Conteneur avec `chroot`

Nous allons combiner `unshare` (pour l'isolation) avec `chroot` (pour changer le dossier racine / vers notre dossier Alpine). 

```bash
cd ~/mon_vrai_conteneur

# Création du conteneur. Notez l'appel à chroot à la fin !
sudo unshare --uts --net --mount --ipc --pid --fork chroot rootfs /bin/sh
```

Vous êtes maintenant à l'intérieur de votre conteneur ! Mais attendez, il manque une chose pour que des outils comme `ps` ou `top` fonctionnent, il faut monter un faux `/proc` propre à ce conteneur :

```bash
# À exécuter À L'INTÉRIEUR du conteneur
mount -t proc proc /proc
```

**Testez votre conteneur :**
* Tapez `ls /` : Vous voyez les fichiers d'Alpine, pas ceux de votre hôte. L'hôte est invisible !
* Tapez `cat /etc/os-release` : Vous êtes sous Alpine Linux.
* Tapez `ps aux` : Vous êtes bien le PID 1.

4. Application des Limites (Depuis l'Hôte)

Gardez le terminal du conteneur ouvert. Dans un **autre terminal** sur l'hôte, trouvez le PID du `sh` de votre conteneur et ajoutez-le au Cgroup :

```bash
# Trouver le PID de 'unshare' puis prendre son enfant direct
ps axf | grep unshare -A 1

PID_CIBLE=12345  # Remplacez par le vrai PID trouvé !
echo $PID_CIBLE | sudo tee $CG_CONT/cgroup.procs
```

Votre processus est maintenant enfermé dans son propre système de fichiers (Alpine), isolé du réseau, seul dans son arbre de processus, limité à 200MB de RAM et 1 CPU. **Félicitations, vous venez de recoder Docker !**

5. Nettoyage

Quittez le conteneur (`exit`) et nettoyez sur l'hôte :

```bash
sudo rmdir /sys/fs/cgroup/mon_vrai_conteneur
sudo rm -rf ~/mon_vrai_conteneur
```

## Conclusion du Lab

Vous venez de manipuler les fondamentaux :

1.  **Namespaces :** Vous avez créé des "boîtes" invisibles pour isoler le réseau, les fichiers, les PIDs, etc.
2.  **Cgroups :** Vous avez appliqué des menottes aux processus pour contrôler leur consommation de RAM et CPU.
3.  **Rootfs (Root Filesystem) :** Vous avez un arborescence de base d'un système, montée à la racine (`/`), qui contient l'ensemble des dossiers, bibliothèques et exécutables vitaux dont un système d'exploitation ou un conteneur a besoin pour démarrer et fonctionner.

**En conclusion, la magie des conteneurs n'existe pas : c'est exactement ce que fait le Docker Engine au quotidien en automatisant les appels aux Namespaces pour l'isolation, aux Cgroups pour les quotas, et au `chroot` pour le système de fichiers, créant ainsi des environnements d'exécution sûrs et isolés.**